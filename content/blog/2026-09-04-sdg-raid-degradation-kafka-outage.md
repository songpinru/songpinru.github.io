---
title: "共享 RAID 故障域导致 Kafka fence 排查报告"
description: "共享 RAID 故障域引发日志写入阻塞、Kafka KRaft 心跳超时与 broker fence 的排查证据及整改建议"
date: 2026-09-04
categories: ["incident-review"]
tags: ["kafka", "kraft", "高可用"]
---

| | |
|---|---|
| 日期 | 2026-09-04 |
| 故障主机 | 10.26.11.56 |
| 故障窗口 | 12:13:48 ~ 12:15:02(存储侧);12:14:08(broker 17482 fence) |
| 运行模式 | KRaft |
| 存储拓扑 | `lsscsi`:sda~sdk 同属 MR9361-8i;sdg 为 `[0:2:6:0]`;NVMe 不受影响 |
| 结论状态 | **共享 RAID 故障域已确认;broker 17482 fence 已确认;log4j 同步写阻塞/appender 锁 convoy 已确认;常规 GC STW 已排除;heartbeat 线程直接传导路径待补** |

---

## 1. 结论(TL;DR)

**已确认:sda~sdk 同属一张 MR9361-8i,监控显示非 NVMe 盘在同窗口全部 IO 100% 持续数十秒;sdg 的读请求出现 Medium Error / Unrecoverable read error,单条命令耗时 29~74 秒;broker 17482 在 12:14:08 被 Controller fence。Kafka 日志在同一 appender 中出现约 68 秒断崖,恢复后同毫秒爆发并出现密集时间戳回跳,确认 log4j 同步写/appender 锁发生阻塞。GC pause 最大 18.253ms,常规 GC STW 排除;Controller 无切换和异常日志,网络/CPU/内存/进程重启均已排除。**

代码事实是:KRaft broker 默认每 2 秒发送心跳,Controller 侧 9 秒无成功心跳即 fence;heartbeat 通道的 RPC 超时也是 2 秒。现在已有 fence 日志、同卡归属、全盘 IO 饱和、日志断崖、恢复爆发和同文件时间戳回跳证据,因果链可以定性为“共享 RAID 故障域 → sda 停滞 → broker 多线程停顿 → 心跳超时 → fence”。appender 锁 convoy 是已确认的 broker 内阻塞/放大机制,且至少影响 Processor、NetworkClient、ReplicaFetcher 线程;但它是否就是 heartbeat 线程的直接停顿原因,仍缺故障期线程栈或直接日志痕迹。

---

## 2. 故障现象与内核日志解读

### 2.1 关键内核日志

```
Sep 04 12:14:18 10.26.11.56 kernel: sd 0:2:6:0: [sdg] tag#10 FAILED Result: hostbyte=DID_OK driverbyte=DRIVER_SENSE cmd_age=29s
Sep 04 12:14:18 10.26.11.56 kernel: sd 0:2:6:0: [sdg] tag#10 Sense Key : Medium Error [current]
Sep 04 12:14:18 10.26.11.56 kernel: sd 0:2:6:0: [sdg] tag#10 Add. Sense: Unrecovered read error
Sep 04 12:15:02 10.26.11.56 kernel: megaraid_sas 0000:e3:00.0: 3978 (841838369s/0x0001/FATAL) - Uncorrectable medium error logged for VD 06/6 at 3895cc00 (on PD 14(e0x08/s7) at 3895cc00)
Sep 04 12:15:02 10.26.11.56 kernel: megaraid_sas 0000:e3:00.0: 3979 (841838369s/0x0002/FATAL) - Unrecoverable medium error during recovery on PD 14(e0x08/s7) at 3895dc00
```

### 2.2 逐条解读

| 证据 | 含义 |
|---|---|
| `Unrecoverable medium error **during recovery** on PD 14` | 控制器报告 PD 14 在恢复/重建相关路径上出现不可恢复介质错误;是否已有前序故障盘、重建进度如何,需 `storcli` 确认 |
| `cmd_age=29s/38s/72s/74s` | 这些 sdg 读命令从下发到失败耗时 29~74 秒。该现象与控制器/队列级异常或恢复负载一致,但仅凭这些记录不能证明控制器上所有 VD 均停滞 |
| `VD 06/6 at 3895cc00 (on PD 14 ... at 3895cc00)` | 控制器把错误归属到 VD 06 / PD 14。sdg 与 VD 06 的稳定映射需用 `lsscsi`/`storcli`/sysfs 再确认 |

### 2.3 CDB 解码(已核实)

`Read(16)` CDB 的 LBA 位于第 2~9 字节，传输长度位于第 10~13 字节。本批 CDB 的传输长度均为 `0x00000200` 个 512B sector，即 `0x40000` 字节，与控制器 debug 行的 requested length 一致。LBA 与内核 `blk_update_request` 中的 sector 值一一对应，例如:

| CDB LBA | 十进制 sector |
|---|---:|
| `0x3895ca22` | 949340706 |
| `0x3895cc22` | 949341218 |
| `0x3895ce22` | 949341730 |
| `0x3895d022` | 949342242 |
| `0x3895da22` | 949344802 |
| `0x3895dc22` | 949345314 |
| `0x3895de22` | 949345826 |

这部分证明内核日志内部自洽，且确认为 256KiB 读请求失败。

### 2.4 同卡影响与 Kafka 侧证据(已更新)

`lsscsi` 显示 `sda`~`sdk` 均挂在同一颗 `AVAGO MR9361-8i` 上,`sdg` 的 SCSI 地址为 `[0:2:6:0]`。监控进一步显示,除 NVMe 外的所有盘在同窗口 IO 100% 持续数十秒。二者合起来已经确认:

1. `sda` 与 `sdg` 属同一控制器;
2. 同窗口内该控制器上的多个逻辑盘同时出现 IO 饱和;
3. NVMe 不受影响,排除了整机 IO 子系统同时故障。

原始 `lsscsi` 输出:

```text
[0:0:8:0]    enclosu LSI      Cub_1            0903  -
[0:2:0:0]    disk    AVAGO    MR9361-8i        4.68  /dev/sda
[0:2:1:0]    disk    AVAGO    MR9361-8i        4.68  /dev/sdb
[0:2:2:0]    disk    AVAGO    MR9361-8i        4.68  /dev/sdc
[0:2:3:0]    disk    AVAGO    MR9361-8i        4.68  /dev/sdd
[0:2:4:0]    disk    AVAGO    MR9361-8i        4.68  /dev/sde
[0:2:5:0]    disk    AVAGO    MR9361-8i        4.68  /dev/sdf
[0:2:6:0]    disk    AVAGO    MR9361-8i        4.68  /dev/sdg
[0:2:7:0]    disk    AVAGO    MR9361-8i        4.68  /dev/sdh
[0:2:8:0]    disk    AVAGO    MR9361-8i        4.68  /dev/sdi
[0:2:9:0]    disk    AVAGO    MR9361-8i        4.68  /dev/sdj
[0:2:10:0]   disk    AVAGO    MR9361-8i        4.68  /dev/sdk
```

