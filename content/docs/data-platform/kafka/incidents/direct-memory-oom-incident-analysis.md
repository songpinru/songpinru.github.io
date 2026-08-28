---
title: Kafka Direct Memory OOM 排查报告
description: Kafka broker direct buffer memory exhaustion investigation and remediation
weight: 20
tags: ['kafka', 'operations', 'direct-memory', 'oom']
---


## 1. 文档信息

- 排查日期：2026-08-12
- 复查日期：2026-08-24
- Kafka 版本：3.9.2 定制分支
- Java 版本：OpenJDK 17.0.7
- GC：G1 GC
- Java Heap：`-Xms16G -Xmx16G`
- Heap Dump：约 555 MiB，包含约 326 万个存活对象
- 分析工具：JDK `jcmd`、JMX、Eclipse Memory Analyzer 1.17

本文中的主机、IP、认证信息等已省略。

## 2. 故障现象

Broker 网络 Processor 报错：

```text
java.lang.OutOfMemoryError:
Cannot reserve 77908729 bytes of direct buffer memory
```

关键调用栈：

```text
java.nio.Bits.reserveMemory
java.nio.DirectByteBuffer.<init>
java.nio.ByteBuffer.allocateDirect
sun.nio.ch.Util.getTemporaryDirectBuffer
sun.nio.ch.IOUtil.read
sun.nio.ch.SocketChannelImpl.read
org.apache.kafka.common.network.PlaintextTransportLayer.read
org.apache.kafka.common.network.NetworkReceive.readFrom
org.apache.kafka.common.network.KafkaChannel.read
kafka.network.Processor.poll
```

本次申请大小为：

```text
77,908,729 bytes = 74.30 MiB
```

## 3. 结论摘要

本次故障不是普通 Java Heap OOM，也不是少量未释放业务对象导致的传统内存泄漏。

直接原因是：

1. JVM Direct Buffer 上限约为 16 GiB。
2. JDK NIO 为 heap `ByteBuffer` 的 Socket/File IO 创建临时 direct buffer。
3. `jdk.nio.maxCachedBufferSize` 未配置，JDK 17 默认允许缓存任意大小的临时 direct buffer。
4. 大型 direct buffer 被长期存活线程的 `ThreadLocal<Util.BufferCache>` 强引用。
5. 592 个真实 direct buffer 共占用约 15.96 GiB，其中 99.93% 位于线程本地缓存。
6. 剩余 direct memory 小于 74.30 MiB 时，新请求触发 OOM。

最大来源是 Tiered Storage 的 200 个 `remote-log-reader` 线程，每个线程缓存约 55 MiB，合计约 10.74 GiB。

## 4. 关键配置

与本次问题直接相关的配置：

```properties
num.network.threads=40
num.io.threads=40
socket.request.max.bytes=104857600
queued.max.requests=200
replica.fetch.max.bytes=104857600
num.replica.fetchers=16
```

监听器包含两个 data-plane listener，因此实际网络 Processor 数量为：

```text
SASL_PLAINTEXT：40
BROKER：        40
总计：          80
```

Heap Dump 显示存在：

```text
remote-log-reader0 ... remote-log-reader199
```

因此可以确认运行时 Remote Log Reader 线程池大小为 200，即有效配置相当于：

```properties
remote.log.reader.threads=200
```

Kafka 代码中的默认值为 10，线上值需要继续确认来自静态配置、配置模板还是部署平台注入。

## 5. Direct Memory 上限确认

执行：

```bash
jcmd $PID VM.flags | grep -E 'MaxHeapSize|MaxDirectMemorySize'
jcmd $PID VM.command_line
jcmd $PID VM.system_properties | grep jdk.nio.maxCachedBufferSize
```

结果：

- `-Xms16G -Xmx16G`
- 未配置 `-XX:MaxDirectMemorySize`
- 未配置 `jdk.nio.maxCachedBufferSize`

JDK 17 未显式配置 `MaxDirectMemorySize` 时，Direct Buffer 上限默认采用 JVM 最大堆大小，因此本进程上限约为 16 GiB。

`jdk.nio.maxCachedBufferSize` 未配置时，JDK NIO 临时 direct buffer 缓存上限为 `Long.MAX_VALUE`。

## 6. Kafka 网络读取代码路径

### 6.1 Kafka 分配 heap buffer

`NetworkReceive` 读取请求长度并申请缓冲区：

```java
int receiveSize = size.getInt();
buffer = memoryPool.tryAllocate(requestedBufferSize);
```

默认 `MemoryPool.NONE` 和 `SimpleMemoryPool` 都调用：

```java
ByteBuffer.allocate(sizeBytes);
```

因此 Kafka 请求缓冲区本身是 Java Heap buffer。

相关代码：

- `clients/src/main/java/org/apache/kafka/common/network/NetworkReceive.java`
- `clients/src/main/java/org/apache/kafka/common/memory/MemoryPool.java`
- `clients/src/main/java/org/apache/kafka/common/memory/SimpleMemoryPool.java`

### 6.2 JDK 申请临时 direct buffer

