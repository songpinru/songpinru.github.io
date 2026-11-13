# KRaft observer 停止复制 metadata 故障排查记录

**日期**：2026-11-12  
**集群**：`kafka-pro-c10`  
**故障节点**：broker 68（`10.54.45.36:10001`）  
**Controller leader**：1003，Raft epoch 643  
**Kafka 基线**：Apache Kafka 3.9.2，包含内部定制  
**最终结论**：命中 [KAFKA-19354](https://issues.apache.org/jira/browse/KAFKA-19354)，observer 在 Fetch 超时后持续向非 leader 的 bootstrap controller 轮询，无法回到真正的 leader，导致 metadata replication 永久停滞。

## 1. 故障现象

故障 broker 的以下指标持续升高：

```text
kafka_server_broker_metadata_metrics_last_applied_record_lag_ms
```

该指标定义为：

```text
当前时间 - 最后一个已应用 metadata record 的时间戳
```

因此，它会随着最后一个已应用 record 变老而近似线性增长，不直接等同于 Raft offset 差值。

监控显示：

- 故障 broker 的 lag 从 11:34 左右开始近似线性增长，约半小时达到 31 分钟。
- 其他 broker 的 lag 始终约为 1 秒。
- 该 broker 的 heartbeat 正常，从 KRaft 视角看仍然在线。

这排除了整个 controller quorum 停止写入 metadata，将问题收敛到单个 broker 的本地复制或发布链路。

## 2. 最终根因摘要

故障 broker 的 Raft 请求始终使用：

```text
fetchOffset=35678651
currentLeaderEpoch=643
lastFetchedEpoch=643
```

但它持续向 `controller01/controller02/controller03` 发送 Fetch，而不是向真正的 leader `1003` 发送。所有响应均为：

```text
errorCode=6 (NOT_LEADER_OR_FOLLOWER)
currentLeader=(leaderId=1003, leaderEpoch=643)
```

3.9.2 的 observer 逻辑在 fetch timeout 到期后只从 bootstrap controller 拉取。非 leader 返回的 `NOT_LEADER_OR_FOLLOWER` 虽然携带真正的 leader 信息，却不会重置 observer 的 fetch timeout，也不会在同 epoch 且已经知道 leader 的情况下使其重新回到 leader。因此形成无限循环：

```text
向 bootstrap follower 发 Fetch
  -> 返回 NOT_LEADER_OR_FOLLOWER 和 leader=1003
  -> 清理当前请求状态
  -> 继续选择另一个 bootstrap follower
  -> 永不回到 1003
```

这解释了全部核心现象：heartbeat 正常、Raft 网络流量很大、没有持续报错、LEO/HW 停止推进、metadata lag 持续增长。

## 3. 故障时间线

| 时间 | 证据 | 含义 |
|------|------|------|
| 11:33:55.233 | 本机高水位更新到 `35678650`，通知 MetadataLoader 读取 `baseOffset=35678649` | 此时复制和发布仍正常 |
| 11:34:01.337 | G1 Young GC，耗时 11.781ms | 不存在长 GC |
| 11:34:01.35 左右 | 大量分区集中执行 ISR shrink | 当时存在调度或处理延迟，但不是长 GC；不是最终根因 |
| 11:34:01.382 | 一条来自 1003 的有效 FetchResponse 被标记为 `no longer needed` | 原请求已超时或状态已清理，是故障入口的旁证，不是根因 |
| 11:34 之后 | `fetch-records-rate` 降为 0，HW 停在 `35678650` | 不再成功追加 Raft record |
| 11:34 之后 | LEO 停在 `35678651`，epoch 始终为 643 | 无新日志追加，无 leader 换届 |
| 11:56:09 | 完整 jstack 显示 Raft、metadata loader 均在正常等待 | 排除业务锁死锁和 publisher 阻塞 |
| 15:15 至 15:17 | 抓包发现 `raft-client-68` 向三个非 leader controller 高速 Fetch | 找到真正的请求路径 |
| 15:38 | 仅在故障 broker 68 开启 TRACE，客户端日志明确显示连续请求三个 follower，全部返回 `errorCode=6`，并携带 `leaderId=1003` | 完整确认 KAFKA-19354 |

## 4. 关键排查方向与证据

### 4.1 排除全局 metadata 停写

其他 broker 的 `last-applied-record-lag-ms` 始终约为 1 秒，而只有 broker 68 的 lag 线性增长。

结论：controller leader 仍在推进 metadata，问题是 broker 68 的本地复制链路。

### 4.2 排除 MetadataLoader 和 publisher 阻塞

完整 jstack 中：

```text
kafka-68-metadata-loader-event-handler
  at org.apache.kafka.queue.KafkaEventQueue$EventHandler.handleEvents
  at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await
```

metadata loader 正在等待空事件队列，没有进入 `ReplicaManager.applyDelta` 或其他 publisher，也没有持有业务锁。完整堆栈没有 `BLOCKED` 线程或 Java deadlock。

结论：不是 metadata 已复制但 publisher 无法应用。

### 4.3 排除长 GC

故障窗口附近的 GC 均约为 10ms：

```text
11:33:45.108  GC ... 10.152ms
11:34:01.348  GC ... 11.781ms
11:34:09.731  GC ... 11.221ms
11:34:19.955  GC ... 11.203ms
```

后续 GC 持续正常执行。

结论：长 GC 不是原因。11:34:01 大量 ISR shrink 集中出现说明当时可能存在短暂调度或处理停顿，但它不能解释后续数小时的复制停滞，应与最终 Raft 选路缺陷区分。

### 4.4 区分 heartbeat 与 Raft Fetch

到 1003 的 TCP 连接保持 ESTAB，`Recv-Q=0`、`Send-Q=0`，且 `lastsnd/lastrcv` 很新，表面上看网络完全正常。

抓包后确认该连接实际是 heartbeat：

```text
apiKey=0x003f = 63 = BROKER_HEARTBEAT
clientId="68"
```

请求约每 2 秒一次。响应中的 `r-48-fetcher-0` 是 controller incarnation ID，不是 Raft client ID。

结论：heartbeat 与 Raft metadata replication 是独立通道。broker 在线不能证明 Raft 复制正常，到 leader 的 TCP 连接活跃也不能证明 Fetch 正常。

### 4.5 排除 snapshot 下载

`__cluster_metadata-0` 目录中：

- 没有 `.checkpoint.part` 文件。
- 最新 checkpoint 是 10:41 生成的 `00000000000035672301-0000000643.checkpoint`。
- 活跃日志段 mtime 停在 11:33。

结论：没有持续下载或残留部分 snapshot。

### 4.6 识别 Raft 网络层仍在高速收发

`raft-channel-metrics` 显示：

- `request_total` 和 `response_total` 持续稳定增长。
- `outgoing_byte_rate` 约为 92 KB/s。
- `incoming_byte_rate` 约为 63 KB/s。
- `connection_creation_total` 不增长，没有反复重建连接。

与此同时：

```text
current_epoch = 643
log_end_epoch = 643
log_end_offset = 35678651
high_watermark = 35678650
fetch_records_rate = 0
```

结论：不是“完全不发请求”，而是请求和响应没有形成有效复制。此时应重点检查请求目的节点、响应 errorCode 和 leader 信息。

### 4.7 抓包发现非 leader Fetch 循环

到 `controller01/controller02/controller03` 的连接均存在大量 `raft-client-68` 请求：

```text
apiKey=0x0001 = FETCH
clientId="raft-client-68"
fetchOffset=0x022069bb = 35678651
currentLeaderEpoch=0x0283 = 643
```

响应中可解析出：

```text
errorCode=0x0006 = NOT_LEADER_OR_FOLLOWER
leaderId=0x03eb = 1003
leaderEpoch=0x0283 = 643
```

三个非 leader 节点都告诉 broker 68 真正的 leader 是 1003，但 broker 68 仍持续轮询这些节点。

## 5. 决定性 TRACE 证据

15:38，**仅在故障 broker 68 上开启 TRACE**，由 `[RaftManager id=68]` 输出的客户端日志完整展示了请求循环。以下发送和接收记录全部来自故障 broker，不是各 controller 的服务端日志；controller01/02/03 只是日志中的请求目的节点或响应来源。

```text
[RaftManager id=68] 15:38:01.495 -> controller01, correlationId=28740860
[RaftManager id=68] 15:38:01.496 <- controller01, errorCode=6, leaderId=1003, leaderEpoch=643

[RaftManager id=68] 15:38:01.496 -> controller02, correlationId=28740861
[RaftManager id=68] 15:38:01.498 <- controller02, errorCode=6, leaderId=1003, leaderEpoch=643

[RaftManager id=68] 15:38:01.498 -> controller01, correlationId=28740862
[RaftManager id=68] 15:38:01.499 <- controller01, errorCode=6, leaderId=1003, leaderEpoch=643

[RaftManager id=68] 15:38:01.499 -> controller03, correlationId=28740863
[RaftManager id=68] 15:38:01.502 <- controller03, errorCode=6, leaderId=1003, leaderEpoch=643
```

每次请求均为：

```text
destination id=-2/-3/-4
fetchOffset=35678651
currentLeaderEpoch=643
lastFetchedEpoch=643
maxWaitMs=500
```

`destination id=-2/-3/-4` 是 bootstrap 节点 ID，进一步证明请求来自 observer 的 bootstrap 选路，而非正常向 leader `1003` 的 Fetch。

这是最终证据链的闭环：

```text
持续向非 leader Fetch
  -> 全部返回 NOT_LEADER_OR_FOLLOWER
  -> 响应明确给出 leader=1003
  -> 客户端仍继续轮询 bootstrap
  -> 无 record 追加
  -> LEO/HW 不动
  -> MetadataLoader 无事件
  -> last-applied-record-lag-ms 持续增长
```

## 6. 代码级原因

### 6.1 observer 超时后只走 bootstrap

当前 3.9.2 代码：

```java
private long pollFollowerAsObserver(FollowerState state, long currentTimeMs) {
    if (state.hasFetchTimeoutExpired(currentTimeMs)) {
        return maybeSendFetchToAnyBootstrap(currentTimeMs);
    } else {
        return maybeSendFetchToBestNode(state, currentTimeMs);
    }
}
```

fetch timeout 一旦到期，后续选路固定在 bootstrap 节点中，而不是重新评估真正的 leader。

### 6.2 非 leader 响应不能使 observer 回到 leader

`NOT_LEADER_OR_FOLLOWER` 在 `maybeHandleCommonResponse` 中作为正常控制流处理：

```java
} else if (epoch > quorum.epoch()
    || error == Errors.FENCED_LEADER_EPOCH
    || error == Errors.NOT_LEADER_OR_FOLLOWER) {
    maybeTransition(leaderId, epoch, leaderEndpoints, currentTimeMs);
    return Optional.of(true);
}
```

本例中响应 epoch 与本机相同，且本机已经知道 leader 1003，所以不会触发 leader 状态变化。该分支随后将响应标记为已处理，清理请求状态，observer 随即选择下一个 bootstrap。

该处理路径不会产生持续的 `Unexpected error` 日志，因此仅看 INFO/WARN/ERROR 很容易漏掉。

### 6.3 初始 `Ignoring response` 的含义

11:34:01 来自 1003 的响应本身是有效的：

```text
errorCode=0
highWatermark=35678651
leaderId=1003
leaderEpoch=643
```

但响应到达时，`RequestManager` 已不再跟踪其 `(source, correlationId)`，所以被丢弃。常见原因是请求已超时、连接状态被重置或响应迟到。

这条日志不是最终根因，而是 observer 进入 bootstrap 状态前后的初始异常信号。最终持续故障由后续选路缺陷维持。

## 7. 社区问题与修复

### KAFKA-19354

- JIRA：[KAFKA-19354](https://issues.apache.org/jira/browse/KAFKA-19354)
- 修复 PR：[apache/kafka#19854](https://github.com/apache/kafka/pull/19854)
- 修复提交：[7339d65b27fd5c1aa5ba239faf193ee0ff55f06d](https://github.com/apache/kafka/commit/7339d65b27fd5c1aa5ba239faf193ee0ff55f06d)
- JIRA 标记影响版本：3.9.1、4.0.0
- 修复版本：4.2.0

社区描述与本例一致：observer 在请求失败或超时后重新 bootstrap，若选中 follower，就会持续收到 `NOT_LEADER_OR_FOLLOWER`，停止 metadata replication，重启后可能恢复。

核心修复：

```java
private long pollFollowerAsObserver(FollowerState state, long currentTimeMs) {
    return maybeSendFetchToBestNode(state, currentTimeMs);
}
```

删除“fetch timeout 后只从 bootstrap 拉取”的固定分支，恢复根据请求超时、退避和 leader 状态选择最佳节点。

官方提交同时修改：

```text
raft/src/main/java/org/apache/kafka/raft/FollowerState.java
raft/src/main/java/org/apache/kafka/raft/KafkaRaftClient.java
raft/src/test/java/org/apache/kafka/raft/KafkaRaftClientTest.java
raft/src/test/java/org/apache/kafka/raft/RequestManagerTest.java
```

建议 backport 完整提交及测试，而不是只修改一行代码。

## 8. 恢复与后续行动

### 临时恢复

重启故障 broker 可重新初始化 Raft 请求和选路状态，通常能够恢复复制，但在未修复代码前仍可能复发。

仅断开部分连接可能暂时改变选路，但仍存在再次选中 follower 的竞态，不建议作为长期解决方案。

### 正式修复

1. 将 KAFKA-19354 修复 backport 到当前 3.9.2 内部分支。
2. 保留官方单元测试，覆盖“observer 从 bootstrap 响应发现 leader 后重新向 leader Fetch”。
3. 增加网络延迟或请求超时故障注入，验证 observer 能在 bootstrap follower 与真正 leader 之间恢复。
4. 灰度验证后滚动发布，重点观察 HW、LEO 和 metadata applied offset 是否恢复同步。

### 监控建议

继续监控：

```text
kafka.server:type=broker-metadata-metrics,name=last-applied-record-lag-ms
kafka.server:type=broker-metadata-metrics,name=last-applied-record-offset
kafka.server:type=raft-metrics,name=high-watermark
kafka.server:type=raft-metrics,name=log-end-offset
kafka.server:type=raft-metrics,name=current-epoch
kafka.server:type=raft-metrics,name=fetch-records-rate
kafka.server:type=raft-channel-metrics,name=request-total
kafka.server:type=raft-channel-metrics,name=response-total
```

可增加诊断保护：

- 对 observer 的 `NOT_LEADER_OR_FOLLOWER` 响应增加限速日志或计数，记录 source、响应中的 leaderId/epoch。
- 对连续 N 次向非 leader 发送 Fetch 的状态增加告警。
- 告警应同时比较 Raft 流量与 record 推进，避免将“网络有流量”误判为复制健康。

## 9. 排查经验总结

1. **broker 在线不等于 metadata 正常**：heartbeat 与 Raft 复制是独立通道。
2. **TCP ESTAB 不等于业务健康**：需要确认连接上的协议类型、clientId 和请求目的节点。
3. **Raft 有流量不等于复制正常**：高速 `NOT_LEADER_OR_FOLLOWER` 循环同样会产生大量双向流量。
4. **HW 与 LEO 要同时看**：本例 LEO=`35678651`、HW=`35678650`，能帮助排除 publisher 阻塞并聚焦有效 Fetch 响应。
5. **无 ERROR 不等于无异常**：本例错误响应属于正常控制流，INFO/WARN 中没有持续报错。
6. **中途假设必须随证据修正**：长 GC、publisher 阻塞、无请求发送、snapshot 下载都曾是候选方向，最终分别被 GC 日志、jstack、网络指标、文件系统和抓包排除。