该输出与内核日志中的 `sd 0:2:6:0: [sdg]` 完全对应,确认报错设备就是同一 SCSI host/channel 下的 `/dev/sdg`。

Kafka 侧已有关键结果证据:

```text
[2026-09-04 12:14:08,119] INFO [QuorumController id=1004]
Fencing broker 17482 because its session has timed out.
```

按默认 9 秒会话超时（需以线上实际 `broker.session.timeout.ms` 复核）反推,broker 17482 的最后一次成功心跳不晚于 **12:13:59**。内核日志中最早的 sdg 命令约在 **12:13:48** 下发,二者相差约 11 秒,足够覆盖“IO 饱和开始 → 心跳停止 → 9 秒超时 → fence”的时间窗。

GC 日志检查结果为正常,仅有 young GC,最高 pause 18.253ms,因此 **常规 GC STW 不是本次 fence 的原因**。Controller 无 leader 切换、无异常日志,网络/CPU/内存正常,进程未重启,故障后自动恢复。这些证据共同把根因收敛到共享 RAID 故障域。

### 2.5 GC 日志时间线

故障前的 young GC 约每 4.4~5.7 秒发生一次;`12:13:53.113` 之后直到 `12:15:03.860` 才出现下一条 GC 记录,间隔约 70.747 秒。GC 日志时间戳由 JVM 在事件发生时生成,窗口内没有任何 GC 事件,恢复后也没有接近 9 秒的 pause 记录;更合理的解读是应用线程停顿或请求量骤降导致分配停止,GC 自然没有触发。由于 GC 日志可能同样位于 `sda`,仍需确认其路径和完整性,但现有证据足以排除常规 GC STW 作为本次 fence 原因。

GC 日志原文:

```text
[2026-09-04T12:13:01.186+0800][gc          ] GC(549584) Concurrent Undo Cycle
[2026-09-04T12:13:01.195+0800][gc          ] GC(549584) Concurrent Undo Cycle 8.981ms
[2026-09-04T12:13:06.001+0800][gc          ] GC(549585) Pause Young (Normal) (G1 Evacuation Pause) 12436M->1955M(16384M) 7.530ms
[2026-09-04T12:13:11.212+0800][gc          ] GC(549586) Pause Young (Normal) (G1 Evacuation Pause) 11779M->1967M(16384M) 7.437ms
[2026-09-04T12:13:16.703+0800][gc          ] GC(549587) Pause Young (Normal) (GCLocker Initiated GC) 11791M->1966M(16384M) 8.265ms
[2026-09-04T12:13:22.171+0800][gc          ] GC(549588) Pause Young (Normal) (G1 Evacuation Pause) 11774M->1968M(16384M) 7.482ms
[2026-09-04T12:13:27.870+0800][gc          ] GC(549589) Pause Young (Normal) (GCLocker Initiated GC) 11776M->1967M(16384M) 7.318ms
[2026-09-04T12:13:33.136+0800][gc          ] GC(549590) Pause Young (Normal) (GCLocker Initiated GC) 11775M->1954M(16384M) 8.191ms
[2026-09-04T12:13:38.724+0800][gc          ] GC(549591) Pause Young (Normal) (G1 Evacuation Pause) 11754M->1953M(16384M) 16.900ms
[2026-09-04T12:13:43.631+0800][gc          ] GC(549592) Pause Young (Normal) (G1 Evacuation Pause) 13841M->1985M(16384M) 8.670ms
[2026-09-04T12:13:48.025+0800][gc          ] GC(549593) Pause Young (Normal) (GCLocker Initiated GC) 12793M->1974M(16384M) 7.520ms
[2026-09-04T12:13:53.113+0800][gc          ] GC(549594) Pause Young (Normal) (GCLocker Initiated GC) 11790M->1987M(16384M) 9.228ms
[2026-09-04T12:15:03.860+0800][gc          ] GC(549595) Pause Young (Normal) (G1 Evacuation Pause) 11795M->2092M(16384M) 18.253ms
[2026-09-04T12:15:04.250+0800][gc          ] GC(549596) Pause Young (Normal) (G1 Evacuation Pause) 4676M->2034M(16384M) 7.304ms
[2026-09-04T12:15:38.183+0800][gc          ] GC(549597) Pause Young (Normal) (G1 Evacuation Pause) 11794M->2007M(16384M) 11.395ms
[2026-09-04T12:16:17.219+0800][gc          ] GC(549598) Pause Young (Normal) (G1 Evacuation Pause) 11791M->2009M(16384M) 15.098ms
[2026-09-04T12:17:00.144+0800][gc          ] GC(549599) Pause Young (Normal) (G1 Evacuation Pause) 11793M->2024M(16384M) 15.297ms
[2026-09-04T12:17:43.026+0800][gc          ] GC(549600) Pause Young (Normal) (G1 Evacuation Pause) 11800M->2011M(16384M) 15.663ms
```

判读结论:

| 指标 | 值 | 结论 |
|---|---:|---|
| 故障前 young GC 间隔 | 约 4.4~5.7s | 分配/GC 周期稳定 |
| 最后一次故障前 GC | 12:13:53.113,9.228ms | 与最后一次成功心跳窗口接近 |
| 下一次 GC | 12:15:03.860,18.253ms | 与存储故障恢复窗口吻合 |
| GC 事件间隔 | 约 70.747s | 不是 GC pause,而是无 GC 事件 |
| 记录到的 pause 最大值 | 18.253ms | 远低于 9s fence 阈值 |

### 2.6 Kafka 日志断崖、回跳与恢复爆发(关键证据)

正常情况下 server/client 日志每秒都有输出。故障窗口内,同一 appender 的日志从 `12:13:54.703` 直接跳到 `12:15:02.717`,断崖约 **68.014 秒**:

```text
[2026-09-04 12:13:53,633] DEBUG Processor 34 listening to new connection from /REFID_003Q.43:49420 (kafka.network.Processor)
[2026-09-04 12:13:53,638] INFO Received request which need to authenticate with username: ... (org.apache.kafka.common.security.authenticator.SaslServerAuthenticator)
[2026-09-04 12:13:54,703] DEBUG Processor 35 listening to new connection from /REFID_004Q.66:34810 (kafka.network.Processor)
[2026-09-04 12:15:02,717] DEBUG Processor 35 listening to new connection from /REFID_005Q.116:11190 (kafka.network.Processor)
[2026-09-04 12:15:02,717] DEBUG Processor 35 listening to new connection from /REFID_006Q.190:14648 (kafka.network.Processor)
[2026-09-04 12:15:02,717] DEBUG Processor 35 listening to new connection from /REFID_007Q.36:41174 (kafka.network.Processor)
[2026-09-04 12:15:02,717] DEBUG Processor 35 listening to new connection from /REFID_008Q.161:11228 (kafka.network.Processor)
```