Kafka 将 heap buffer 传入：

```java
socketChannel.read(dst);
```

JDK `IOUtil.read` 发现目标不是 direct buffer 后，会根据 `dst.remaining()` 申请同等大小的临时 direct buffer：

```java
ByteBuffer temporary = Util.getTemporaryDirectBuffer(remaining);
```

读取完成后，该 buffer 被放回当前线程的 `Util.BufferCache`，而不是立即释放。

本次 74.30 MiB 的 OOM 申请来自这条路径。

### 6.3 网络 / IO线程缓冲区大小的影响参数

Kafka中的“网络线程”对应 Processor，负责 socket 收发；“IO线程”对应 `data-plane-kafka-request-handler-*`，负责请求处理和日志写入。它们的大型 direct buffer 都来自 JDK 对 heap `ByteBuffer` IO 的临时转换，实际大小取决于应用层 buffer 的 `remaining()`，而不是内核 socket buffer。

| 参数 | 影响对象 | 对缓冲区的影响 |
|---|---|---|
| `num.network.threads` | 网络 Processor | 决定每个 listener 的 Processor 数量，不直接决定单个 buffer 大小，但会放大总缓存：`Processor数 * 历史最大单次IO` |
| `num.io.threads` | Request Handler | 决定 request handler 数量，不直接决定单个 buffer 大小，但同样放大总缓存 |
| `socket.request.max.bytes` | 网络 Processor | Broker允许的最大请求字节数，是请求 heap buffer及网络读取临时 direct buffer的硬上限 |
| `message.max.bytes` / topic `max.message.bytes` | 网络 Processor、Request Handler | 限制单个 record batch最大字节数，影响 Produce请求大小 |
| producer `max.request.size` / `batch.size` | 网络 Processor、Request Handler | 决定客户端实际能构造多大的请求和批次 |
| `fetch.max.bytes` | Fetch / Remote Log Reader | Broker单次 Fetch响应总大小；Remote Log读取会按该值附近分配 heap buffer，并触发临时 direct buffer |
| consumer `max.partition.fetch.bytes` | Fetch 路径 | 单分区 Fetch的目标大小，影响实际读取窗口 |
| `replica.fetch.max.bytes` | ReplicaFetcher | 副本拉取请求中单分区可返回的最大字节数 |
| `replica.fetch.response.max.bytes` | ReplicaFetcher | 副本 Fetch响应总大小上限，需要注意与单分区上限的叠加关系 |
| `num.replica.fetchers` | ReplicaFetcher | 每个源 Broker的 fetcher数量，放大副本拉取线程数量和总缓存风险 |
| `remote.log.reader.threads` | Remote Log Reader | 远程日志读取线程数，直接放大 `fetch.max.bytes` 对应的缓存总量 |
| `queued.max.requests` | 请求队列 | 只限制排队请求个数，不限制单个请求字节数 |
| `queued.max.request.bytes` | 请求队列 | 限制排队请求总字节数，可提供字节级反压，但不缩小单次 IO buffer |
| `socket.receive.buffer.bytes` | OS socket | 内核接收缓冲区大小，不是 Kafka应用层 `ByteBuffer`，不限制大请求读取 |
| `socket.send.buffer.bytes` | OS socket | 内核发送缓冲区大小，不是 Kafka应用层 `ByteBuffer`，不限制单次响应写入 |
| `jdk.nio.maxCachedBufferSize` | 所有 NIO线程 | 限制允许进入 ThreadLocal缓存的最大临时 direct buffer；不限制瞬时 direct allocation |
| `-XX:MaxDirectMemorySize` | JVM | 限制 direct memory总量，只决定何时 OOM，不减少缓存需求 |

当前配置中的关键值：

```properties
num.network.threads=40
num.io.threads=40
socket.send.buffer.bytes=1048576
socket.receive.buffer.bytes=1048576
socket.request.max.bytes=104857600
queued.max.requests=200
num.replica.fetchers=16
replica.fetch.max.bytes=104857600
```

其中：

```text
socket.request.max.bytes = 104857600 bytes = 100 MiB
```

因此，`socket.receive.buffer.bytes=1 MiB` 不能阻止 79 MiB 请求进入 Kafka应用层 `ByteBuffer`。请求大小由 `socket.request.max.bytes`、消息批次配置和客户端请求大小共同决定。

风险模型可以写成：

```text
常驻 direct buffer 高水位
≈ 长期存活 NIO线程数
  * 每个线程历史处理过的最大单次 heap ByteBuffer IO 剩余空间
```

### 6.4 与零拷贝路径的关系

Kafka的零拷贝主要覆盖本地日志消费路径：PLAINTEXT场景下，Broker可以把本地 log segment通过 `FileChannel.transferTo()` 直接发送到 socket，路径上不需要把消息读入 JVM heap buffer，也不会为这段数据创建同等大小的 JDK临时 direct buffer。

但 Produce和远程日志读取路径不在该零拷贝范围内：

