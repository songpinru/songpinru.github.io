---
title: "Client Api Version Cap Compatibility Analysis"
---

# 客户端 API 版本封顶兼容性问题排查

## 1. 问题背景

当前分支实现了客户端 API 版本封顶功能。启用后，broker 期望只对客户端 listener 广告并接受以下最高版本：

```text
METADATA:6
FETCH:8
LIST_OFFSETS:3
OFFSET_COMMIT:5
OFFSET_FETCH:4
OFFSET_FOR_LEADER_EPOCH:1
```

设计目标是让新客户端退回到不使用 TopicId 和 leaderEpoch fencing 的旧协议行为，同时保持 broker 内部通信使用完整高版本。

实际兼容性测试发现，以下非 Java 客户端的较新版本无法正常读写：

- librdkafka 1.9.2 之后的部分版本；
- kafka-go；
- 较新的 kafka-python 或基于它的发行版本。

主要现象包括：

- 消费时报 `UnsupportedApiVersion` 或 `UNSUPPORTED_VERSION`；
- 消费或生产初始化时报 `UnknownPartition`、`UnknownTopicOrPartition` 或 `topic partition not found`；
- 元数据、offset 查询、Fetch 或 offset 提交持续失败或重试。

本文记录服务端与客户端源码排查过程、问题原理、验证方法和修复建议。

## 2. 结论摘要

根因是 broker 在 SASL 认证前后提供了互相矛盾的 API 能力：

```text
SASL 认证前 ApiVersions：未应用 cap，广告完整高版本
SASL 认证后业务请求：KafkaApis 应用 cap，拒绝高版本
```

librdkafka、kafka-go 和较新的 kafka-python 都可能在 SASL 认证前查询 ApiVersions，并把该响应作为后续请求的能力表。它们通常不会在认证完成后再次查询，因此会根据 broker 认证前广告的完整版本发送高版本 Metadata、Fetch 和 offset 请求。

这些请求随后在 `KafkaApis.handle` 中被版本封顶兜底逻辑拒绝。客户端看到的两类错误具有同一根因：

- `UnsupportedApiVersion`：Fetch、ListOffsets、OffsetFetch、OffsetCommit 等响应可以直接携带错误码 35；
- `UnknownPartition`：Metadata 查询全部 topic 时无法在响应中表达顶层版本错误，broker 返回空 metadata，客户端因此认为 topic 或 partition 不存在。

因此，问题并不是这些客户端不会根据正确的 ApiVersions 响应降级，而是它们实际收到的第一份、也是最终使用的 ApiVersions 响应没有被封顶。

## 3. 排查过程

### 3.1 确认服务端版本封顶位置

认证后的正常 ApiVersions 请求在 `KafkaApis` 中处理：

`core/src/main/scala/kafka/server/KafkaApis.scala:2078-2102`

其中 `2092-2098` 行先构造响应，再应用版本封顶：

```scala
val response = apiVersionManager.apiVersionResponse(requestThrottleMs, request.header.apiVersion() < 4)
if (ClientApiVersionCap.shouldCap(
    config.clientApiVersionCapEnable,
    request.context.listenerName,
    config.interBrokerListenerName)) {
  ClientApiVersionCap.capResponse(response, config.clientApiVersionCapOverrides)
}
```

超过 cap 的业务请求会在统一分发入口被拒绝：

`core/src/main/scala/kafka/server/KafkaApis.scala:190-197`

```scala
if (ClientApiVersionCap.shouldCap(...) &&
    ClientApiVersionCap.isVersionRejected(...)) {
  throw new UnsupportedVersionException(...)
}
```

这说明认证后的上报层和请求拒绝层都实现了 cap。

### 3.2 确认 SASL 认证前存在独立 ApiVersions 路径

Kafka 允许客户端在 SASL handshake 前发送 ApiVersionsRequest。该请求不进入 `KafkaApis`，而是由 `SaslServerAuthenticator` 直接处理：

`clients/src/main/java/org/apache/kafka/common/security/authenticator/SaslServerAuthenticator.java:612-624`

```java
sendKafkaResponse(context, apiVersionSupplier.apply(apiVersionsRequest.version()));
```

该 supplier 在 SocketServer 中构造：

`core/src/main/scala/kafka/network/SocketServer.scala:966-977`