随后在 `12:15:02.773~12:15:02.828` 内集中出现大量 `NetworkClient` request timeout 日志,涉及 remote log metadata consumer 和多个 ReplicaFetcher:

```text
[2026-09-04 12:15:02,773] INFO [Consumer clientId=__remote_log_metadata_client_17482_consumer, groupId=null] Disconnecting from node 17482 due to request timeout. (org.apache.kafka.clients.NetworkClient)
[2026-09-04 12:15:02,774] INFO [ReplicaFetcher replicaId=17482, leaderId=2384, fetcherId=7] Disconnecting from node 2384 due to request timeout. (org.apache.kafka.clients.NetworkClient)
[2026-09-04 12:15:02,774] INFO [ReplicaFetcher replicaId=17482, leaderId=14208, fetcherId=2] Disconnecting from node 14208 due to request timeout. (org.apache.kafka.clients.NetworkClient)
[2026-09-04 12:15:02,824] INFO [Consumer clientId=__remote_log_metadata_client_17482_consumer, groupId=null] Disconnecting from node 2386 due to request timeout. (org.apache.kafka.clients.NetworkClient)
[2026-09-04 12:15:02,828] INFO [Consumer clientId=__remote_log_metadata_client_17482_consumer, groupId=null] Disconnecting from node 14206 due to request timeout. (org.apache.kafka.clients.NetworkClient)
```

线上按相邻行扫描同一日志文件,格式为 `行号 上一行时间 -> 当前行时间`;右侧时间小于左侧,即当前行的写出顺序晚于上一行,但事件创建时间更早。代表性样本:

```text
653155 2026-09-04 11:58:12,419 -> 2026-09-04 11:58:12,418
669561 2026-09-04 12:15:02,718 -> 2026-09-04 12:15:02,717
669685 2026-09-04 12:15:02,720 -> 2026-09-04 12:15:02,718
669732 2026-09-04 12:15:02,769 -> 2026-09-04 12:15:02,722
669754 2026-09-04 12:15:02,770 -> 2026-09-04 12:15:02,769
669817 2026-09-04 12:15:02,773 -> 2026-09-04 12:15:02,770
669853 2026-09-04 12:15:02,773 -> 2026-09-04 12:15:02,772
689277 2026-09-04 12:24:30,357 -> 2026-09-04 12:24:30,333
689279 2026-09-04 12:24:30,331 -> 2026-09-04 12:24:30,087
689282 2026-09-04 12:24:30,012 -> 2026-09-04 12:24:29,919
689287 2026-09-04 12:24:29,471 -> 2026-09-04 12:24:29,450
689292 2026-09-04 12:24:29,330 -> 2026-09-04 12:24:29,087
689296 2026-09-04 12:24:28,857 -> 2026-09-04 12:24:28,669
```

其中 `12:15:02.718~775` 是密集回跳,最大相邻跨度约 48ms;`12:24:30` 附近存在连续反序段,写出顺序从 `12:24:30.357` 一路降到 `12:24:28.549`,跨度约 1.808s。`11:58:12` 的孤立 1ms 回跳说明毫秒级偶发反序可由并发 append 顺序造成,不能单独当作故障证据;故障恢复点的密集回跳与 68 秒断崖、同毫秒爆发组合在一起,才构成 appender 锁 convoy 的强证据。这个现象非常关键:

1. log4j 的时间戳在 logging event 创建时生成,不是 `write(2)` 返回后生成;
2. `AppenderSkeleton.doAppend` 是 synchronized,等待锁的线程各自持有已创建的 event;
3. 锁释放后,线程按获得锁的顺序写出,不保证按 event 创建时间排序;
4. 因此同一文件中出现时间戳回跳,正是 appender 锁 convoy 解除后的典型指纹。

结合三点事实:

- Kafka 数据路径在 NVMe,已确认的 `sda` Kafka 路径是 log4j 日志;系统盘也在 `sda`;
- sda 与 sdg 同属 MR9361-8i,且同窗口 IO 100%;
- 日志断崖、同毫秒爆发、同文件时间戳回跳同时出现;

可以确认 **log4j 同步写/appender 锁阻塞是 broker 内已确认的阻塞/放大机制**。它至少影响了 Processor 网络线程和多个 NetworkClient/ReplicaFetcher 线程;heartbeat 线程是否也直接阻塞在同一锁上,仍需故障期线程栈或直接日志痕迹确认,但时间线上与心跳超时吻合。

仍需说明:日志断崖、同毫秒爆发和同文件时间戳回跳已经确认 appender 写阻塞这一机制;线程栈只能进一步指出第一个卡在 `write(2)`/`flush()` 的线程,以及 heartbeat 线程当时是否直接等待同一把 appender 锁。不能把“日志线程被 convoy 捕获”直接外推为“heartbeat 线程必然在同一锁上排队”。

> 注:若数据盘确为 NVMe 直连,数据日志路径本身不经 megaraid;但这不自动排除网络、CPU、内存、GC、配置或 Controller 侧原因。

---

## 3. 阻塞机制:三条通道

### 3.0 前置事实:线上日志配置的 IO 特征

线上使用 log4j 1.x(reload4j 1.2.25),配置了 **10 个** `SinaDailyRollingFileAppender`(all、server、state-change、request、controller、coordinator、log、client、raft、tiered),且未配置 `AsyncAppender`。经确认,该自定义类继承标准 `DailyRollingFileAppender`,仅增加日志删除功能,无其他改动,因此适用以下 reload4j 行为。

对 reload4j 1.2.25 反汇编确认的锁语义(详见附录 B):

| 机制 | 后果 |
|---|---|
| `AppenderSkeleton.doAppend` 为 **synchronized** | 同一 appender 的所有写日志线程在一把锁上串行 |
| `Category.callAppenders` 持 logger 类目监视器遍历 appender | 同一 logger 上的写线程二次串行化 |
| `immediateFlush` 默认 **true** | 每条日志事件会强制 flush;通常产生底层 `write(2)`,可能多次,但这不是 `fsync` |
| `DailyRollingFileAppender.rollOver`(close+delete+rename+open)在锁内执行 | 若沿用标准实现,滚动窗口内锁持有时间可能被磁盘延迟放大(本窗口非整点,未触发,列为隐患) |

#### 3.0.1 reload4j 源码级传导机制

按 Maven Central `reload4j-1.2.25-sources.jar` 源码核对,Kafka 的同步文件日志调用链为:

```text
kafka.utils.Logging.debug/info/warn/error
→ SLF4J Reload4jLoggerAdapter
→ org.apache.log4j.Category.log
→ Category.forcedLog
→ new LoggingEvent(...)
→ Category.callAppenders(event)
→ AppenderAttachableImpl.appendLoopOnAppenders(event)
→ AppenderSkeleton.doAppend(event)
→ WriterAppender.append(event)
→ DailyRollingFileAppender.subAppend(event)
→ WriterAppender.subAppend(event)
→ QuietWriter.write(layout.format(event))
→ QuietWriter.flush()
→ OutputStreamWriter / FileOutputStream
→ write(2)
```