1. Produce必须先从 socket读入 JVM heap请求 buffer，再写入日志文件。
2. heap buffer参与 `SocketChannel.read` 或 `FileChannel.write` 时，JDK会创建临时 direct buffer。
3. SSL等加密传输层不能直接使用最简单的 `transferTo` 零拷贝路径；SASL场景还取决于具体 TransportLayer实现。
4. Tiered Storage远程读取需要先把远程数据读入 heap buffer，再进入正常 Fetch返回路径。

因此，零拷贝能减少本地日志 Fetch的用户态拷贝，但无法消除 Produce、SSL、SASL、远程日志读取这些路径上的 heap-to-direct转换。

## 7. 现场数据采集

### 7.1 JMX Direct BufferPool

Prometheus/JMX 指标：

```text
java_nio_direct_totalcapacity  17,134,924,451
java_nio_direct_memoryused     17,134,924,452
java_nio_direct_count          595
```

换算结果：

```text
Direct MemoryUsed：约 15.96 GiB
Direct Count：     595
Direct剩余空间：   约 42.86 MiB
本次申请：         74.30 MiB
```

因此：

```text
42.86 MiB < 74.30 MiB
```

OOM 与 Direct Buffer 上限完全吻合。

建议持续监控：

```text
java.nio:type=BufferPool,name=direct
  MemoryUsed
  TotalCapacity
  Count

java.nio:type=BufferPool,name=mapped
```

### 7.2 Class Histogram

执行：

```bash
jcmd $PID GC.class_histogram -all |
  grep -E 'DirectByteBuffer|MappedByteBuffer|Util\$BufferCache|Deallocator'
```

关键结果：

```text
java.nio.DirectByteBuffer                 5440
java.nio.DirectByteBufferR                1672
java.nio.DirectByteBuffer$Deallocator      590
sun.nio.ch.Util$BufferCache                428
```

解释：

- `DirectByteBuffer` 和 `DirectByteBufferR` 中包含大量 slice、duplicate 和只读视图。
- 这些视图可能共享同一块 native memory，不能按对象数累加容量。
- `DirectByteBuffer$Deallocator` 数量约 590，与 JMX Direct Buffer Count 基本一致。
- 真实 native direct allocation 大约为 590 个。

### 7.3 线程统计

线程 Dump 确认：

```text
Kafka网络Processor：80
ReplicaFetcherThread：76
```

`num.replica.fetchers=16` 表示每个源 Broker 最多 16 个 fetcher，而不是整个 Broker 总共 16 个。

Fetcher 由以下组合唯一确定：

```text
(sourceBrokerId, fetcherId)
```

## 8. Heap Dump 与 MAT 分析

### 8.1 生成 Heap Dump

在低峰期执行：

```bash
jcmd $PID GC.heap_dump /independent-disk/kafka-direct-$PID.hprof
```

注意：

- Heap Dump 可能触发较长安全点停顿。
- 不应写入繁忙的 Kafka 数据盘。
- Dump 可能包含消息内容、认证信息等敏感数据。
- HPROF 不保存 16 GiB native数据本身，但会保存 DirectByteBuffer wrapper、容量和 GC Root 引用。

### 8.2 MAT 解析结果

MAT 识别：

```text
Direct root buffers：                  592
Direct root capacity：                 17,137,743,183 bytes
                                        15.96 GiB
Util.BufferCache：                     430
非空 BufferCache：                    426
被 BufferCache 持有的 direct buffer： 589
缓存 direct capacity：                17,126,208,772 bytes
缓存占比：                            99.93%
未被 BufferCache 持有：               3
```

结论：

```text
99.93%的 Direct Memory 被存活线程的 ThreadLocal NIO BufferCache 持有。
```

Full GC 无法释放这些 buffer，因为它们仍然存在强引用。

## 9. Direct Memory 占用明细

| 来源 | 线程数 | Buffer数 | 容量 | 占比 |
|---|---:|---:|---:|---:|
| `remote-log-reader*` | 200 | 200 | 10.74 GiB | 67.3% |
| SASL客户端网络 Processor | 40 | 约 230 | 3.05 GiB | 19.1% |
| Kafka Request Handler | 40 | 40 | 1.77 GiB | 11.1% |
| ReplicaFetcher | 76 | 75 | 393 MiB | 2.4% |
| 其他线程 | 少量 | 少量 | 约 12 MiB | 小于 0.1% |

### 9.1 Remote Log Reader

200 个 `remote-log-reader` 线程全部持有约 55 MiB 的临时 direct buffer：

```text
线程数：200
最小容量：55.00 MiB
P50：     55.00 MiB
P95：     55.00 MiB
最大容量：55.00 MiB
平均容量：55.00 MiB
合计：   10.74 GiB
```

Remote Log Reader 线程池创建代码：

```java
remoteStorageReaderThreadPool = new RemoteStorageThreadPool(
    "remote-log-reader",
    rlmConfig.remoteLogReaderThreads(),
    rlmConfig.remoteLogReaderMaxPendingTasks()
);
```

位置：