```scala
version => apiVersionManager.apiVersionResponse(
  throttleTimeMs = 0,
  version < 4
)
```

这里直接返回 `ApiVersionManager` 的完整响应，没有调用 `ClientApiVersionCap.capResponse`。

由此确认服务端存在两条不同路径：

| 请求阶段 | 处理组件 | 是否应用 cap |
|---------|---------|-------------|
| SASL 认证前 | `SaslServerAuthenticator` | 否 |
| SASL 认证后 | `KafkaApis` | 是 |

### 3.3 确认非 Java 客户端的协商时序

三类客户端的典型连接顺序都是：

```text
建立 TCP/TLS 连接
  -> 发送 ApiVersionsRequest
  -> 缓存 broker API 能力表
  -> 执行 SaslHandshake 和认证
  -> 根据缓存能力选择业务 API 版本
```

认证成功后通常不会再次查询 ApiVersions。

这与服务端缺陷组合后形成确定的失败链路：

```text
认证前收到未 cap 的完整能力表
  -> 客户端选择高版本 Metadata/Fetch/offset API
  -> 认证后请求进入 KafkaApis
  -> 超过 cap，被返回 UNSUPPORTED_VERSION
```

### 3.4 确认错误响应的协议表现

`KafkaApis` 捕获版本异常后，通过以下路径构造每种 API 的标准错误响应：

`core/src/main/scala/kafka/server/RequestHandlerHelper.scala:77-95`

```scala
val response = requestBody.getErrorResponse(throttleMs, error)
```

异常会映射为：

```text
UNSUPPORTED_VERSION = 35
```

定义位于：

`clients/src/main/java/org/apache/kafka/common/protocol/Errors.java:120-121`

Fetch、ListOffsets、OffsetFetch、OffsetCommit 和 OffsetsForLeaderEpoch 都可以在顶层、group 或 partition 结果中携带错误码，因此客户端通常直接报告版本不支持。

Metadata 的行为不同，具体见第 6 节。

## 4. 完整失败链路

以 SASL 客户端为例：

```text
1. 客户端连接 SASL listener
2. 客户端在认证前发送 ApiVersionsRequest
3. SaslServerAuthenticator 调用未封顶的 apiVersionSupplier
4. Broker 广告 Metadata/Fetch 等完整高版本
5. 客户端缓存该能力表
6. 客户端完成 SASL 认证
7. 客户端按缓存选择高版本业务请求
8. 请求进入 KafkaApis.handle
9. KafkaApis 发现请求版本超过 cap
10. Broker 返回 UNSUPPORTED_VERSION
11. 客户端失败、刷新 metadata、重连或重复发送相同版本
```

这里违反了 Kafka ApiVersions 协商的基本契约：

```text
Broker 广告支持版本 X，但随后拒绝同一连接上的版本 X。
```

客户端没有义务在收到 `UNSUPPORTED_VERSION` 后自动逐级降低版本。常见实现只会断线重连或刷新元数据，而新连接仍会收到相同的未封顶能力表，因此可能无限重复失败。

## 5. 各客户端行为分析

### 5.1 librdkafka

librdkafka 每个 API 的正常版本选择逻辑是：

```text
min(客户端实现的最高版本, broker 广告的最高版本)
```

ApiVersions 在 SASL 认证前完成，响应被保存为该 broker 连接的能力表，认证后不会再次协商。

如果正确收到 cap，librdkafka 会选择不超过 cap 的版本。当前问题是它收到的是未封顶版本。

以 librdkafka 1.9.2 为例，可能出现：

| API | 客户端可选择版本 | 服务端 cap | 结果 |
|-----|-----------------|-----------|------|
| Metadata | 4 | 6 | 不超过 cap |
| Fetch | 11 | 8 | 被拒绝 |
| OffsetCommit | 7 | 5 | 被拒绝 |
| OffsetFetch | 7 | 4 | 被拒绝 |

这可以解释某些版本中生产仍能工作，但消费、查询提交位点或提交位点失败。

更高版本 librdkafka 支持 Metadata v9/v12、ListOffsets v5/v7、Fetch v16 等，Metadata 也可能被拒绝，因此生产可能在获取 topic/partition leader 前失败。

相关实现参考：