reload4j 关键源码行为如下:

1. **日志 IO 发生在 Kafka 调用线程内**。`FileAppender` 直接创建 `FileOutputStream`,`WriterAppender.subAppend` 在当前线程执行格式化、写出和 flush;没有 `AsyncAppender` 时没有后台线程隔离。`immediateFlush=true` 不是 `fsync`,而是把 Java writer 的编码缓冲刷到底层输出流,最终可能触发一次或多次 `write(2)`。
2. **双重串行化**。`Category.callAppenders` 对 logger category 对象执行 `synchronized(c)`;随后 `AppenderSkeleton.doAppend` 本身也是 `synchronized`。当前写日志线程若卡在底层 write,同时持有 category monitor 和 appender monitor。同一 logger 上的其他线程会先卡 category monitor;不同 logger 但落到同一 appender 的线程会卡 appender monitor。没有超时、降级或丢弃策略。
3. **异常日志放大锁持有时间**。`WriterAppender.subAppend` 对 throwable 的每一行分别调用 `qw.write(line)` 和 `qw.write(Layout.LINE_SEP)`。带堆栈的 ERROR 日志会增加格式化、编码和系统调用次数,进一步放大同步锁持有时间。
4. **日志时间戳在 event 创建时生成**。`LoggingEvent` 构造函数中执行 `timeStamp = System.currentTimeMillis()`,发生在进入 appender 锁之前。多个线程竞争 category/appender monitor 的顺序不等于 event 创建顺序,因此物理文件中后续行的日志时间戳可以小于前一行。这解释了同文件时间戳回跳,也说明事故窗口日志时间戳只能代表 event 创建时间,不能代表写盘完成时间。
5. **滚动操作也在锁内**。`DailyRollingFileAppender.subAppend` 到点后先执行 `rollOver()`,再调用 `super.subAppend(event)`。`rollOver` 包含 `closeFile → File.delete → File.renameTo → setFile/open`,均发生在 `AppenderSkeleton.doAppend` 的 synchronized 调用链内。本窗口不是滚动点,但这是长期锁持有风险。

这组源码事实把 broker 内影响面收敛为:

```text
sda 日志文件 write/flush 停滞
→ 当前 Kafka 线程在 synchronized doAppend 内阻塞
→ appender monitor 长期被持有
→ 其他写同一 appender/category 的线程 BLOCKED
→ Processor / request handler / NetworkClient / ReplicaFetcher 停止推进
→ 客户端超时、重连、副本拉取失败
```

具体 Kafka 路径:

- `SocketServer.Processor.run()` 的主循环顺序执行 `configureNewConnections → processNewResponses → poll → ...`;`Processor listening to new connection` 是 DEBUG 日志。该日志一旦卡在 appender,当前 Processor 无法继续 poll 或处理收发包,影响其负责的所有连接。
- `KafkaRequestHandler` 的异常处理路径会输出 `Exception when handling request` 并携带异常堆栈;请求线程卡住后不再消费 request channel 或执行 callback。
- `NetworkClient.handleTimedOutRequests` 会输出 `Disconnecting from node ... due to request timeout`;该日志在对应 NetworkClient/ReplicaFetcher 线程执行,卡住后线程无法继续 poll、重连或重试 fetch。
- 恢复期大量连接同时超时又会产生多条 INFO 日志,形成日志量放大和锁竞争的正反馈。

heartbeat 需要单独限定:

- 正常心跳发送路径是 TRACE(`BrokerLifecycleManager.scala:463-465`);RUNNING 状态处理响应是 DEBUG(`BrokerLifecycleManager.scala:535-537`);heartbeat request thread 的正常轮询路径不打日志。若这些 logger 的生效级别是 INFO,不能仅由 server 日志断崖推出 heartbeat 线程必然进入 appender。
- 若 DEBUG/TRACE 生效,lifecycle 事件线程在处理响应时存在直接日志路径。
- 若 heartbeat 请求在发送队列超时,`BrokerHeartbeatResponseHandler.onTimeout()` 会先输出 INFO,再把 timeout event `prepend` 回 lifecycle event queue。该 INFO 一旦被 appender 卡住,heartbeat request thread 无法继续,lifecycle 侧 `communicationInFlight` 保持 true,也不会发起下一次心跳;这条路径足以导致 9 秒会话超时。

因此,源码确认了“日志写可以阻塞 Kafka 关键线程”的机制,也给出了 heartbeat 被阻塞的条件化路径;但本次是否正是这些路径,仍需生效日志级别、恢复期 heartbeat 日志或故障期线程栈确认。

#### 3.0.2 `AsyncAppender` 能否消除该风险

结论:**默认 `AsyncAppender` 不能防住本次 68 秒级日志盘停顿;必须使用有界队列 + `Blocking=false`,并确保 Kafka 线程没有直接可达的同步文件 appender。**

reload4j 1.2.25 `AsyncAppender` 源码语义:

| 事实 | 源码行为 | 对本事故的含义 |
|---|---|---|
| 默认队列容量 | `DEFAULT_BUFFER_SIZE = 128` 条事件 | 对持续数十秒的停顿几乎必然填满 |
| 默认队列满策略 | `blocking = true`,调用线程在 `buffer.wait()` 等待 | 只是把同步写阻塞推迟到队列满之后,Kafka 线程仍会被拖住 |
| 等待位置 | `AsyncAppender.append()` 在 `AppenderSkeleton.doAppend` 的 synchronized 调用链内执行 | 队列满时仍可能把写同一 AsyncAppender 的其他线程串住 |
| 后台 dispatcher | 从队列取事件后同步调用底层文件 appender | 底层 sda write 卡住时 dispatcher 卡住,队列随后填满 |
| `Blocking=false` | 队列满时调用线程把事件放入 discard summary 并立即返回 | 这是保住 Processor/NetworkClient/heartbeat 等 Kafka 线程的关键语义 |
| 丢弃可见性 | dispatcher 稍后输出 `Discarded N messages due to full event buffer including: ...` | 故障时会丢日志,必须接受并监控丢弃量 |
| 入队前预处理 | 获取 NDC、thread name、MDC copy、rendered message 和 throwable stack trace | 队列按事件数计容量,但异常堆栈会显著增加内存占用 |
| dispatcher 死亡 | `append()` 退回同步调用底层 appender | 需要监控 dispatcher 存活/处理进度 |

因此,整改不是简单把 appender class 改成 `org.apache.log4j.AsyncAppender`。目标配置必须满足:

1. **有界队列**:按 `峰值日志速率 × 期望容忍的日志盘停顿时长` 估算,并结合事件平均内存评估 JVM 压力;不能无脑调大。
2. **`Blocking=false`**:宁可丢弃日志,也不让 Kafka 调用线程等待日志盘。
3. **覆盖所有可达 appender**:只包 `serverAppender` 不够。若 logger 经 additivity 仍能同步到达 `allAppender`、`clientAppender`、`controllerAppender`、`raftAppender` 等,对应 Kafka 线程仍会被未异步化的 appender 卡住。
4. **不要双路径引用**:同一个底层文件 appender 不能同时被 AsyncAppender 包装,又被 logger/root 直接同步引用。
5. **配置方式**:reload4j `AsyncAppender` 官方注释说明应通过 XML DOMConfigurator 配置;不能假设 `log4j.properties` 里的 `appender-ref` 一定生效。若维持 properties 体系,应实现自定义异步 appender。
6. **可观测性**:原生 `AsyncAppender` 未内置完整 JMX 指标,需自定义或通过日志输出监控队列水位、丢弃事件数和 dispatcher 存活状态。

目标 XML 形态如下(队列大小需按线上峰值重算):

```xml
<appender name="asyncServerAppender" class="org.apache.log4j.AsyncAppender">
  <param name="BufferSize" value="50000"/>
  <param name="Blocking" value="false"/>
  <appender-ref ref="serverFileAppender"/>
</appender>
```

同时必须保留存储隔离:`AsyncAppender(blocking=false)` 只是把日志盘慢写与 Kafka 线程解耦,不修复 MR9361-8i 共享故障域,也不消除 GC 日志、stdout、类加载、dirty throttling 等其他 sda IO 路径风险。

### 3.1 通道 A:appender 锁 convoy(已确认影响日志线程)

Kafka 所有组件经由 `Logging` trait(`core/src/main/scala/kafka/utils/Logging.scala:43`)打日志,绝大多数落到 `serverAppender`(kafka.* / org.apache.kafka.* 均 INFO 级绑定)。

可能的日志放大源(是否实际发生需查故障窗口日志):

- `SocketServer.scala:1126` — `Processor $id poll failed`,网络线程主循环(`SocketServer.scala:1020-1037`)的 poll 异常路径;
- `SocketServer.scala:1055` — `error(s"Closing socket for $channelId because of error", e)`,**每个异常连接一条,带完整堆栈多行输出**;
- `KafkaRequestHandler.scala:146/165` — `Exception when handling request`,请求线程异常路径;
- `NetworkClient.java:843/849/871/1021` — 客户端/副本 fetcher 断连、重连失败各一条 INFO(大量连接同时超时时成灾);
- 配置中 `kafka.network.Processor=DEBUG`(`SocketServer.scala:1298`,每个新建连一条)在重连风暴期间放大流量。

**成立条件已满足**:Kafka 数据路径在 NVMe,已确认的 `sda` Kafka 路径是 log4j 日志;sda 同窗口 IO 100%;同一 appender 文件出现约 68 秒断崖、恢复后同毫秒爆发和密集时间戳回跳。由于 `SinaDailyRollingFileAppender` 继承标准 `DailyRollingFileAppender` 且只增加删除功能,其 `doAppend` 锁与 `WriterAppender` 写出路径适用标准 reload4j 语义。

因此可以确认:某个线程在 `append()`/`write()`/`flush()` 中被 sda IO 卡住,后续写同一 appender 的网络线程、请求线程和 fetcher 线程在锁上排队。`12:15:02` 的同毫秒连接日志、密集时间戳回跳和大量 request timeout 日志是锁释放后的恢复爆发。客户端 Producer/Consumer 的 `request.timeout.ms` 默认为 30 秒(线上配置可能不同),超时后的重连可能进一步增加日志量。

### 3.2 已排除的通道 B:常规 GC STW

部署链路事实:

- `bin/kafka-server-start.sh:32` 默认携带 `-loggc`;
- `bin/kafka-run-class.sh` 在 `KAFKA_GC_LOG_OPTS` 为空时启用 GC 日志,并在新 JDK 下生成 `-Xlog:gc*:file=$LOG_DIR/kafkaServer-gc.log:...`;
- `LOG_DIR` 默认为 `$base_dir/logs`;线上是否通过环境变量改到 sda,需要按进程环境和 fd 实测。

本次故障已检查 `kafkaServer-gc.log`:仅有 young GC,最高 pause 18.253ms;同时 `12:13:53.113~12:15:03.860` 没有任何 GC 事件,恢复后也没有接近 9 秒的 pause 记录。GC 日志时间戳由 JVM 在事件发生时生成,因此该证据比单纯“日志文件在窗口内没有新增字节”更强。仍需确认 GC 日志实际路径和完整性;在完成该确认前,结论表述为“常规 GC STW 已排除”,而不是排除所有 IO 阻塞。

限制:排除常规 GC STW 不等于排除其他 IO 阻塞。70 秒无 GC 事件更可能反映应用分配/活动停顿,而不是 JVM 正在执行一次 70 秒 GC;具体阻塞线程仍需线程级证据确认。

### 3.3 候选通道 C:脏页回写节流与内存压力

若日志/GC 写入产生大量脏页且 sda 回写停滞,`balance_dirty_pages()` 可能使后续写线程阻塞;脏页累积和 direct reclaim 也可能让请求处理路径在内存分配上变慢。这是背景性放大假设,需要 `/proc/vmstat`、`sar -B`、dirty/writeback 采样或历史监控验证。

### 3.4 条件汇合点:心跳中断 → fence

心跳链路涉及两个线程。通道 A 已确认会影响 Processor 和 NetworkClient/ReplicaFetcher 线程;heartbeat 事件线程或 heartbeat RPC 线程若在故障窗口进入日志路径或等待同一 appender 锁,同样会中断。但需要区分“心跳停止”与“心跳线程直接被 appender 锁阻塞”:在当前线上 INFO 生效级别下,正常心跳发送路径是 TRACE(`BrokerLifecycleManager.scala:463-465`),RUNNING 状态处理响应是 DEBUG(`BrokerLifecycleManager.scala:535-537`),heartbeat request thread 的正常轮询路径不产生日志。因此,不能仅由 server 日志断崖推出 heartbeat 线程必然进入 appender;若实际开启 DEBUG/TRACE,或恢复日志中出现这些日志,才存在直接代码路径。

1. `BrokerLifecycleManager` 事件队列线程(心跳调度与发送,`BrokerLifecycleManager.scala:585-594`,间隔 `broker.heartbeat.interval.ms`=2s);
2. `NodeToControllerRequestThread`(实际收发 RPC,`NodeToControllerChannelManager.scala:246-262`)。`BrokerServer.scala:361-368` 为 heartbeat 通道传入 `retryTimeoutMs=broker.heartbeat.interval.ms`;`NodeToControllerChannelManager.scala:188` 再取 `min(controller.socket.timeout.ms=30s, retryTimeoutMs=2s)`,因此默认 heartbeat RPC 超时是 **2 秒**,不是 30 秒。

心跳断后,Controller 侧会话计时器到点(`KRaftConfigs.java:52`,`broker.session.timeout.ms` 默认 **9000ms**),由 `QuorumController` 调度 fence 检查(`QuorumController.java:1324-1331`),执行:

```java
// metadata/src/main/java/org/apache/kafka/controller/ReplicationControlManager.java:1640
log.info("Fencing broker {} because its session has timed out.", brokerId);
```