```text
core/src/main/java/kafka/log/remote/RemoteLogManager.java
storage/src/main/java/org/apache/kafka/storage/internals/log/RemoteStorageThreadPool.java
```

Remote Log读取代码：

```java
int maxBytes = Math.min(fetchMaxBytes, fetchInfo.maxBytes);
int updatedFetchSize = ...;
ByteBuffer buffer = ByteBuffer.allocate(updatedFetchSize);
Utils.readFully(remoteSegInputStream, buffer);
```

#### 9.1.1 为什么读取大小是 55 MiB

Broker默认：

```java
fetch.max.bytes = 55 * 1024 * 1024;
```

即：

```text
55 MiB = 57,671,680 bytes
```

Remote Log读取时先计算：

```java
int maxBytes = Math.min(fetchMaxBytes, fetchInfo.maxBytes);
```

当Broker全局Fetch上限和分区Fetch上限都允许55 MiB时，`maxBytes` 和 `updatedFetchSize` 通常为55 MiB。

随后Kafka申请一个Java Heap buffer：

```java
ByteBuffer buffer = ByteBuffer.allocate(updatedFetchSize);
```

这里的 `ByteBuffer.allocate` 不是direct allocation，55 MiB首先计入Java Heap。

#### 9.1.2 为什么MAT中的容量略小于55 MiB

Kafka找到第一个RecordBatch后，会先把它写入目标heap buffer：

```java
firstBatch.writeTo(buffer);
```

然后才读取远程流的剩余数据：

```java
Utils.readFully(remoteSegInputStream, buffer);
```

MAT中200个Remote Reader持有的direct buffer容量范围为：

```text
57,669,856 ～ 57,671,068 bytes
```

与完整55 MiB的差值为：

```text
612 ～ 1,824 bytes
```

该差值与先写入的第一个RecordBatch大小一致。因此，JDK临时direct buffer的容量实际对应：

```text
updatedFetchSize - firstBatchSize
```

这也是200个缓存都非常接近55 MiB，但并不完全相等的原因。

#### 9.1.3 heap buffer为什么又产生direct buffer

`Utils.readFully` 取出heap ByteBuffer背后的数组，并把全部剩余长度传给一次 `InputStream.read`：

```java
int length = destinationBuffer.remaining();

inputStream.read(
    array,
    initialOffset + totalBytesRead,
    length - totalBytesRead
);
```

第一次调用时，`length - totalBytesRead` 接近55 MiB，而不是按64 KiB或1 MiB分块。

现场Tiered Storage插件使用Hadoop/HDFS 3.3.6。远程存储/HDFS底层网络IO需要把数据读入heap目标区域；当底层JDK NIO Channel面对heap目标buffer时，JDK `IOUtil.read` 会创建一个同等 `remaining()` 容量的临时direct buffer：

```java
int rem = dst.remaining();
ByteBuffer temporary = Util.getTemporaryDirectBuffer(rem);
```

数据路径可以表示为：

```text
HDFS/DataNode或远程存储IO
            |
            v
JDK temporary DirectByteBuffer，接近55 MiB
            |
            | copy
            v
Kafka heap byte[] / ByteBuffer，55 MiB
```

因此一次Remote Log读取在执行期间可能同时存在：

```text
约55 MiB Java Heap buffer
+ 约55 MiB temporary DirectByteBuffer
```

HPROF不保存native allocation的原始调用栈，因此不能仅靠Heap Dump还原远程存储插件内部的每一层调用；但Buffer容量、所属线程和Kafka传入的剩余读取长度精确对应，可以确认该direct buffer来自这一大块heap IO的JDK临时缓冲机制。

#### 9.1.4 为什么读取结束后仍不释放

JDK NIO在IO结束后不会立即free临时direct buffer，而是执行：

```java
Util.offerFirstTemporaryDirectBuffer(temporary);
```

buffer随后进入当前线程自己的ThreadLocal缓存：

```text
remote-log-reader-N
 -> ThreadLocalMap
 -> sun.nio.ch.Util$BufferCache
 -> DirectByteBuffer，capacity接近55 MiB
```

未配置 `jdk.nio.maxCachedBufferSize` 时，JDK 17默认缓存大小上限为 `Long.MAX_VALUE`，所以55 MiB buffer也允许进入缓存。

缓存的是一块可复用的native IO工作区，不是Kafka主动维护的消息缓存。旧内容可能暂时留在内存中，但下次IO会覆盖它。复用时position和limit会重置，capacity仍保持约55 MiB。

后续即使该线程只读取1 MiB，也可以复用这块55 MiB buffer；JDK不会因为请求变小而自动缩容。

#### 9.1.5 为什么每个Reader线程都有一块

Remote Log Reader使用固定线程池：

```java
super(
    numThreads,
    numThreads,
    0L,
    TimeUnit.MILLISECONDS,
    ...
);
```

即：

```text
corePoolSize = maximumPoolSize = remote.log.reader.threads
```

代码没有开启核心线程超时，因此核心线程会长期存活。线程不退出时，其ThreadLocal、`Util.BufferCache` 和DirectByteBuffer也不会销毁。