- <https://github.com/confluentinc/librdkafka/blob/v1.9.2/src/rdkafka_broker.c>
- <https://github.com/confluentinc/librdkafka/blob/master/src/rdkafka_sasl.c>
- <https://github.com/confluentinc/librdkafka/blob/master/CONFIGURATION.md>

### 5.2 kafka-go

kafka-go 同时存在现代 `Client/Transport` 和旧 `Conn/Reader` 两套路径，但两者都会在 SASL 认证前查询 ApiVersions，认证后不重新查询。

现代 `Client/Transport` 收到完整版本表后可能选择：

```text
Metadata v8
Fetch v11
ListOffsets v5
OffsetCommit v7
OffsetFetch v5
```

旧 `Conn/Reader` 路径可能选择：

```text
Metadata v6
Fetch v10
ListOffsets v1
OffsetCommit v2
OffsetFetch v1
```

因此旧 Reader 即使 Metadata 没有超过 cap，也可能因为 Fetch v10 超过 cap 8 而无法消费。

如果现代路径正确收到 cap，它会选择 Metadata v6、Fetch v8、ListOffsets v3、OffsetCommit v5 和 OffsetFetch v4，不会主动越过 cap。

相关实现参考：

- <https://github.com/segmentio/kafka-go/blob/v0.4.51/transport.go>
- <https://github.com/segmentio/kafka-go/blob/v0.4.51/protocol/conn.go>
- <https://github.com/segmentio/kafka-go/blob/v0.4.51/conn.go>
- <https://github.com/segmentio/kafka-go/blob/v0.4.51/dialer.go>

### 5.3 kafka-python

官方 kafka-python 没有 `2.4.x` release。现场所称“2.4 之后”可能是 Kafka broker 版本、vendor fork 或其他 Python 客户端发行版本，应先确认实际安装包和 `kafka.__version__`。

较新 kafka-python 自动协商模式也会在 SASL 认证前发送 ApiVersions，并缓存得到的能力表。收到正确 cap 时，它会逐 API 选择双方共同支持的最高版本；收到当前未封顶响应时，则可能发送高版本业务请求。

kafka-python 还有一个独立风险：2.x 显式设置以下配置时会跳过真实 ApiVersions 请求，直接使用客户端内置的静态 broker 能力表：

```python
api_version=(2, 4)
```

这种配置即使修复服务端认证前响应，也可能继续绕过自定义 cap。自动协商应使用：

```python
api_version=None
```

较老版本中的字符串 `'auto'` 通常会转换为 `None`，但不建议依赖这一兼容行为。

相关实现参考：

- <https://github.com/dpkp/kafka-python/blob/2.2.4/kafka/conn.py>
- <https://github.com/dpkp/kafka-python/blob/2.2.4/kafka/client_async.py>
- <https://github.com/dpkp/kafka-python/blob/2.2.4/kafka/protocol/broker_api_versions.py>
- <https://github.com/dpkp/kafka-python/blob/3.0.0/kafka/net/connection.py>

### 5.4 与 Kafka Java 客户端的差异

Kafka Java 客户端在认证前的 ApiVersions 主要用于确定 SASL Handshake 和 SaslAuthenticate 版本。认证完成后，NetworkClient 仍会执行正常的 broker API 版本发现。

因此 Java 客户端可能在认证后获得 `KafkaApis` 返回的封顶版本表，而非 Java 客户端直接使用认证前版本表。这使当前问题在跨语言测试中更明显，也说明服务端实现不能依赖 Java 客户端特有的连接状态机。

## 6. 为什么分别表现为 UnsupportedApiVersion 和 UnknownPartition

### 6.1 UnsupportedApiVersion

客户端收到未封顶能力表后可能发送：

```text
Fetch v10/v11/v16        > cap 8
ListOffsets v4/v5/v7     > cap 3
OffsetCommit v6/v7/v9    > cap 5
OffsetFetch v5/v7/v9     > cap 4
OffsetForLeaderEpoch v2  > cap 1
```

这些请求被 `KafkaApis.handle` 拒绝后，其标准错误响应可以携带错误码 35。

例如 Fetch 的错误响应会设置顶层错误码，并在低于 v13 时为每个请求分区设置相同错误：

`clients/src/main/java/org/apache/kafka/common/requests/FetchRequest.java:343-369`