以上代码路径已确认:fence 记录写入元数据日志后会触发副本从 ISR 剔除和 leader 变更(`ReplicationControlManager.java:1319-1330`)。本次故障已有实际 fence 日志:

```text
[2026-09-04 12:14:08,119] INFO [QuorumController id=1004]
Fencing broker 17482 because its session has timed out.
```

因此“心跳中断 → 会话超时 → Controller fence”这一段已经由日志闭环;上游的 broker 多线程停顿也由日志断崖、同毫秒爆发和时间戳回跳确认。剩余问题不只是缺少 heartbeat 线程的直接栈帧,还缺少它进入 appender 日志路径的直接证据;不能区分它是直接等待 appender 锁,还是被 dirty page 回写、类加载或其他 sda 系统路径拖住。

### 3.5 若该主机同时承载 Controller(combined 模式)

需额外关注以下风险:

- `QuorumController` 使用单线程 `KafkaEventQueue`(`QuorumController.java:412-414`),心跳处理与 fence 检查同队列调度。事件线程若因日志锁、GC 或其他原因停顿,可能延迟心跳续约或 fence 检查;是否会造成误 fence 取决于事件顺序和停顿时长;
- 配置中 `org.apache.kafka.controller=DEBUG` + `org.apache.kafka.raft=DEBUG` 会增加日志量,但增加日志量本身不证明事件线程被阻塞;
- Raft 选举超时默认 **1 秒**(`raft/src/main/java/org/apache/kafka/raft/QuorumConfig.java:73`)。若 controller 日志显示 leader 频繁切换,需要区分是本机 IO/GC 停顿、网络问题还是其他 Controller 节点问题。

本次故障中 Controller 无 leader 切换、无异常日志,可排除 Controller 侧停顿作为主因。

---

## 4. 定量对齐

| 参数 | 值 | 代码位置 |
|---|---|---|
| 心跳间隔 | 2s(默认,需复核线上配置) | `KRaftConfigs.java:48` |
| 会话超时(fence 阈值) | **9s**(默认,需复核线上配置) | `KRaftConfigs.java:52` |
| 客户端请求超时(默认) | 30s | clients 默认值 |
| heartbeat RPC 超时 | **2s** | `BrokerServer.scala:361-368` + `NodeToControllerChannelManager.scala:188` |
| controller.socket.timeout.ms | 30s(heartbeat 通道被 2s retryTimeout 压低) | `ReplicationConfigs.java:40` |
| Raft 选举超时 | 1s | `QuorumConfig.java:73` |
| **sdg 单命令延迟(实测)** | **29~74s** | 内核日志 cmd_age |
| **实际 fence 时间(实测)** | **12:14:08** | Controller 日志 |

按 `cmd_age` 回推,可见 sdg 命令最早约 12:13:48/49 下发,最后一批错误在 12:15:02 完成。实际 fence 发生在 12:14:08,按 9 秒会话超时反推,最后一次成功心跳不晚于 12:13:59。这个时间点落在存储故障窗口内,且与 12:13:48 的 IO 异常起点相差约 11 秒,足够覆盖心跳停止后的 9 秒超时。

因此,时间序列上已经支持:

```text
12:13:48 sdg IO 异常开始
  → 12:13:59 前 broker 最后一次成功心跳
  → 9s 无心跳
  → 12:14:08 Controller fence broker 17482
```

剩余不确定的是 12:13:48~12:14:08 之间,broker 内哪个具体路径被 IO 饱和卡住。

---

## 5. 线上日志配置审计

基于完整线上 `log4j.properties`:

### 5.1 与本事故直接相关的问题

| # | 问题 | 影响 |
|---|---|---|
| 1 | 10 个文件 appender 未配置 AsyncAppender,配置路径均基于 `${kafka.logs.dir}` | `SinaDailyRollingFileAppender` 已确认继承标准同步实现,通道 A 具备载体 |
| 2 | GC 日志默认开启,可能位于同一 `LOG_DIR` | 本次已实测 GC pause 最高 18.253ms,常规 GC STW 已排除 |
| 3 | `root=DEBUG` 与注释"兜底...只保留警告+错误"**不符** | 未显式配置的 logger(org.apache.zookeeper、io.netty、com.yammer、scala 库等)以 DEBUG 级进入 all.log;任何新引入库的 DEBUG 输出都是意外流量 |
| 4 | `kafka.network.Processor/Acceptor/SocketServer=DEBUG` | 每连接一条日志;重连风暴期间放大日志量(常态可接受,故障期是正反馈的一环) |
| 5 | `org.apache.kafka.controller=DEBUG` + `org.apache.kafka.raft=DEBUG` | Controller 节点高流量;加剧单事件线程的锁竞争(见 3.5) |

### 5.2 配置修正说明(相对仓库默认配置的差异,均为利好)

- `kafka.request.logger=WARN` —— 常规每请求 DEBUG 审计不生效;仅 deprecated API version 的 INFO 记录仍可能输出,正确;
- `org.apache.kafka.clients=INFO` —— **无**每 RPC DEBUG 洪水(NetworkClient 的 per-RPC debug 不生效),正确;
- stdout ConsoleAppender 已定义但未挂载到 root —— daemon 模式下 `kafkaServer.out` 仅承接 stdout 输出,无日志双写,正确。

### 5.3 次要问题(与本次事故无关,建议一并修复)

| # | 问题 | 说明 |
|---|---|---|
| 6 | `org.I0Itec.zkclient.ZkClient=INFO,serverAppender` 未设 `additivity=false` | 双写:同时进 serverAppender 与 root(allAppender);KRaft 模式下 ZkClient 未启用,暂无实际影响 |
| 7 | `SinaDailyRollingFileAppender` 为定制类,不在本仓库 | 已确认仅继承 `DailyRollingFileAppender` 并增加删除功能;`MaxBackupIndex=14` 的清理逻辑若在 `doAppend` 锁内执行,日滚动(00:00)窗口存在锁长持有风险 |

**已排除的误判**:`SaslServerAuthenticator` / `SaslClientAuthenticator` 不会因未单独设置 `additivity=false` 而双写。reload4j 1.2.25 的传播规则是沿父 logger 逐级判断;父 logger `org.apache.kafka.common.security.authenticator` 已设置 `additivity=false`,因此子 logger 输出 clientAppender 后不会继续传播到 root/allAppender。

---

## 6. ZooKeeper 模式历史事故:待验证的类比

若历史 ZK 事故同样具备“sda IO 停滞 + 日志/其他 IO 阻塞 + 会话心跳中断”，则可按同一机制归因。本次 KRaft 事故中常规 GC STW 通道已排除；历史 ZK 事故仍需单独复验：

```
RAID 控制器降级或 sda 所在故障域停滞
  → 日志盘(sda)write 停滞
  → [A] appender 锁 convoy 冻结网络/请求线程
     [B] GC 日志同步写拉长 STW,冻结全 JVM
     [C] 脏页回写节流 + 内存压力
  → 会话心跳中断
  → 超过会话超时阈值 → 节点被判定下线
  → 副本迁移 / leader 重选 → 客户端服务中断
```