线上有200个线程：

```text
remote-log-reader0
...
remote-log-reader199
```

随着Remote Log请求持续被线程池分发，越来越多线程至少执行过一次接近55 MiB的读取。每个线程达到一次大IO水位后，就会保留自己的direct buffer：

```text
remote-log-reader0   -> 约55 MiB
remote-log-reader1   -> 约55 MiB
...
remote-log-reader199 -> 约55 MiB
```

由于：

```text
remote.log.reader.threads = 200
fetch.max.bytes           = 55 MiB
```

所以仅此一项的稳定缓存上限接近：

```text
200 * 55 MiB = 10.74 GiB
```

这是一种线程级的历史最大水位行为，而不是每次请求结束后归零。JMX通常表现为阶梯式增长：每触达一个尚未处理过大请求的Reader线程，Direct Memory就增加约55 MiB，最终达到稳定高位。

#### 9.1.6 配置变化对缓存的影响

降低 `fetch.max.bytes` 只能降低后续IO请求尺寸，不能让当前线程已经缓存的55 MiB buffer自动缩小。较小请求仍会复用较大的缓存。

配置：

```bash
-Djdk.nio.maxCachedBufferSize=16777216
```

可以阻止超过16 MiB的临时buffer长期进入缓存，但不能阻止一次55 MiB读取发生瞬时direct allocation。若仍保留200个并发Reader，还可能出现大量申请和释放以及较高峰值。

要释放现有缓存，需要让持有它的线程退出，通常需要修改配置后滚动重启Broker。

从机制上解决该问题，需要限制传给底层 `InputStream.read` 或NIO Channel的单次读取长度。例如按1 MiB分块后，单个Reader线程的JDK临时direct buffer高水位可以从约55 MiB降低到约1 MiB。

### 9.2 SASL_PLAINTEXT 网络 Processor

两个 data-plane listener各有40个 Processor，但大 buffer 几乎全部位于外部客户端 listener：

```text
SASL_PLAINTEXT：3.05 GiB
BROKER：        约 20 KiB
```

说明网络侧 direct memory 增长主要来自客户端大请求，不是 Broker复制流量。

单个 SASL Processor 最大缓存：

```text
168.68 MiB，包含3个direct buffer
```

多个 Processor 缓存超过 100 MiB。

异常中的精确 buffer：

```text
capacity = 77,908,729 bytes
owner    = SASL_PLAINTEXT network Processor
```

这直接证明 OOM 请求已经存在于网络线程缓存中。

#### 9.2.1 单个请求大小的决定因素

这类约 79 MiB的 buffer不是由 `socket.receive.buffer.bytes` 或 `socket.send.buffer.bytes` 决定的。这两个参数是内核 socket收发缓冲区大小，线上为 1 MiB，但它们不会限制 Kafka应用层 `ByteBuffer` 的大小。

直接上限来自：

```properties
socket.request.max.bytes=104857600
```

即 Broker允许单个请求最大 100 MiB。Kafka收到请求头后会按请求大小分配 heap `ByteBuffer`；JDK在执行 `SocketChannel.read(heapBuffer)` 时，会按 `heapBuffer.remaining()` 创建同等大小的临时 direct buffer。

大 Produce请求的路径可以简化为：

```text
客户端 Produce请求
        |
        v
网络 Processor：SocketChannel.read(heap request buffer)
        |
        v
JDK temporary DirectByteBuffer
        |
        v
Request Handler：FileChannel.write(heap records buffer)
```

实际请求大小还可能受以下配置影响：

```properties
broker: message.max.bytes
topic:  max.message.bytes
client: max.request.size
client: batch.size
```

一个请求内多个分区或多个 record batch叠加时，单个请求可以明显大于单条消息上限。Heap Dump只能证明 buffer容量、所属线程和引用关系，不能直接还原请求内容；定位来源还需要结合请求日志、审计日志、客户端指标和 topic / producer配置。

当前 Broker全局配置中未显式看到 `message.max.bytes`，因此需要继续确认是否来自 topic级配置或客户端侧请求配置。

```bash
bin/kafka-configs.sh --bootstrap-server <broker> \
  --entity-type topics \
  --entity-name <topic> \
  --describe | grep max.message.bytes
```

### 9.3 Kafka Request Handler

40个 Request Handler 合计缓存约 1.77 GiB。

多个 Request Handler 的 direct buffer 容量与网络请求 buffer 高度匹配，例如：

```text
网络读取 Buffer：约 79,159,409 bytes
请求处理 Buffer：约 79,159,345 bytes
```

两者仅相差请求头等少量字节，符合大 Produce 请求在网络读取后，写入 FileChannel 时再次触发 JDK临时 direct buffer 的行为。

因此一个大 Produce 请求可能同时形成：

```text
heap request buffer
+ 网络读取 temporary direct buffer
+ 磁盘写入 temporary direct buffer
```

#### 9.3.1 缓存增长与释放机制