ListOffsets、OffsetCommit、OffsetFetch 和 OffsetsForLeaderEpoch 也会把错误映射到请求中的 partition 或 group。因此客户端会直接报告：

```text
UnsupportedApiVersion
UnsupportedVersion
UNSUPPORTED_VERSION
```

### 6.2 UnknownPartition

`UnknownPartition` 通常不是最初的 broker 错误，而是 Metadata 请求被拒绝后的二级症状。

Metadata 协议没有顶层 error code。其错误响应由以下代码构造：

`clients/src/main/java/org/apache/kafka/common/requests/MetadataRequest.java:142-159`

当请求明确列出 topic 时，broker 可以为每个 topic 返回错误码 35 和空分区列表。

当客户端查询全部 topic 时，`MetadataRequest.data.topics()` 为 `null`。错误响应无法添加任何 topic 错误项，最终近似为：

```text
brokers = []
topics = []
```

客户端无法知道真实原因是 Metadata 版本过高，只能观察到：

- 没有目标 topic；
- 没有对应 partition；
- 没有 leader 或 broker 路由。

随后客户端或上层库会产生：

```text
UnknownPartition
UnknownTopicOrPartition
topic partition not found
```

典型因果链如下：

```text
高版本 Metadata 被拒绝
  -> 返回空 metadata 或 topic 级错误
  -> 客户端 metadata cache 没有 partition/leader
  -> 业务请求无法路由
  -> UnknownPartition
```

因此排查现场问题时，不能只检查最后一条 `UnknownPartition` 日志，还需要向前查找 Metadata 请求版本和服务端的 `UnsupportedVersionException`。

## 7. 生产与消费的影响差异

当前默认 cap 不包含 Produce，因此 Produce RPC 本身不会被这组规则直接拒绝。

但生产者必须先通过 Metadata 找到 topic、partition 和 leader：

- 如果客户端 Metadata 版本不超过 6，生产可能继续正常；
- 如果客户端根据未封顶响应选择 Metadata v7 以上，Metadata 会被拒绝，生产者可能在发送 Produce 前失败；
- 已缓存元数据可能让生产短暂成功，但 leader 变化或 metadata refresh 后仍会失败。

消费路径受影响更直接，因为默认 cap 覆盖了 Fetch、ListOffsets、OffsetFetch、OffsetCommit 和 OffsetsForLeaderEpoch。即使 Metadata 正常，任何一个 offset 初始化或 Fetch 环节超过 cap 都会导致消费者无法工作。

这解释了以下常见现象：

```text
旧版本客户端可以生产，但无法消费
较新客户端生产和消费都失败
同一客户端已有连接短暂正常，重连后失败
```

## 8. 现场验证方法

### 8.1 抓取第一条 ApiVersionsResponse

关键不是认证后的 ApiVersions 响应，而是同一物理连接在 SASL 认证前收到的第一条响应。

需要确认其中是否仍广告：

```text
Metadata > 6
Fetch > 8
ListOffsets > 3
OffsetCommit > 5
OffsetFetch > 4
OffsetForLeaderEpoch > 1
```

如果认证前广告高版本，随后相同连接上的业务请求收到错误码 35，即可确认根因。

### 8.2 对比 SASL 与非 SASL listener

分别连接：

- `SASL_PLAINTEXT` 或 `SASL_SSL` listener；
- `PLAINTEXT` 或完成 TLS 后不走 Kafka SASL 状态机的 listener。

预期当前实现中：

- SASL listener 认证前响应未 cap；
- 非 SASL listener 的 ApiVersions 请求进入 `KafkaApis`，响应已 cap。

### 8.3 服务端日志与抓包字段

建议记录或抓取：

- connectionId；
- listenerName；
- ApiKey；
- ApiVersion；
- correlationId；
- ApiVersionsResponse 中每个目标 API 的 min/max version；
- 返回错误码；
- SASL 状态和认证完成时间。

重点关联以下顺序：

```text
ApiVersions -> SaslHandshake -> SaslAuthenticate -> Metadata/Fetch
```

### 8.4 客户端侧检查

librdkafka 建议开启：

```text
debug=broker,protocol,security,feature
```

重点检查连接状态、ApiVersions 查询顺序和最终选择的请求版本。

kafka-go 需要确认使用的是：

- `Reader/Conn`；或
- `Client/Transport`。

