---
title: "KRaft observer metadata 停止复制排查报告"
description: "KRaft observer 在 Fetch 超时后持续向非 leader controller 请求 metadata 的完整排查过程、证据链与 KAFKA-19354 根因分析"
date: 2026-11-12
tags: ['kafka', 'operations', 'kraft', 'raft', 'incident']
---

**故障日期**：2026-11-12  
**集群**：`kafka-pro-c10`  
**故障节点**：broker 68（`10.54.45.36:10001`）  
**Controller leader**：1003，Raft epoch 643  
**Kafka 版本**：3.9.2 基线，包含内部定制  
**最终定位**：[KAFKA-19354](https://issues.apache.org/jira/browse/KAFKA-19354)

## 1. 文档目的

本文按实际排查顺序记录故障定位过程，而不是只记录最终根因。每个阶段保留：

1. 当时看到的原始证据。
2. 根据证据提出的怀疑方向。
3. 为验证方向补充的检查。
4. 哪些假设被排除或修正。
5. 新证据如何将排查推进到下一阶段。

最终结论是：broker 68 的 observer 在一次 Raft 请求超时后进入 bootstrap 选路状态，持续向非 leader controller 发送 Fetch。所有响应均返回 `NOT_LEADER_OR_FOLLOWER`，并明确告知真正的 leader 是 1003，但 3.9.2 的旧逻辑不能使 observer 回到该 leader，形成永久循环。

## 2. 最初现象与原始证据

### 2.1 单节点 metadata lag 告警

最早发现的是：

```text
kafka_server_broker_metadata_metrics_last_applied_record_lag_ms
```

只有 broker 68 的指标从 11:34 左右开始升高。到 12:05 左右已经达到约 31.4 分钟，曲线近似以墙钟速度线性增长；同集群其他 broker 保持在约 1.04 秒。

源码中的指标定义为：

```scala
addMetric(metrics, lastAppliedRecordLagMsName) { now =>
  now - lastAppliedImageProvenance.get.lastContainedLogTimeMs()
}
```

因此该指标不是简单的“leader offset - broker offset”，而是：

```text
当前时间 - 最后一个已应用 metadata record 的日志时间戳
```

线性增长表示最后一个已应用 record 的时间戳不再更新。

**当时的第一层判断：**

- 如果所有 broker 同时升高，可能是 controller 停写或没有新 metadata record。
- 只有一台升高，应优先排查该 broker 的 Raft 拉取、日志追加或 metadata 发布链路。

### 2.2 `kafka-raft.log` 在 11:34 后停止

12:00:51 检查日志时，最后几条正常复制日志停在 11:33:55：

```text
[2026-11-12 11:33:54,733] DEBUG [RaftManager id=68] Follower high watermark updated to 35678649
[2026-11-12 11:33:54,733] DEBUG [RaftManager id=68] Notifying listener org.apache.kafka.image.loader.MetadataLoader@1328400246 of batch for baseOffset 35678648 and lastOffset OptionalLong.empty

[2026-11-12 11:33:55,233] DEBUG [RaftManager id=68] Follower high watermark updated to 35678650
[2026-11-12 11:33:55,233] DEBUG [RaftManager id=68] Notifying listener org.apache.kafka.image.loader.MetadataLoader@1328400246 of batch for baseOffset 35678649 and lastOffset OptionalLong.empty
```

最后一条日志是 11:34:01.382：

```text
[2026-11-12 11:34:01,382] DEBUG [RaftManager id=68] Ignoring response InboundResponse(
  correlationId=20449817,
  data=FetchResponseData(
    errorCode=0,
    responses=[... PartitionData(
      errorCode=0,
      highWatermark=35678651,
      logStartOffset=35504828,
      divergingEpoch=EpochEndOffset(epoch=-1, endOffset=-1),
      currentLeader=LeaderIdAndEpoch(leaderId=1003, leaderEpoch=643),
      snapshotId=SnapshotId(endOffset=-1, epoch=-1),
      records=MemoryRecords(size=72, ...))],
    nodeEndpoints=[NodeEndpoint(
      nodeId=1003,
      host='10.30.7.85',
      port=8081)]),
  source=10.30.7.85:8081 (id: 1003))
  since it is no longer needed
```

这条响应本身完全正常：

- 来自当时的 leader 1003。
- `errorCode=0`。
- `leaderEpoch=643`。
- leader 的高水位为 `35678651`。
- 没有要求 truncation 或 snapshot。

真正异常的是它被 `RequestManager` 判定为不再需要。

**当时形成的候选方向：**

1. 请求超时、响应迟到或 Raft 状态变化导致 Fetch 链路中断。
2. Raft 仍复制，但 MetadataLoader 或 publisher 阻塞。
3. 长 GC、磁盘 I/O 或其他停顿导致线程无法推进。
4. `Ignoring response` 可能只是正常竞态，不能直接把它当根因。

## 3. 第一阶段：通过线程堆栈判断卡点

### 3.1 最初的 Raft 线程堆栈

raft I/O 线程：

```text
"kafka-68-raft-io-thread"
java.lang.Thread.State: TIMED_WAITING (parking)
  at java.util.concurrent.LinkedBlockingQueue.poll
  at org.apache.kafka.raft.internals.BlockingMessageQueue.poll(BlockingMessageQueue.java:48)
  at org.apache.kafka.raft.KafkaRaftClient.poll(KafkaRaftClient.java:3334)
  at org.apache.kafka.raft.KafkaRaftClientDriver.doWork(KafkaRaftClientDriver.java:64)
```

raft 网络发送线程：

```text
"kafka-68-raft-outbound-request-thread"
java.lang.Thread.State: RUNNABLE
  at sun.nio.ch.EPoll.wait(Native Method)
  at org.apache.kafka.common.network.Selector.select(Selector.java:895)
  at org.apache.kafka.common.network.Selector.poll(Selector.java:485)
  at org.apache.kafka.clients.NetworkClient.poll(NetworkClient.java:596)
  at org.apache.kafka.server.util.InterBrokerSendThread.pollOnce(InterBrokerSendThread.java:110)
```

两个线程都在等待事件，没有显示正在执行的异常路径。

**当时的初步推断：**

- 不像是 raft I/O 线程正在执行某个死循环或阻塞式磁盘调用。
- 可能是 `RequestManager` 仍认为有 in-flight 请求，I/O 线程等待响应；网络线程又在等待 socket 事件。
- 但一次线程快照不能证明持续状态，也不能证明没有发送其他请求。

这个“in-flight 请求卡住”的方向后来被网络指标推翻。

### 3.2 完整 jstack 排除 metadata publisher 阻塞

11:56:09 的完整堆栈中，metadata loader 为：

```text
"kafka-68-metadata-loader-event-handler"
java.lang.Thread.State: WAITING (parking)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await
  at org.apache.kafka.queue.KafkaEventQueue$EventHandler.handleEvents(KafkaEventQueue.java:282)
  at org.apache.kafka.queue.KafkaEventQueue$EventHandler.run(KafkaEventQueue.java:186)
```

完整堆栈没有：

- `BLOCKED` 线程；
- Java deadlock 报告；
- metadata loader 进入 `ReplicaManager.applyDelta`；
- publisher 等待业务锁或阻塞 I/O。

**该证据排除了：**

- metadata 已复制但无法发布；
- 某个 publisher 在处理特定 record 时永久卡住；
- Raft 与 metadata publisher 之间的 Java 锁死锁。

**该证据没有排除：**

- MetadataLoader 没收到事件，因为 Raft 没有有效追加；
- 某个内部 `BatchReader` 状态异常，因为没有对象级指标；
- Raft 请求在网络上持续循环但没有有效响应。

因此排查重点继续放在 Raft 复制链路。

## 4. 第二阶段：ISR shrink 与 GC 的干扰及排除

### 4.1 大量 ISR shrink 集中发生

11:34:01.357 至 11:34:01.398 的约 40ms 内，大量本地 leader 分区同时收缩 ISR，例如：

```text
[2026-11-12 11:34:01,357] INFO [Partition potential-content-merge-pb-367 broker=68]
Shrinking ISR from 13,40,68 to 40,68.
Leader: (highWatermark: 153941535, endOffset: 153941538).
Out of sync replicas:
(brokerId: 13, endOffset: 153941535, lastCaughtUpTimeMs: 1794454435282).
```

以及：

```text
[2026-11-12 11:34:01,368] INFO [Partition video-stream-engine-push-log-233 broker=68]
Shrinking ISR from 4,36,68 to 68.
Leader: (highWatermark: 821178741, endOffset: 821178743).
Out of sync replicas:
(brokerId: 4, endOffset: 821178741, lastCaughtUpTimeMs: 1794454435418)
(brokerId: 36, endOffset: 821178741, lastCaughtUpTimeMs: 1794454435416).
```

这些日志与 Raft 停止发生在同一秒，且涉及不同 topic、不同 follower。

`lastCaughtUpTimeMs=1794454435xxx` 对应 11:33:55 CST，与 ISR shrink 时间相差约 6 秒。若实际 `replica.lag.time.max.ms` 配置为约 5000ms，这与一次过期检查后集中补跑相符。

**当时提出的方向：**

- 是否发生了全 JVM 停顿，导致 Raft 请求过期、响应迟到？
- 长 GC 或 safepoint 是否是故障入口？

这个方向只能解释最初几秒，不能直接解释之后持续数小时的 lag。

### 4.2 GC 原始日志排除长 GC

故障窗口附近的原始 GC 日志：

```text
[2026-11-12T11:33:45.098+0800] GC(985282) Pause Young ... start
[2026-11-12T11:33:45.108+0800] GC(985282) ... 12410M->2595M(16384M) 10.152ms

[2026-11-12T11:34:01.337+0800] GC(985283) Pause Young ... start
[2026-11-12T11:34:01.348+0800] GC(985283) ... 12403M->2601M(16384M) 11.781ms

[2026-11-12T11:34:09.720+0800] GC(985284) Pause Young ... start
[2026-11-12T11:34:09.731+0800] GC(985284) ... 12401M->2598M(16384M) 11.221ms

[2026-11-12T11:34:19.944+0800] GC(985285) Pause Young ... start
[2026-11-12T11:34:19.955+0800] GC(985285) ... 12398M->2597M(16384M) 11.203ms
```

后续 GC 继续运行，均约 10ms。

**修正结论：**

- 可以排除长 GC。
- ISR shrink 集中出现说明当时可能存在短暂调度、处理停顿或日志突发，但不是持续故障的根因。
- 11:34:01 的响应被忽略可能是一次请求超时的结果；真正需要解释的是之后为什么无法恢复。

## 5. 第三阶段：通过指标收敛故障层级

### 5.1 `fetch-records-rate` 与 HW 同时停止

监控显示：

```text
kafka_server_raft_metrics_fetch_records_rate
```

在 11:34 后从约 2 records/s 降为 0。

同时：

```text
kafka_server_raft_metrics_high_watermark = 35678650
```

之后一直保持不变。

**当时判断：**

- 不是只有 metadata lag 指标异常，Raft 确实不再追加 record。
- MetadataLoader 空闲是结果，而不是根因。

### 5.2 LEO 与 epoch 的补充证据

后续指标：

```text
current_epoch = 643
log_end_epoch = 643
log_end_offset = 35678651
high_watermark = 35678650
poll_idle_ratio_avg 约为 0.988
```

其中：

- `current_epoch` 始终为 643，没有 leader 换届。
- LEO 为 `35678651`，表示本地日志下一个待写 offset。
- HW 为 `35678650`，停留在 LEO 前一条 record。
- I/O 线程仍持续执行 poll 循环，并非线程退出。

**收敛出的关键问题：**

在 leader epoch 没变的情况下，如果本机向真正的 leader 发送 `fetchOffset=35678651`，只要 leader 返回 `highWatermark=35678651`，本机即使没有新 record 可追加，也应将 HW 从 `35678650` 更新到 `35678651`。

所以后面必须解释：**为什么请求/响应没有走到有效的 leader Fetch 处理路径。**

## 6. 第四阶段：TCP 连接造成的误判与纠正

### 6.1 `ss` 显示到 1003 的连接健康

故障 broker 到 1003 有两条连接：

```text
ESTAB 0 0 [::ffff:10.54.45.36]:56262 -> [::ffff:10.30.7.85]:8081
ESTAB 0 0 [::ffff:10.54.45.36]:50918 -> [::ffff:10.30.7.85]:8081
```

详细 TCP 信息：

```text
Recv-Q=0
Send-Q=0
bytes_acked=3255639
bytes_received=1547070
lastsnd=1873
lastrcv=1872
```

**当时的初步判断：**

- 没有 TCP 发送队列堵塞。
- 没有已接收但应用未读取的数据。
- 连接仍有约 2 秒一次的双向流量。

但不能据此断定 Raft Fetch 正常，因为同一对地址上可能同时存在 heartbeat、forwarding 和 Raft 通道。

### 6.2 抓包证明该连接是 heartbeat

对 `56262` 抓包：

```text
00000024 003f 0001 004db1b8 0002 3638 ...
```

解析：

```text
00000024 = Kafka message size 36
003f     = apiKey 63 = BROKER_HEARTBEAT
0001     = apiVersion 1
004db1b8 = correlationId
0002     = clientId length 2
3638     = clientId "68"
```

响应包含：

```text
r-48-fetcher-0
```

这是 controller incarnation ID。请求每 2 秒一次，与 heartbeat 完全吻合。

**修正结论：**

- 到 1003 的活跃连接是 heartbeat，不是 Raft Fetch。
- broker 在 KRaft 中仍在线，是因为 heartbeat 独立正常。
- 之前“TCP 有流量说明 Raft 仍在通信”的推断不成立。

这一步是排查中的重要转折：不能只看 IP、端口和 TCP 状态，必须看 Kafka 协议与 clientId。

## 7. 第五阶段：网络指标推翻“完全不发送 Fetch”

### 7.1 `raft-channel-metrics` 的原始趋势

专用 Raft NetworkClient 指标显示：

- `request_total` 从约 24.4M 增长到 26.45M。
- `response_total` 同步增长到约 26.47M。
- `outgoing_byte_rate` 约 92 KB/s。
- `incoming_byte_rate` 约 63 KB/s。
- `connection_creation_total` 保持 149，没有增长。

Raft selector 的指标在 `Selector.recordCompletedSend` 和 `recordCompletedReceive` 时递增，分别对应真实完成的网络发送与完整响应接收。

**这直接修正了此前“没有新 Fetch”的怀疑：**

- Raft 客户端一直在发送请求。
- 对端也一直在返回完整响应。
- 没有持续重建连接。
- 故障不是没有网络请求，而是请求/响应未产生有效复制。

### 7.2 当时剩下的候选方向

结合 LEO/HW 不动，剩余方向变为：

1. 持续 FetchSnapshot 下载，因为 snapshot 不增加 `fetch-records-rate`。
2. 响应被 `RequestManager` 持续忽略。
3. 请求发往非 leader，持续收到 `NOT_LEADER_OR_FOLLOWER`。
4. 其他特殊响应，例如 truncation、snapshot 或 epoch 异常。

## 8. 第六阶段：排除 snapshot 下载

检查 `metadata.log.dir` 下的 `__cluster_metadata-0`：

```text
-rw-r--r-- 1 kafka kafka 1436954 Nov 12 10:41 00000000000035672301-0000000643.checkpoint
-rw-rw-r-- 1 kafka kafka      17 Nov 12 10:41 leader-epoch-checkpoint
-rw-rw-r-- 1 kafka kafka 72711414 Nov 12 11:33 00000000000034668778.log
```

目录中：

- 没有 `.checkpoint.part`。
- 最新 checkpoint 是 10:41 的 `35672301`。
- 活跃日志段 mtime 停在 11:33。

**修正结论：**

- 没有正在下载的 snapshot。
- 没有反复遗留的部分 snapshot 文件。
- 网络流量不是由持续 FetchSnapshot 引起。

此时，最值得检查的就是 Fetch 的目的节点和响应 errorCode。

## 9. 第七阶段：搜索社区 issue 并建立初步关联

在最终 TRACE 前，已检索 Apache JIRA 和 GitHub，主要候选为：

### KAFKA-19354

社区描述：

- observer 的 Fetch 遇到网络延迟或超时后可能重新 bootstrap。
- 如果选中 follower，observer 会持续向 follower 发 Fetch。
- 持续收到 `NOT_LEADER_OR_FOLLOWER`，metadata replication 停止。
- 重启 observer 可能恢复。

这与已有证据有较高相关性：

- 有一次请求响应被忽略，符合超时入口。
- 后续没有 leader epoch 变化。
- heartbeat 正常但复制停止。
- 网络仍有大量请求和响应。

但当时还缺少直接证据，因为尚未证明请求确实发往非 leader，也未看到 `errorCode=6`。

### KAFKA-20514

该问题描述 observer 可能卡在不可路由的 leader 地址，未可靠回退到 bootstrap。

与本例不完全一致：

- 本例到 leader 1003 的地址可路由且曾成功响应。
- 最终流量显示的是持续 bootstrap follower 循环，而不是卡在不可达 leader。

### KAFKA-15100

该问题涉及 leader 处理其他 replica Fetch 时，timeout 回调跨线程操作 Raft 状态，修复于 3.4.2/3.5.2/3.6.0。

本例是 broker observer 的发送与选路问题，版本为 3.9.2，不匹配。

**这一阶段的结论：**

KAFKA-19354 是最相关候选，但不能仅凭“observer 卡住”相似就下结论，必须抓到非 leader 请求和 `NOT_LEADER_OR_FOLLOWER` 响应。

## 10. 第八阶段：抓包证明请求发往三个非 leader

对 `controller01/controller02/controller03` 的连接抓包，找到大量：

```text
clientId="raft-client-68"
```

请求头及 Fetch 字段：

```text
0001 = apiKey 1 = FETCH
0011 = apiVersion 17
```

请求中：

```text
fetchOffset=0x022069bb = 35678651
currentLeaderEpoch=0x0283 = 643
lastFetchedEpoch=0x0283 = 643
```

响应中：

```text
errorCode=0x0006 = 6 = NOT_LEADER_OR_FOLLOWER
leaderId=0x03eb = 1003
leaderEpoch=0x0283 = 643
```

三个响应都包含 leader endpoint：

```text
nodeId=1003
host='10.30.7.85'
port=8081
```

这证明：

- 请求不是发往真正的 leader 1003。
- 三个非 leader 都明确告知真正 leader 是 1003。
- 客户端仍继续向这些非 leader 发请求。
- 这与 KAFKA-19354 的故障模型高度一致。

同时，之前三条连接“对称大流量”的原因也清楚了：它们都是 broker 68 主动发出的 `raft-client-68` 请求，而不是其他 controller 到 broker 的无关流量。

## 11. 最终确认：故障 broker 的 TRACE 完整还原无限循环

**仅在故障 broker 68 上开启 TRACE** 后获得决定性证据。以下所有日志均由故障 broker 的 `[RaftManager id=68]` 输出：`Sent outbound request` 是 broker 68 作为 Raft 客户端发送请求，`Received inbound message` 是 broker 68 收到响应。**没有开启 controller 的 TRACE，也没有使用各 controller 的服务端日志。**

### 故障 broker 发往 controller01 的请求与响应

```text
[2026-11-12 15:38:01,495] TRACE [RaftManager id=68] Sent outbound request: OutboundRequest(correlationId=28740860, data=FetchRequestData(clusterId='bLGO4qbsQ36xkG5C46g65A', replicaId=-1, replicaState=ReplicaState(replicaId=68, replicaEpoch=-1), maxWaitMs=500, minBytes=0, maxBytes=8388608, isolationLevel=0, sessionId=0, sessionEpoch=-1, topics=[FetchTopic(topic='__cluster_metadata', topicId=AAAAAAAAAAAAAAAAAAAAAQ, partitions=[FetchPartition(partition=0, currentLeaderEpoch=643, fetchOffset=35678651, lastFetchedEpoch=643, logStartOffset=-1, partitionMaxBytes=0, replicaDirectoryId=FGXyMU9b4wyKh0PtY5aB-g)])], forgottenTopicsData=[], rackId=''), createdTimeMs=1794469081495, destination=kraft-c10-controller01.kafka.data.sina.com.cn:8081 (id: -2 rack: null)) (org.apache.kafka.raft.KafkaRaftClient)
[2026-11-12 15:38:01,496] TRACE [RaftManager id=68] Received inbound message InboundResponse(correlationId=28740860, data=FetchResponseData(throttleTimeMs=0, errorCode=0, sessionId=0, responses=[FetchableTopicResponse(topic='', topicId=AAAAAAAAAAAAAAAAAAAAAQ, partitions=[PartitionData(partitionIndex=0, errorCode=6, highWatermark=-1, lastStableOffset=-1, logStartOffset=35540568, divergingEpoch=EpochEndOffset(epoch=-1, endOffset=-1), currentLeader=LeaderIdAndEpoch(leaderId=1003, leaderEpoch=643), snapshotId=SnapshotId(endOffset=-1, epoch=-1), abortedTransactions=[], preferredReadReplica=-1, records=MemoryRecords(size=0, buffer=java.nio.HeapByteBuffer[pos=0 lim=0 cap=38]))])], nodeEndpoints=[NodeEndpoint(nodeId=1003, host='10.30.7.85', port=8081, rack=null)]), source=kraft-c10-controller01.kafka.data.sina.com.cn:8081 (id: -2 rack: null)) (org.apache.kafka.raft.KafkaRaftClient)
```

### 故障 broker 发往 controller02 的请求与响应

```text
[2026-11-12 15:38:01,496] TRACE [RaftManager id=68] Sent outbound request: OutboundRequest(correlationId=28740861, data=FetchRequestData(clusterId='bLGO4qbsQ36xkG5C46g65A', replicaId=-1, replicaState=ReplicaState(replicaId=68, replicaEpoch=-1), maxWaitMs=500, minBytes=0, maxBytes=8388608, isolationLevel=0, sessionId=0, sessionEpoch=-1, topics=[FetchTopic(topic='__cluster_metadata', topicId=AAAAAAAAAAAAAAAAAAAAAQ, partitions=[FetchPartition(partition=0, currentLeaderEpoch=643, fetchOffset=35678651, lastFetchedEpoch=643, logStartOffset=-1, partitionMaxBytes=0, replicaDirectoryId=FGXyMU9b4wyKh0PtY5aB-g)])], forgottenTopicsData=[], rackId=''), createdTimeMs=1794469081496, destination=kraft-c10-controller02.kafka.data.sina.com.cn:8081 (id: -3 rack: null)) (org.apache.kafka.raft.KafkaRaftClient)
[2026-11-12 15:38:01,498] TRACE [RaftManager id=68] Received inbound message InboundResponse(correlationId=28740861, data=FetchResponseData(throttleTimeMs=0, errorCode=0, sessionId=0, responses=[FetchableTopicResponse(topic='', topicId=AAAAAAAAAAAAAAAAAAAAAQ, partitions=[PartitionData(partitionIndex=0, errorCode=6, highWatermark=-1, lastStableOffset=-1, logStartOffset=35543450, divergingEpoch=EpochEndOffset(epoch=-1, endOffset=-1), currentLeader=LeaderIdAndEpoch(leaderId=1003, leaderEpoch=643), snapshotId=SnapshotId(endOffset=-1, epoch=-1), abortedTransactions=[], preferredReadReplica=-1, records=MemoryRecords(size=0, buffer=java.nio.HeapByteBuffer[pos=0 lim=0 cap=38]))])], nodeEndpoints=[NodeEndpoint(nodeId=1003, host='10.30.7.85', port=8081, rack=null)]), source=kraft-c10-controller02.kafka.data.sina.com.cn:8081 (id: -3 rack: null)) (org.apache.kafka.raft.KafkaRaftClient)
```

### 故障 broker 再次发往 controller01 的请求与响应

```text
[2026-11-12 15:38:01,498] TRACE [RaftManager id=68] Sent outbound request: OutboundRequest(correlationId=28740862, data=FetchRequestData(clusterId='bLGO4qbsQ36xkG5C46g65A', replicaId=-1, replicaState=ReplicaState(replicaId=68, replicaEpoch=-1), maxWaitMs=500, minBytes=0, maxBytes=8388608, isolationLevel=0, sessionId=0, sessionEpoch=-1, topics=[FetchTopic(topic='__cluster_metadata', topicId=AAAAAAAAAAAAAAAAAAAAAQ, partitions=[FetchPartition(partition=0, currentLeaderEpoch=643, fetchOffset=35678651, lastFetchedEpoch=643, logStartOffset=-1, partitionMaxBytes=0, replicaDirectoryId=FGXyMU9b4wyKh0PtY5aB-g)])], forgottenTopicsData=[], rackId=''), createdTimeMs=1794469081498, destination=kraft-c10-controller01.kafka.data.sina.com.cn:8081 (id: -2 rack: null)) (org.apache.kafka.raft.KafkaRaftClient)
[2026-11-12 15:38:01,499] TRACE [RaftManager id=68] Received inbound message InboundResponse(correlationId=28740862, data=FetchResponseData(throttleTimeMs=0, errorCode=0, sessionId=0, responses=[FetchableTopicResponse(topic='', topicId=AAAAAAAAAAAAAAAAAAAAAQ, partitions=[PartitionData(partitionIndex=0, errorCode=6, highWatermark=-1, lastStableOffset=-1, logStartOffset=35540568, divergingEpoch=EpochEndOffset(epoch=-1, endOffset=-1), currentLeader=LeaderIdAndEpoch(leaderId=1003, leaderEpoch=643), snapshotId=SnapshotId(endOffset=-1, epoch=-1), abortedTransactions=[], preferredReadReplica=-1, records=MemoryRecords(size=0, buffer=java.nio.HeapByteBuffer[pos=0 lim=0 cap=38]))])], nodeEndpoints=[NodeEndpoint(nodeId=1003, host='10.30.7.85', port=8081, rack=null)]), source=kraft-c10-controller01.kafka.data.sina.com.cn:8081 (id: -2 rack: null)) (org.apache.kafka.raft.KafkaRaftClient)
```

### 故障 broker 发往 controller03 的请求与响应

```text
[2026-11-12 15:38:01,499] TRACE [RaftManager id=68] Sent outbound request: OutboundRequest(correlationId=28740863, data=FetchRequestData(clusterId='bLGO4qbsQ36xkG5C46g65A', replicaId=-1, replicaState=ReplicaState(replicaId=68, replicaEpoch=-1), maxWaitMs=500, minBytes=0, maxBytes=8388608, isolationLevel=0, sessionId=0, sessionEpoch=-1, topics=[FetchTopic(topic='__cluster_metadata', topicId=AAAAAAAAAAAAAAAAAAAAAQ, partitions=[FetchPartition(partition=0, currentLeaderEpoch=643, fetchOffset=35678651, lastFetchedEpoch=643, logStartOffset=-1, partitionMaxBytes=0, replicaDirectoryId=FGXyMU9b4wyKh0PtY5aB-g)])], forgottenTopicsData=[], rackId=''), createdTimeMs=1794469081499, destination=kraft-c10-controller03.kafka.data.sina.com.cn:8081 (id: -4 rack: null)) (org.apache.kafka.raft.KafkaRaftClient)
[2026-11-12 15:38:01,502] TRACE [RaftManager id=68] Received inbound message InboundResponse(correlationId=28740863, data=FetchResponseData(throttleTimeMs=0, errorCode=0, sessionId=0, responses=[FetchableTopicResponse(topic='', topicId=AAAAAAAAAAAAAAAAAAAAAQ, partitions=[PartitionData(partitionIndex=0, errorCode=6, highWatermark=-1, lastStableOffset=-1, logStartOffset=35369727, divergingEpoch=EpochEndOffset(epoch=-1, endOffset=-1), currentLeader=LeaderIdAndEpoch(leaderId=1003, leaderEpoch=643), snapshotId=SnapshotId(endOffset=-1, epoch=-1), abortedTransactions=[], preferredReadReplica=-1, records=MemoryRecords(size=0, buffer=java.nio.HeapByteBuffer[pos=0 lim=0 cap=38]))])], nodeEndpoints=[NodeEndpoint(nodeId=1003, host='10.30.7.85', port=8081, rack=null)]), source=kraft-c10-controller03.kafka.data.sina.com.cn:8081 (id: -4 rack: null)) (org.apache.kafka.raft.KafkaRaftClient)
```

至此，故障循环被完整证明：

```text
controller01 -> errorCode=6, leader=1003
controller02 -> errorCode=6, leader=1003
controller01 -> errorCode=6, leader=1003
controller03 -> errorCode=6, leader=1003
```

所有请求 offset 固定为 `35678651`，没有任何请求转到真正的 leader 1003。

`destination id=-2/-3/-4` 是 bootstrap 节点 ID，也证明客户端正在 bootstrap 列表中选路。

## 12. 将 TRACE 与 KAFKA-19354 和源码逐项对应

### 12.1 社区故障模型匹配

| KAFKA-19354 特征 | 本例证据 |
|------------------|----------|
| observer 遇到请求失败或超时 | 11:34:01 的有效响应因 `no longer needed` 被忽略，之前请求状态已失效 |
| observer 重新 bootstrap | TRACE 中 destination 为 bootstrap id -2/-3/-4 |
| bootstrap 可能选中 follower | 请求发往 controller01/02/03，真正 leader 是 1003 |
| 持续收到 `NOT_LEADER_OR_FOLLOWER` | 所有 TRACE 和抓包响应均为 `errorCode=6` |
| 响应中包含 leader 信息但无法恢复 | 所有响应均为 `leaderId=1003, leaderEpoch=643`，仍不回到 1003 |
| metadata replication 停止 | LEO/HW 不动，`fetch-records-rate=0`，applied lag 线性增长 |
| heartbeat 可保持正常 | 单独抓包确认 `BROKER_HEARTBEAT` 每 2 秒正常执行 |

### 12.2 3.9.2 代码路径匹配

旧 observer 选路：

```java
private long pollFollowerAsObserver(FollowerState state, long currentTimeMs) {
    if (state.hasFetchTimeoutExpired(currentTimeMs)) {
        return maybeSendFetchToAnyBootstrap(currentTimeMs);
    } else {
        return maybeSendFetchToBestNode(state, currentTimeMs);
    }
}
```

一旦 fetch timeout 到期，observer 固定在 bootstrap 中选路。

非 leader 响应进入：

```java
} else if (epoch > quorum.epoch()
    || error == Errors.FENCED_LEADER_EPOCH
    || error == Errors.NOT_LEADER_OR_FOLLOWER) {
    maybeTransition(leaderId, epoch, leaderEndpoints, currentTimeMs);
    return Optional.of(true);
}
```

本例 epoch 相同，本机已经知道 leader 1003，所以不会产生新的 leader 状态变化。响应被标记为已处理，清理请求状态后继续选择 bootstrap。

这还解释了为什么日志中没有持续 ERROR：

- `NOT_LEADER_OR_FOLLOWER` 是公共响应处理中的正常控制流；
- 不会进入 `Unexpected error ... in FETCH response`；
- 每次请求和响应都成功完成，所以也没有持续 timeout 或 disconnect。

### 12.3 修复提交匹配

官方修复：

- JIRA：[KAFKA-19354](https://issues.apache.org/jira/browse/KAFKA-19354)
- PR：[apache/kafka#19854](https://github.com/apache/kafka/pull/19854)
- Commit：[7339d65b27fd5c1aa5ba239faf193ee0ff55f06d](https://github.com/apache/kafka/commit/7339d65b27fd5c1aa5ba239faf193ee0ff55f06d)

核心修改：

```java
private long pollFollowerAsObserver(FollowerState state, long currentTimeMs) {
    return maybeSendFetchToBestNode(state, currentTimeMs);
}
```

删除“fetch timeout 后只走 bootstrap”的固定分支，使 observer 重新根据 leader 和请求状态选择最佳节点。

官方提交还修改 `FollowerState` 及相应测试，backport 时应保留完整提交。

## 13. 排查过程中被排除或修正的方向汇总

| 曾经的方向 | 支持它的早期证据 | 排除或修正它的后续证据 |
|------------|------------------|------------------------|
| 全局 controller 停写 | 单节点 lag 增长 | 其他 broker lag 始终约 1 秒 |
| MetadataLoader/publisher 阻塞 | metadata 不再应用 | jstack 中 loader 空队列等待，无业务锁和 publisher 调用 |
| 长 GC | 11:34:01 大量 ISR shrink 与 Raft 日志同时出现 | GC 全部约 10ms，后续持续正常 |
| Java 锁死锁 | 多个线程都在等待 | 完整 jstack 无 `BLOCKED`、无 deadlock，均为正常队列/epoll 等待 |
| 完全不发送新 Fetch | raft 日志停止、I/O 等队列 | `request_total/response_total` 持续增长 |
| TCP 连接断开或队列堵塞 | metadata 停止复制 | 连接 ESTAB、队列均为 0、持续有流量 |
| 到 1003 的活跃流量是 Raft | `lastsnd/lastrcv` 很新 | 抓包证明是 `BROKER_HEARTBEAT`、clientId 为 `68` |
| 持续 FetchSnapshot | 网络流量大但 fetch-records-rate=0 | 无 `.checkpoint.part`，无 snapshot 下载痕迹 |
| 单纯 in-flight 请求永久卡住 | raft I/O 与网络线程都在等待 | 网络请求/响应持续完成，实际是快速错误响应循环 |
| KAFKA-20514 | observer 可能卡住 | 本例 leader 地址可达，且实际卡在 bootstrap follower 轮询 |
| KAFKA-19354 | observer 超时后无法恢复 | 抓包和 TRACE 完整证明非 leader Fetch 与 errorCode=6 循环，最终确认 |

## 14. 完整证据链

```text
单个 broker metadata lag 线性增长，其他 broker 正常
  -> 排除 controller 全局停写

metadata loader 空队列等待，Raft HW/LEO 不动
  -> 排除 publisher 阻塞，定位到有效 Raft 复制停止

长 GC 不存在，无 Java deadlock
  -> 排除 JVM 长停顿和锁死锁

heartbeat 每 2 秒正常
  -> 解释 broker 仍在线，但不能证明 Raft 正常

Raft request/response 总数持续增长
  -> 排除完全不发送请求和 TCP 中断

无 snapshot 部分文件
  -> 排除持续 snapshot 下载

抓包发现 raft-client-68 向三个非 leader 发 Fetch
  -> 目的节点错误，而非网络不可达

所有响应 errorCode=6，并携带 leaderId=1003、epoch=643
  -> 非 leader 已明确告知正确 leader

TRACE 证明客户端仍按 bootstrap 顺序轮询，永不回到 1003
  -> 完整匹配 KAFKA-19354

3.9.2 旧 observer 选路与公共响应处理解释循环为何持续
  -> 根因闭环
```

## 15. 后续修复与复盘建议

1. 将 KAFKA-19354 官方修复提交完整 backport 到 3.9.2 内部分支。
2. 增加“observer 从 bootstrap follower 响应发现 leader 后回到 leader”的测试。
3. 对连续 `NOT_LEADER_OR_FOLLOWER` 响应增加限速日志或计数，包含 source 和响应中的 leaderId/epoch。
4. 监控中联合使用 `fetch-records-rate`、LEO、HW、applied offset 和 Raft 网络流量，避免把“有流量”当作复制健康。
5. 排查 controller 网络问题时，应同时检查协议 apiKey、clientId 和 destination，不能只依赖 TCP 状态或 heartbeat。