Request Handler中的大型 direct buffer不是业务缓存，而是 JDK在 heap `ByteBuffer` 执行文件 IO时创建的临时 direct buffer。Kafka写日志时，Request Handler会把 Produce请求中的 heap buffer写入 `FileChannel`。JDK发现写入目标是 heap buffer后，会按 `remaining()` 创建同等大小的临时 direct buffer。

因此，一个 Request Handler缓存 79 MiB，表示该线程历史上至少处理过一次约 79 MiB的日志追加 IO。缓存大小是线程历史高水位，不会随着后续小请求自动缩小。

这类 buffer不会因为 Full GC自动释放。释放条件通常是：

```text
持有 BufferCache的线程退出
或 JDK清理对应 ThreadLocal
或重启进程
```

Kafka request handler是长期存活线程，正常运行期间不会退出，所以现有缓存需要滚动重启才能清空。配置 `jdk.nio.maxCachedBufferSize` 后也只影响新缓存行为，不能缩小已经进入缓存的旧 buffer。

### 9.4 2026-08-24 堆外内存复查

复查使用的 Heap Dump：

```text
/Users/pinru/Downloads/kafka-direct-17631.hprof
```

本次 Dump 中共有 7,227 个 `DirectByteBuffer` / `DirectByteBufferR` wrapper。按底层堆外地址去重后，总容量约 19.75 GiB。需要注意的是，这个总数同时包含两类堆外内存：

| 类型 | 容量 | 说明 |
|---|---:|---|
| JDK NIO 临时 direct buffer 缓存 | 12.81 GiB | 由 `ThreadLocal` 的 `sun.nio.ch.Util$BufferCache` 持有 |
| Log index mmap | 6.88 GiB | `OffsetIndex` 约 3.93 GiB，`TimeIndex` 约 2.95 GiB |
| JDK jimage mmap | 约 0.13 GiB | JVM 自身模块镜像映射 |

因此，本次真正需要优先治理的是 12.81 GiB 的 NIO 临时 direct buffer 缓存；6.88 GiB log index mmap 是文件映射内存，随索引文件生命周期存在，不属于 JDK 临时 direct buffer 缓存，也不应由 `jdk.nio.maxCachedBufferSize` 治理。

与上次事故相比：

| 指标 | 2026-08-12 OOM | 2026-08-24 复查 | 变化 |
|---|---:|---:|---|
| `remote-log-reader` 线程数 | 200 | 100 | 减少 100 |
| `remote-log-reader` 缓存 | 10.74 GiB | 5.77 GiB | 减少 4.97 GiB |
| SASL 网络 Processor 缓存 | 3.05 GiB | 3.90 GiB | 增加 0.85 GiB |
| Request Handler 缓存 | 1.77 GiB | 2.66 GiB | 增加 0.89 GiB |
| ReplicaFetcher 缓存 | 393 MiB | 469.5 MiB | 增加 76.5 MiB |

本次线程组明细：

| 线程组 | 线程数 | 缓存容量 | 线程平均缓存 | 单个buffer最大值 |
|---|---:|---:|---:|---:|
| `remote-log-reader*` | 100 | 5.77 GiB | 57.7 MiB | 57.67 MiB |
| `SASL_PLAINTEXT-* network processor` | 40 | 3.90 GiB | 97.4 MiB | 79.13 MiB |
| `data-plane-kafka-request-handler-*` | 40 | 2.66 GiB | 66.6 MiB | 79.13 MiB |
| `ReplicaFetcherThread-*` | 84 | 469.5 MiB | 5.6 MiB | 79.2 MiB |
| 其他线程 | 少量 | 约 11.3 MiB | - | - |

SASL 网络 Processor 的单个 buffer 最大值为 79.13 MiB；由于一个线程的 `BufferCache` 可以保留多个不同大小的 buffer，其线程平均缓存约 97.4 MiB。Request Handler 和 Remote Log Reader 本次均为每线程 1 个大 buffer。

#### 9.4.1 Remote Log Reader

100 个 `remote-log-reader` 线程各持有 1 个缓存 buffer，容量均为：

```text
57,671,680 bytes = 57.67 MiB
```

该值与 Kafka Broker 默认 `fetch.max.bytes` 的 55 MiB 完全对应：

```text
55 MiB = 55 * 1024 * 1024 = 57,671,680 bytes
```

与上次 200 个线程相比，Reader 线程数已减半，对应缓存也从 10.74 GiB 降至 5.77 GiB。这说明线程数调整方向有效，但 100 个长期线程仍然会形成：

```text
100 * 57.67 MiB ≈ 5.64 GiB
```

的稳定缓存水位。

#### 9.4.2 Request Handler

本次 40 个 `data-plane-kafka-request-handler-*` 线程各持有 1 个缓存 buffer：

```text
总容量：2.66 GiB
最小值：51.2 MiB
最大值：79.13 MiB
```

容量分布大致为：

| 单线程缓存大小 | 线程数 |
|---|---:|
| 约 50 MiB | 2 |
| 约 55 MiB | 2 |
| 约 60 MiB | 9 |
| 约 65 MiB | 9 |
| 约 70 MiB | 8 |
| 约 75 MiB | 7 |
| 约 79-80 MiB | 3 |