两者可能选择不同的高版本，但都会受认证前响应影响。

kafka-python 需要确认：

```python
kafka.__version__
client.config['api_version']
client.get_api_versions()
```

并排除显式 `api_version=(...)` 绕过真实协商的情况。

### 8.5 其他需要排除的变量

还应确认：

- 所有 broker 的 cap 开关和 overrides 完全一致；
- bootstrap 地址与 Metadata 返回地址使用同一类 listener；
- 没有代理把 ApiVersions 与业务请求转发到不同后端；
- 客户端没有复用来自其他集群或 listener 的全局能力表；
- 运行包与预期版本一致，不是 vendor fork。

## 9. 修复建议

### 9.1 必须修复认证前 ApiVersions

给 `SaslServerAuthenticator` 注入的 supplier 必须应用与 `KafkaApis` 完全相同的 cap：

```scala
version => {
  val response = apiVersionManager.apiVersionResponse(
    throttleTimeMs = 0,
    version < 4
  )

  if (ClientApiVersionCap.shouldCap(
      config.clientApiVersionCapEnable,
      listenerName,
      config.interBrokerListenerName)) {
    ClientApiVersionCap.capResponse(
      response,
      config.clientApiVersionCapOverrides
    )
  }

  response
}
```

该示例只说明关键逻辑。正式实现还应同时处理 control-plane listener，避免内部 controller 通信被错误封顶。

### 9.2 统一所有 ApiVersions 响应入口

更稳妥的做法不是在两个位置复制逻辑，而是抽取统一响应构造方法，让以下路径全部调用同一实现：

- SASL 认证前 `SaslServerAuthenticator`；
- 认证后的 `KafkaApis.handleApiVersionsRequest`；
- 其他可能直接使用 `ApiVersionManager.apiVersionResponse` 的客户端 listener 路径。

统一入口应接收：

- request API version；
- listener name；
- listener 角色；
- throttle time；
- 是否启用 cap；
- cap overrides。

这样可以避免认证状态、listener 或后续重构再次产生两份不同能力表。

### 9.3 兜底拒绝逻辑的定位

超 cap 拒绝可以保留，用于阻止未协商或恶意客户端硬发高版本，但它只能是最后保护，不能替代正确广告。

正确关系应为：

```text
ApiVersions 广告 <= cap
客户端正常请求 <= cap
只有绕过协商的请求才收到 UNSUPPORTED_VERSION
```

### 9.4 补充集成测试

至少需要覆盖：

1. SASL 认证前 ApiVersions 返回 cap 后版本；
2. SASL 认证后 ApiVersions 与认证前结果一致；
3. 模拟 librdkafka：认证前协商后不再查询，直接发送 Fetch v8；
4. 模拟 kafka-go Reader：不能根据认证前响应选到 Fetch v10；
5. 查询全部 topic 的高版本 Metadata 请求不会静默变成空 metadata；
6. SASL 与非 SASL 客户端 listener 返回相同的 cap；
7. inter-broker listener 保持完整版本；
8. ZK control-plane listener 保持完整版本；
9. librdkafka、kafka-go、kafka-python 的容器化兼容性测试；
10. 所有 broker 配置一致和不一致时的行为验证。

现有 `SaslApiVersionsRequestTest.testApiVersionsRequestBeforeSaslHandshakeRequest` 已证明认证前路径存在，但测试没有开启版本 cap，也没有断言目标 API 的最大版本，需要扩展。

## 10. 最终结论

当前客户端 API 版本封顶功能只覆盖了认证后的 ApiVersions 请求，没有覆盖 SASL 认证前的版本协商路径。

非 Java 客户端在认证前缓存了未封顶能力表，认证后按该表发送高版本请求，却被 broker 的兜底逻辑拒绝。这是 librdkafka、kafka-go 和较新 kafka-python 无法正常读写的主要原因。

`UnsupportedApiVersion` 是业务 API 被直接拒绝的原始错误；`UnknownPartition` 通常是高版本 Metadata 被拒绝后返回空或不完整 metadata 导致的二级错误。

修复必须保证同一个 listener 上所有认证状态下的 ApiVersions 响应完全一致。只有先修正能力广告，后置版本拒绝才能作为有效兜底，而不会破坏正常客户端的协议协商。