两模式仅"掉线判定"一环不同:

| | KRaft(本次) | ZooKeeper(此前) |
|---|---|---|
| 会话超时 | `broker.session.timeout.ms` = **9s**(`KRaftConfigs.java:52`) | `zookeeper.session.timeout.ms` = **18s**(`server/src/main/java/org/apache/kafka/server/config/ZkConfigs.java:86`) |
| 心跳载体 | BrokerHeartbeat RPC(2s 间隔) | ZK client session 心跳 |
| 掉线动作 | Controller fence(`ReplicationControlManager.java:1640`) | ZK session expired → 临时 broker 节点删除 → 触发重选 |
| 取证日志 | controller.log 的 `Fencing broker` | server.log 的 ZK session expired / controller 的 broker 失效记录 |

ZK 的 18s 窗口比 KRaft 的 9s 宽一倍。若历史事故中确实存在传导到 ZK 客户端线程的 29~74 秒停顿，该停顿足以超过会话窗口；但“两次事故同一根因”必须由各自时间线、线程状态和 IO 指标证明。

---

## 7. KRaft fence 因果链取证结果

`grep "Fencing broker"` 已找到结果证据。结合 `lsscsi`、监控、GC 日志、Kafka 日志断崖/回跳和 Controller 状态，当前证据状态如下：

| 层级 | 当前状态 | 证据 |
|---|---|---|
| L1 故障域 | **已确认** | `lsscsi`:sda~sdk 同属 MR9361-8i;sdg 为 `[0:2:6:0]` |
| L2 sda 停滞 | **已确认** | 监控显示非 NVMe 盘同窗口 IO 100% 持续数十秒 |
| L3 broker/JVM 停顿 | **已确认多线程停顿;appender 阻塞已确认** | 68s 日志断崖、同毫秒爆发、密集同文件时间戳回跳;heartbeat 线程直接传导路径待补 |
| L4 心跳与 fence | **已确认** | Controller 日志:12:14:08 fence broker 17482 |
| L5 排他 | **基本确认** | 网络、CPU、内存正常;进程无重启;Controller 无切换、无异常日志;NVMe 正常 |

当前定性：

1. **共享 RAID 故障域导致 broker 心跳超时并被 fence**：已可定性。
2. **broker 内传导机制**：log4j 同步写/appender 锁 convoy 已确认,并至少影响 Processor、NetworkClient、ReplicaFetcher;heartbeat 线程是直接等待 appender 锁,还是受 dirty page/回写、类加载等更广泛 IO 阻塞影响,待线程级证据。
3. **Controller 侧故障**：已排除，无 leader 切换和异常日志。
4. **网络、CPU、内存、进程重启**：已排除。

要把 heartbeat 线程的精确阻塞点补齐，需要故障期连续线程栈或 JFR IO/monitor 事件。

### 7.1 线上取证操作

| # | 操作 | 判读 |
|---|---|---|
| 1 | 已完成:活跃 Controller `grep -n "Fencing broker.*session has timed out" controller.log*` | 已找到 12:14:08 fence broker 17482 的记录 |
| 2 | 导出 metadata log:`bin/kafka-dump-log.sh --cluster-metadata-decoder --files <metadata-log-segment>` | 核对 `BrokerRegistrationChangeRecord` / fence 记录及其元数据时间戳，与 controller.log 互证 |
| 3 | 已完成:检查同窗口 Controller leader 是否切换、`kafka.controller:type=ControllerEventManager,name=EventQueueTimeMs/EventQueueProcessingTimeMs` 是否突增 | Controller 无 leader 切换、无异常日志 |
| 4 | 故障 broker:查 `Unable to send a heartbeat`、`Unable to send broker heartbeat`、`Processing heartbeat response`、`has been unfenced`、连接/认证异常日志 | broker 侧心跳失败与恢复痕迹;注意成功 heartbeat 默认不逐条记录，日志无失败记录不能反证未中断。若出现 `Processing heartbeat response`,说明 DEBUG 生效且 lifecycle 事件线程确实进入日志路径 |
| 5 | 保留的历史指标:查 Controller 侧 `kafka.network:type=RequestMetrics,name=RequestsPerSec,request=BrokerHeartbeat` 或等价 request metrics | BrokerHeartbeat 速率在同窗口是否归零/骤降 |
| 6 | 若有故障期抓包:过滤 broker→controller 的 BrokerHeartbeat 请求 | 可直接给出最后一次成功心跳时间;这是最精确但通常只有预置采集才有的证据 |
| 7 | 已完成:`kafkaServer-gc.log` / JFR 查 GC pause、safepoint duration | 仅 young GC,最高 18.253ms;常规 GC STW 已排除 |
| 8 | 若有故障期连续 `jstack`/thread dump:查 `broker-<id>-lifecycle-manager-`、`broker-<id>-to-controller-heartbeat-channel-manager` | 期望看到 lifecycle/event 线程停顿,或多个线程 blocked 在 `SinaDailyRollingFileAppender`/`AppenderSkeleton.doAppend`;单次 dump 只能作弱证据 |
| 9 | 已完成:故障期日志行数与相邻行时间戳回跳检查 | server 日志约 68s 断崖;恢复后同毫秒爆发;12:15:02.718~775 密集回跳,12:24:30 附近连续反序约 1.808s,确认 appender 写出顺序与事件创建时间解耦 |
| 10 | 已完成:`sar -d -f /var/log/sa/sa04 -s 12:10:00 -e 12:20:00` 或 node_exporter 历史数据 | 已确认非 NVMe 盘同窗口 IO 100% 持续数十秒 |
| 11 | 已完成:`lsscsi -t`、`readlink /sys/block/sda /sys/block/sdg`、`storcli /c0 show` | 已确认 sda~sdk 同属 MR9361-8i;PD 14 状态仍需 `storcli` 补充 |
| 12 | `findmnt -T <log>`、`ls -l /proc/<kafka-pid>/fd`、`readlink <fd>` | 确认 server.log、controller.log、GC 日志、stdout 实际设备和挂载点，而不是只依赖配置推断 |
| 13 | 复查系统与节点监控:OOM、CPU steal、network retransmit/drop、NVMe error、进程重启、磁盘空间 | 排除替代原因 |

### 7.2 后续可观测性

为下一次事件预置以下能力，可以显著降低取证难度：

1. 保留 node_exporter 的 per-device `node_disk_io_time_seconds_total`、`node_disk_io_time_weighted_seconds_total`、`node_disk_read/write_time_seconds_total`。
2. 保留 Kafka JMX，包括 `BrokerHeartbeat` request rate、Controller event queue time/processing time、request queue time、response send time。
3. 在 staging 或低峰期预置 `jstack` 采集脚本：检测 Kafka 日志或 GC pause 异常时连续抓取 10~20 次线程栈。
4. 对 controller listener 做低开销流量统计或抓包开关，用于精确确定 BrokerHeartbeat 最后成功时间。
5. 在 staging 用受控块设备延迟注入复现“日志盘停滞 → 心跳超时 → fence”链路；不要在生产直接注入故障。
6. 若引入异步日志,增加队列水位、丢弃事件数、dispatcher 存活/处理进度监控;reload4j 原生 `AsyncAppender` 未提供完整 JMX 指标,必要时用自定义 appender 暴露这些指标。