与上次相比，Request Handler缓存从 1.77 GiB增加到 2.66 GiB。

单个 Request Handler 的缓存范围、增长机制和释放条件见 `9.3.1`。

### 9.5 复查结论

Remote Log Reader 线程数从 200 降至 100 后，其 direct buffer 缓存从 10.74 GiB 降至 5.77 GiB，说明该项整改有效。但本次网络 Processor 和 Request Handler 的缓存继续增长到 3.90 GiB 和 2.66 GiB，单线程最大 buffer 达到 79.13 MiB，说明大请求 / 大批次路径已经成为新的主要增长点。

本次 Heap Dump 中等待 Cleaner 释放的垃圾 direct buffer 仅约 62.6 MiB，说明主要占用不是未回收对象，而是存活线程持有的 ThreadLocal 缓存。仅触发 Full GC 不能解决问题。

## 10. 根因

### 10.1 第一根因：Remote Log Reader线程数过高

`remote.log.reader.threads` 的有效值为200，是默认值10的20倍。

每个 Reader线程缓存约55 MiB，单项占用10.74 GiB，是最主要来源。

### 10.2 第二根因：JDK大型临时 Buffer缓存不受限

未配置：

```text
-Djdk.nio.maxCachedBufferSize
```

JDK 17 会长期缓存线程处理过的最大临时 direct buffer。线程不退出，buffer通常不会释放。

### 10.3 第三根因：允许约100 MiB的客户端请求

```properties
socket.request.max.bytes=104857600
```

外部 SASL listener已处理大量50至80 MiB请求，导致网络 Processor和Request Handler分别缓存大型 direct buffer。

### 10.4 放大因素：线程数量

```properties
num.network.threads=40
num.io.threads=40
num.replica.fetchers=16
remote.log.reader.threads=200
```

Direct Memory风险近似为：

```text
长期存活IO线程数 * 每个线程处理过的最大IO尺寸
```

### 10.5 复查更新

2026-08-24 复查确认，Remote Log Reader 已从 200 个线程降至 100 个，对应缓存降至 5.77 GiB；但外部客户端网络 Processor 和 Request Handler 的大请求缓存继续增长，最大单次 IO 达到 79.13 MiB。因此当前风险已经从单一 Remote Log Reader 放大，转为“Remote Log Reader 大 Fetch 缓存 + 客户端大 Produce 请求缓存”叠加。

## 11. 整改建议

### 11.1 P0：降低 Remote Log Reader线程数

首先确认200的配置来源：

```bash
grep '^remote.log.reader.threads' server.properties

kafka-configs.sh --bootstrap-server <broker> \
  --entity-type brokers \
  --entity-name <broker-id> \
  --describe --all
```

将线程数从200显著降低。Kafka默认值为10，生产目标值应根据以下指标压测确定：

```text
RemoteLogReaderAvgIdlePercent
RemoteLogReaderTaskQueueSize
RemoteLogReaderFetchRateAndTimeMs
远程读取P95/P99延迟
远程存储带宽和限流
```

不应在没有流量评估的情况下直接保留200个线程。

### 11.2 P0：配置后滚动重启

当前缓存被 ThreadLocal 强引用：

```text
Thread
 -> ThreadLocalMap
 -> Util.BufferCache
 -> DirectByteBuffer
```

执行 Full GC 或 `jcmd GC.run` 无法释放。

完成配置修改后需要滚动重启 Broker，使旧线程退出并释放缓存。

### 11.3 P1：限制JDK临时 Buffer缓存

可以评估加入：

```bash
-Djdk.nio.maxCachedBufferSize=16777216
```

示例值为16 MiB，不是无条件推荐值。

注意：超过阈值的临时buffer不再长期缓存，但大IO仍会发生瞬时direct分配。阈值过小可能导致大请求反复申请和释放direct memory，需要配合线程数调整并进行性能压测。

### 11.4 P1：评估降低 `fetch.max.bytes`

当前默认值：

```properties
fetch.max.bytes=57671680
```

该值直接影响Remote Log读取的heap buffer及底层临时direct buffer大小。

降低该值可能影响消费者单次Fetch吞吐，需要结合消息批次大小和客户端Fetch配置评估。

### 11.5 P1：限制排队请求总字节数

当前：

```properties
queued.max.requests=200
queued.max.request.bytes=-1
```

仅限制请求数量，未限制总字节数。建议设置：

```properties
queued.max.request.bytes=<容量评估值>
```

该值必须不小于 `socket.request.max.bytes`，用于控制 heap 请求缓冲区并提供网络反压。

### 11.6 P1：重新评估网络与IO线程数

根据以下指标评估：

```text
NetworkProcessorAvgIdlePercent
RequestHandlerAvgIdlePercent
请求P95/P99延迟
磁盘吞吐和IO等待
```

当前每个data-plane listener有40个网络线程，总计80个。线程数越多，可形成的独立JDK BufferCache越多。

### 11.7 P2：代码层限制单次IO窗口

长期方案可以考虑限制单次heap buffer IO窗口，而不是将整个剩余容量交给JDK NIO。

候选位置：

```text
clients/src/main/java/org/apache/kafka/common/network/PlaintextTransportLayer.java
clients/src/main/java/org/apache/kafka/common/utils/Utils.java
```

例如Remote Log读取时，将单次 `InputStream.read` 的长度限制在1至4 MiB：

```java
int chunkSize = Math.min(destinationBuffer.remaining(), MAX_IO_CHUNK_SIZE);
inputStream.read(array, offset, chunkSize);
```

网络读取可以临时缩小 `ByteBuffer.limit()`，限制单次 `SocketChannel.read` 看到的 `remaining()`。

代码修改需要评估：

- 系统调用次数
- 大请求吞吐
- CPU开销
- Remote Storage SDK行为
- Java 17和后续JDK版本兼容性

### 11.8 复查后的整改优先级

1. 定位产生 50-79 MiB 请求的 topic 和 producer，确认 broker / topic / client 侧 `max.message.bytes`、`max.request.size`、`batch.size` 等配置。
2. 评估将 `message.max.bytes`、topic `max.message.bytes` 和 producer 批次大小控制在业务允许范围内，避免单个请求接近 `socket.request.max.bytes` 的 100 MiB 上限。
3. 继续评估 `remote.log.reader.threads` 是否需要从 100 进一步下调，同时观察远程读取队列和延迟。
4. 评估 `jdk.nio.maxCachedBufferSize`，优先考虑 1 MiB、4 MiB、16 MiB 等候选值做性能对比，而不是直接照抄固定值。
5. 配置生效后滚动重启 Broker，清空现有网络线程、Request Handler 和 Remote Log Reader 的 ThreadLocal 缓存。

## 12. 验证方案

整改后持续观察：

```text
java_nio_direct_memoryused
java_nio_direct_totalcapacity
java_nio_direct_count
jvm_buffer_pool_used_bytes{pool="direct"}
jvm_buffer_pool_used_buffers{pool="direct"}
```

建议验收条件：

1. Broker重启后 Direct Memory显著下降。
2. Remote Log流量持续运行后，Direct Memory不再接近16 GiB。
3. Direct Memory不存在随线程逐步触达大请求而单调上涨至上限的趋势。
4. `RemoteLogReaderTaskQueueSize` 未持续堆积。
5. Remote Fetch P95/P99延迟满足SLA。
6. 网络 Processor和Request Handler空闲率处于合理范围。
7. 不再出现 `Cannot reserve ... bytes of direct buffer memory`。

建议告警：

```text
direct_memory_used / direct_memory_capacity > 70%
```

并对80%和90%设置更高等级告警。

## 13. 常用排查命令

### JVM参数

```bash
jcmd $PID VM.command_line
jcmd $PID VM.flags | grep -E 'MaxHeapSize|MaxDirectMemorySize'
jcmd $PID VM.system_properties | grep jdk.nio.maxCachedBufferSize
```

### 线程

```bash
jcmd $PID Thread.print -l > /tmp/kafka-threads.txt

grep '^"' /tmp/kafka-threads.txt |
  grep -c 'kafka-network-thread'

grep '^"ReplicaFetcherThread-' /tmp/kafka-threads.txt |
  wc -l
```

### Direct Buffer对象

```bash
jcmd $PID GC.class_histogram -all |
  grep -E 'DirectByteBuffer|MappedByteBuffer|Util\$BufferCache|Deallocator'
```

### JMX/Prometheus

```bash
curl -s http://127.0.0.1:<metrics-port>/metrics |
  grep -iE 'buffer.*direct|direct.*buffer'
```

### Heap Dump

```bash
jcmd $PID GC.heap_info
jcmd $PID help GC.heap_dump
jcmd $PID GC.heap_dump /independent-disk/kafka-direct-$PID.hprof
```

## 14. 最终结论

本次Direct Memory OOM由多个因素叠加：

```text
200个 Remote Log Reader * 约55 MiB缓存
+ 40个外部SASL网络Processor的大请求缓存
+ 40个Request Handler的大请求写入缓存
+ ReplicaFetcher及少量其他NIO缓存
= 约15.96 GiB
```

其中决定性因素是：

```text
remote.log.reader.threads=200
+ fetch.max.bytes=55 MiB
+ JDK临时direct buffer无限缓存
```

最优先整改项是降低Remote Log Reader线程数，并通过滚动重启释放旧线程缓存；随后限制JDK缓存、重新评估Fetch/请求尺寸和线程数量。单纯增加 `MaxDirectMemorySize` 只会推迟故障，并增加进程或容器被操作系统OOM Kill的风险。

2026-08-24 复查显示，Remote Log Reader 线程数和缓存已显著下降，但网络 Processor 与 Request Handler 的大请求缓存仍在增长。下一步除了继续控制 Remote Log Reader 并发，还必须定位并限制 50-79 MiB 的客户端请求来源，否则仅降低远程读取线程数不能彻底消除 Direct Memory 风险。