---

## 8. 整改建议

### P0(立即)

1. **更换 PD 14**,巡检阵列(`storcli /c0 show`):确认 PD 14 状态、是否存在前序故障盘、其余 PD 介质错误计数和重建进度。
2. **将日志与 GC 日志迁出 MR9361-8i 故障域**:现已确认 sda~sdk 同属该控制器,且非 NVMe 盘同窗口全部 IO 100%。将 `kafka.logs.dir` 与 `LOG_DIR` 指向容量、寿命和告警均独立的设备,优先选择不受该 RAID 卡影响的 NVMe 或独立存储路径。

### P1(近期)

3. **Appender 异步化,但禁止默认配置直接上线**:默认 `AsyncAppender` 队列仅 128 条且 `Blocking=true`,只能把阻塞推迟到队列满,不能防 68 秒级停顿。必须配置有界队列 + `Blocking=false`,队列大小按 `峰值日志速率 × 期望容忍停顿时长` 估算并评估内存;同时包装所有 Kafka 线程可达的高流量 appender,避免 additivity 或直接引用留下同步路径。reload4j `AsyncAppender` 应通过 XML DOMConfigurator 配置;若维持 properties 体系,优先实现自定义异步 appender。
4. **修正 root 级别**:`log4j.rootLogger=DEBUG` 改为 `WARN`,与注释声明的兜底意图一致;排障结束后回收 `kafka.network.Processor/Acceptor/SocketServer` 的 DEBUG;重新评估 controller/raft DEBUG 常态保留的必要性。
5. **审计 `SinaDailyRollingFileAppender` 的删除逻辑**,确认 MaxBackupIndex 清理是否在 `doAppend` 锁内执行、滚动异常如何处理。另为 `ZkClient` 补 `additivity=false`;SASL logger 经验证无需修复。

### P2(中期)

6. **RAID 卡健康监控告警**:PD medium error 计数、降级/重建状态、控制器温度,提前发现单盘故障。
7. **心跳余量**:`broker.session.timeout.ms` 适度上调(如 15~30s,需与 controller 侧故障检测延迟权衡)作为抖动余量——对 30~74s 量级停顿无效,仅对秒级抖动有效,不能替代 P0/P1。
8. **客户端重连风暴复盘**:批量客户端同时 `request.timeout` 后重连可能放大 broker 侧日志与认证负载,评估客户端侧退避(jitter)配置。

---

## 附录 A:代码证据索引

| 文件 | 位置 | 内容 |
|---|---|---|
| `core/src/main/scala/kafka/network/SocketServer.scala` | 1020-1037 | Processor 网络线程主循环 |
| 同上 | 1055 / 1126 | 通道错误/poll 失败的 error 日志(带堆栈) |
| 同上 | 1298 | 新建连接 DEBUG 日志(重连风暴放大器) |
| `core/src/main/scala/kafka/server/KafkaRequestHandler.scala` | 114 / 146 / 165 | 请求线程取请求(300ms)/ 异常 error 日志 |
| `core/src/main/scala/kafka/utils/Logging.scala` | 43 | 所有组件统一日志入口 |
| `core/src/main/scala/kafka/server/BrokerLifecycleManager.scala` | 463-465 / 480 / 535-537 / 585-594 | 正常心跳 TRACE / 超时 INFO / RUNNING 响应 DEBUG / 2s 调度 |
| `core/src/main/scala/kafka/server/NodeToControllerChannelManager.scala` | 188 / 246-262 | heartbeat RPC 超时与实际请求线程 |
| `core/src/main/scala/kafka/server/BrokerServer.scala` | 361-368 | heartbeat 通道 retryTimeout=2s |
| `server/src/main/java/org/apache/kafka/server/config/KRaftConfigs.java` | 48 / 52 | 心跳 2s / 会话超时 9s 默认值 |
| `server/src/main/java/org/apache/kafka/server/config/ReplicationConfigs.java` | 40 | controller.socket.timeout.ms = 30s |
| `server/src/main/java/org/apache/kafka/server/config/ZkConfigs.java` | 86 | ZK session timeout = 18s |
| `metadata/src/main/java/org/apache/kafka/controller/ReplicationControlManager.java` | 1319-1330 / 1633-1646 | fence 后 ISR/leader 变更、fence 执行与日志 |
| `metadata/src/main/java/org/apache/kafka/controller/QuorumController.java` | 412-414 / 1323-1337 | 单线程事件队列、fence 检查调度 |
| `raft/src/main/java/org/apache/kafka/raft/QuorumConfig.java` | 73 | Raft 选举超时 1s |
| `clients/src/main/java/org/apache/kafka/clients/NetworkClient.java` | 843/849/871/1021 | 断连/重连失败 INFO 日志 |
| `bin/kafka-server-start.sh` | — | 默认 `-name kafkaServer -loggc` |
| `bin/kafka-run-class.sh` | 297-329 / 351 | GC 日志路径 / daemon stdout 重定向 |
| `config/log4j.properties` | 18-28 | 仓库默认配置(与线上差异见第 5 节) |

## 附录 B:reload4j 1.2.25 锁语义与异步边界(源码确认)

从 Maven Central `reload4j-1.2.25-sources.jar` 源码核对,关键锁语义此前亦经项目依赖反汇编复核:

1. **`AppenderSkeleton.doAppend` 为 `public synchronized`** —— 同一 appender 实例上的所有日志写串行;一次阻塞的底层写会使后续写者排队。
2. **`Category.callAppenders`** 字节码为 `monitorenter → appendLoopOnAppenders → monitorexit` —— 写线程持有 logger 类目监视器遍历 appender,同一 logger 的写者二次串行。
3. **`WriterAppender.immediateFlush` 默认 true** —— `subAppend` 中每次 `QuietWriter.write` 后 flush;通常触发底层写,但这不是 `fsync`,且系统调用次数取决于编码器和写入内容。
4. **`DailyRollingFileAppender.subAppend` 触发 `rollOver`**,后者在锁内执行 `closeFile → File.delete → File.renameTo → setFile`;若底层文件操作被磁盘延迟放大,锁持有时间随之放大。
5. **`AsyncAppender` 默认队列 128 条且 `Blocking=true`** —— 队列满时调用线程等待,不能防 68 秒级停顿;必须显式配置有界队列和 `Blocking=false`。

`SinaDailyRollingFileAppender` 为定制类(不在本仓库)。已确认其继承标准 `DailyRollingFileAppender`,仅增加日志删除功能,无其他改动,因此上述锁与写出语义适用。
