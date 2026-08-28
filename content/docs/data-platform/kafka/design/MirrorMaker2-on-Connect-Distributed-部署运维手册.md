---
title: "MirrorMaker2 On Connect Distributed 部署运维手册"
---


> 文档版本：v3.7
>
> 更新时间：2026-08-19
>
> 适用链路：`wbads -> wbbz`
>
> 适用模式：Kafka Connect Distributed 直接运行 MM2 Connector
>
> 核对基准：Apache Kafka 3.x 与 4.0 的官方文档和源码
>
> v3.0 变更：重组章节结构，Connector 配置改为“公共片段 + 专属字段”分层提交，消除重复的集群地址与凭据。
>
> v3.1 变更：澄清 Connect 内部 topic 可自动创建，补齐 Heartbeat Connector 必需的 `source.cluster.alias`，并修正 topic 配置同步和专用模式反向心跳的边界条件。
>
> v3.2 变更：补充 Worker 与 Connector 的配置边界，说明 Source Task producer、converter 和权限的来源。
>
> v3.3 变更：移除外部 JSON 处理依赖和配置片段合并流程，改为四个可直接提交的完整 Connector JSON，并补充凭据重复的原因。
>
> v3.4 变更：澄清独立管理集群下 Source Task producer 的默认目标与 `producer.override.*` 覆盖机制。
>
> v3.5 变更：明确 Heartbeat Connector 为可选监控组件，调整默认提交脚本只提交 Source 和 Checkpoint。
>
> v3.6 变更：澄清单向复制的标准心跳拓扑，以及反向 Heartbeat 缺少 `producer.override.*` 时的错误行为。
>
> v3.7 变更：恢复双方向 Heartbeat 为默认提交项，保留可选说明但不再通过环境变量控制。

## 1. 架构总览

将原 `connect-mirror-maker.sh` 专用模式转换为普通 Kafka Connect Distributed 部署，转换后具备：

- 通过 Connect REST API 动态修改同步 topic 和 group，无需重启 Worker。
- 多个 Worker 自动分配 Connector 和 Task。
- 保持现有单向复制、topic 不加前缀、consumer group offset 自动同步等语义。
- 可选保留专用模式的正向和反向心跳行为。

拓扑：

```text
                         Kafka Connect Distributed
                        （管理 Kafka 使用 wbbz）
                                   |
              +--------------------+--------------------+
              |                    |                    |
     MirrorSourceConnector  MirrorCheckpointConnector  正向 Heartbeat
          wbads -> wbbz          wbads -> wbbz         wbads -> wbbz
              |                    |                    |
              +--------------------+--------------------+
                                   |
                                  wbbz

     反向 MirrorHeartbeatConnector：wbbz -> wbads
     通过 producer.override.* 将 heartbeats 写入 wbads，
     再由 wbads -> wbbz 的 Source Connector 复制为
     wbbz 上的 wbads.heartbeats。
```

完整监控链路部署四个 Connector。Source Connector 复制业务数据；Checkpoint 用于 consumer group offset 同步；两个方向的心跳共同提供目标集群可写性和端到端复制路径监控：

| Connector 名称 | 类 | 作用 |
|---|---|---|
| `mm2-wbads-to-wbbz-source` | `MirrorSourceConnector` | 复制业务数据、分区信息和 offset-sync |
| `mm2-wbads-to-wbbz-checkpoint` | `MirrorCheckpointConnector` | 生成 checkpoint，并同步目标 group offset |
| `mm2-wbads-to-wbbz-heartbeat` | `MirrorHeartbeatConnector` | 可选：向 wbbz 写正向心跳 |
| `mm2-wbbz-to-wbads-heartbeat` | `MirrorHeartbeatConnector` | 可选：向 wbads 写反向心跳，观察完整复制路径 |

内部 topic 位置（保持原配置默认行为）：

| topic | 所在集群 | 作用 |
|---|---|---|
| `mm2-offset-syncs.wbbz.internal` | wbads | 源 offset 到目标 offset 的映射 |
| `wbads.checkpoints.internal` | wbbz | consumer group checkpoint |
| `heartbeats` | wbads 和 wbbz | 本地心跳 topic |
| `wbads.heartbeats` | wbbz | 由 wbads 反向心跳复制而来的远端心跳 topic |
| Connect config/offset/status topic | wbbz | Connect 集群管理和 Source Task 进度 |

## 2. 关键语义

本节集中说明配置的实际行为，后续章节不再重复解释。

### 2.1 复制语义

- `topics=.*`、`groups=.*`：复制所有未被默认排除规则排除的 topic 和 group。默认排除：

  ```text
  topic: .*[\-\.]internal, .*\.replica, __.*
  group: console-consumer-.*, connect-.*, __.*
  ```

  如显式配置 `topics.exclude`/`groups.exclude`，必须保留上述默认排除项。上述 topic 默认排除值对应 Kafka 3.x；Kafka 4.0 起默认值改为 `mm2.*\.internal, .*\.replica, __.*`，显式覆盖前先按实际 Kafka 版本确认。

- `IdentityReplicationPolicy`：普通业务 topic 在 wbbz 上与 wbads 同名。例外是 heartbeat：wbads 上的 `heartbeats` 复制到 wbbz 后名为 `wbads.heartbeats`，用于观察端到端路径。当前是严格单向复制；以后若开启 `wbbz -> wbads` 业务复制，必须重新评估 Identity 策略，否则会形成复制循环。
- `sync.topic.configs.enabled=false` 关闭的是对已有目标 topic 的周期性配置同步，不会关闭目标 topic 创建和分区扩容。新建目标 topic 时，MM2 仍会从源 topic 复制非只读、非敏感配置作为初始配置。
- `refresh.topics` 会重新发现源 topic/partition 并重配 Task；源 topic 消失后停止处理，但不会删除目标 topic。从白名单移除 topic 只停止后续复制，不会删除 wbbz 上已有数据。
- `offset-syncs.topic.location` 未设置，使用默认值 `source`，offset-sync topic 位于 wbads。如需改到 wbbz，必须同时在 Source 和 Checkpoint Connector 设置 `"offset-syncs.topic.location": "target"`；这属于行为变更，不是原配置的等价转换。
- `sync.group.offsets.enabled=true` 会将翻译后的 offset 写入 wbbz 的 `__consumer_offsets`。MM2 只同步当前在 wbbz 没有活跃成员的 consumer group；灾备切换前必须确保同一 group 不会同时在 wbads 和 wbbz 消费，避免两个站点分别推进 offset。

### 2.2 Heartbeat 是否必需

Heartbeat 不复制业务数据，也不参与 consumer group offset 同步；它只用于确认复制链路存活。单向同步的最小部署是只提交 `MirrorSourceConnector`。如果不需要 group offset 同步，Checkpoint Connector 也可以不部署。

单向同步不要求“双向业务复制”，但如果要复刻 MM2 专用模式的默认心跳监控，官方设计确实会使用两个方向的心跳 Connector：

- `wbads -> wbbz` Heartbeat 直接写入 wbbz 的 `heartbeats`，验证 Connect 到目标集群的可写性。
- `wbbz -> wbads` Heartbeat 先写入 wbads 的 `heartbeats`，再由正向 `MirrorSourceConnector` 复制到 wbbz 的 `wbads.heartbeats`，验证真实复制路径。

第二类才是更有价值的端到端心跳。只部署正向 Heartbeat 不能证明 MirrorSourceConnector 正在复制数据。

专用模式中即使 `wbbz->wbads.enabled=false`，只要全局心跳开启，`MirrorMakerConfig.clusterPairs()` 仍会包含 `wbbz -> wbads` 反向心跳链路：反向心跳先写入 wbads，再由正向 Source Connector 复制到 wbbz，用于观察完整复制路径。需要注意的是，如果启动专用模式时用 `--clusters wbbz` 限制了目标集群，这个反向 herder 会被过滤掉，不会实际运行。普通 Connect 模式不会自动补齐这一步，因此本文显式部署 `mm2-wbbz-to-wbads-heartbeat`；如果原专用模式本来就没有反向心跳，这属于新增监控能力，不是等价迁移。

反向 Heartbeat 有一个关键前提：普通 Connect 模式下必须配置 `producer.override.*` 指向 wbads。如果只设置 `source.cluster.alias=wbbz` 和 `target.cluster.alias=wbads`，而不覆盖 producer，Source Task producer 会默认写 Worker 的管理集群 wbbz；此时 `target.cluster.*` 只会让 AdminClient 在 wbads 创建一个空的 `heartbeats` topic，随后正向 Source Connector 可能把这个空 topic 复制为 wbbz 上的空 `wbads.heartbeats`。这种配置是错误的。

### 2.3 常见误解

- `group.id` 是 Connect Worker 集群的组，不是 MM2 在 wbads 上消费数据的 consumer group。MirrorSourceTask 使用手工 partition assignment，不能用 `kafka-consumer-groups --group <Connect group.id>` 监控复制 lag；lag 的监控方式见第 8 节。
- 删除 Connector 不会自动删除其 Source offset，不要把“删除 Connector”等同于“清空 offset”，重跑的正确姿势见 7.3。
- Connect 管理 Kafka 不强制等于下游 Kafka，可以使用独立管理集群（见 9.1），但必须用 `producer.override.*` 指定 SourceRecord 的目标集群。

## 3. 上线前检查

### 3.1 IdentityReplicationPolicy 冲突

普通业务 topic 在 wbads 和 wbbz 上同名，上线前必须确认：

- wbbz 上的同名 topic 不是另一条独立生产链路的写入目标。
- 不存在另一套 `wbbz -> wbads` 业务复制。
- 目标同名 topic 的 partition 数不大于源 topic；Kafka 不支持缩减 partition。
- 下游应用能够接受镜像数据直接进入现有同名 topic。

### 3.2 消息大小

当前参数：wbads 源端 consumer `max.partition.fetch.bytes=8388608`，wbbz 端 Connect producer `max.request.size=11457280`。还应确认：

- wbads broker 允许 consumer 拉取对应大小的 record batch。
- wbbz broker 的 `message.max.bytes` 或 topic `max.message.bytes` 足够大。
- wbbz follower 的复制 fetch 限制足够大。

否则 Task 可能持续重试或失败。

### 3.3 权限

当前账号为 admin，通常具备完整权限。如果后续改为最小权限账号，至少需要：

| 集群 | 权限 |
|---|---|
| wbads | 读取和描述源业务 topic；列出/描述 consumer group；创建、写入和读取 `mm2-offset-syncs.wbbz.internal`；创建和写入 `heartbeats` |
| wbbz | Connect Worker group 权限；读写 config/offset/status topic；创建、写入和描述目标业务 topic；扩分区；创建/读写 checkpoint 和 heartbeat topic；在启用 group offset 同步时 ALTER consumer group offset |

`sync.topic.configs.enabled=false` 降低的是对已有目标 topic 的周期性配置变更需求；`sync.topic.acls.enabled=false` 关闭 ACL 同步。新建目标 topic 仍需要源端 DescribeConfigs、目标端 CreateTopics 等基础权限。

### 3.4 REST 安全

Broker 的 SASL 配置不会保护 Connect REST API。当前使用明文 HTTP（`listeners=http://10.52.139.55:18088`），必须通过防火墙、反向代理或专用管理网络限制访问。任何能访问 REST API 的主体都可能修改、停止或删除 Connector。

## 4. 部署 Connect Worker

### 4.1 Worker 配置与 Connector 配置的边界

Kafka Connect 不能完全类比成 Kubernetes：Worker 不只是调度 Connector，还承担 Source Task 的运行时。`MirrorSourceTask.poll()` 返回的是 Connect 内部的 `SourceRecord`；真正把记录序列化后发送到目标 Kafka、发送成功后提交 Source offset 的，是 Worker 托管的执行框架和 producer。

因此 Worker 配置里有三类内容：

| 配置 | 归属 | 作用 |
|---|---|---|
| `bootstrap.servers`、`group.id`、config/offset/status topic | Connect 集群自身 | Worker 组协调、Connector 配置存储、Source offset 存储、状态存储 |
| `producer.*`、`consumer.*`、`admin.*` | Worker 创建的客户端默认值 | 分布式模式下分别配置内部 producer/consumer/AdminClient，以及 Source Task producer 的默认参数 |
| `key.converter`、`value.converter`、`header.converter` | Worker 记录序列化默认值 | Worker 把 `SourceRecord` 转成 Kafka key/value/header 字节 |

启动 Worker 时不需要提交 Connector；提交 Connector 后，Worker 才按配置创建 Task 和对应客户端。Worker 层的 `producer.*` 是 Source Task producer 的默认值，Connector 层的 `producer.override.*` 可以覆盖单个 Connector 的 producer 配置。反向心跳就是利用这一机制把 producer 指向 wbads。

同理，converter 配置不是“任务提交配置”，而是 Worker 的记录序列化默认值。MM2 需要按字节透传，所以使用 `ByteArrayConverter`。本文在 Worker 层和每个 Connector JSON 里都显式配置，二者取其一即可；保留 Worker 层配置可避免 Connector 遗漏配置时退化为 JSON 转换，保留 Connector 层配置则便于迁移到托管 Connect。

凭据为什么会重复：

- Worker properties 里的 `producer.*`、`consumer.*`、`admin.*` 是 Kafka Connect 框架的客户端配置命名空间，分别作用于框架创建的 Source Task producer、内部 consumer、AdminClient。安全集群上这三类客户端不会统一继承一个全局身份，所以要分别写。
- Connector JSON 里的 `source.cluster.*`、`target.cluster.*` 是 MM2 Connector 自己识别的配置，用来创建访问源/目标集群的 AdminClient、Consumer 或执行集群管理操作。它们不是 Kafka Connect 的通用权限配置，因此不能替代 Worker 的 `producer.*`。
- `producer.override.*` 又是 Kafka Connect 的 Source Task producer 覆盖命名空间。反向心跳需要用它把框架 producer 从默认 wbbz 改到 wbads；它和 `target.cluster.*` 表达的是同一个集群，但作用在不同客户端上，所以凭据看起来重复。
- 本方案的管理集群、目标集群和 MM2 操作恰好都能使用同一个 admin 账号，凭据值因此大量相同。这是部署选择造成的重复，不是每个命名空间都必须使用同一身份。

权限看起来多，是因为同一个部署里同时存在四类访问：

1. Worker 加入 `group.id` 的组协调权限。
2. Worker 读写 config/offset/status 三个内部 topic 的权限。
3. MM2 在源集群列举 topic/group、读取业务数据、写 offset-sync 的权限。
4. MM2 在目标集群创建/扩容 topic、写业务数据、写 checkpoint/heartbeat、同步 group offset 的权限。

使用 admin 账号只是简化部署，不代表 Connect 本身必须这么大权限。生产环境可按第 3.3 节拆分最小权限账号。

### 4.2 地址与凭据约定

全文统一使用以下取值：

```bash
# wbads（源集群）
WBADS_BS='10.26.28.41:9111,10.26.28.29:9111,10.26.28.28:9111,10.78.18.47:9111,10.78.18.46:9111'

# wbbz（目标集群，兼 Connect 管理 Kafka）
WBBZ_BS='10.75.12.95:9111,10.75.12.96:9111,10.75.12.97:9111,10.52.140.33:9111,10.52.140.34:9111'

# Connect REST（示例 Worker 地址，多节点时替换为任一 Worker）
CONNECT=http://10.52.139.55:18088
```

两个集群使用同一 admin 账号，SASL 配置统一为：

```properties
security.protocol=SASL_PLAINTEXT
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="admin" password="d4a12dfe3f97e641edd9f206eca5ae92";
```

下文所有 properties 和 JSON 中的集群地址、SASL 配置均按此填写，不再重复说明。生产环境建议使用 `FileConfigProvider` 或其他 Secret 管理方式，避免凭据长期明文存放。

### 4.3 文件规划

```text
/opt/kafka/config/
├── mm2-connect-wbbz.properties    # Connect Worker 配置
├── wbads-client.properties        # 命令行客户端配置
├── wbbz-client.properties
└── mm2/
    ├── mm2-wbads-to-wbbz-source.json
    ├── mm2-wbads-to-wbbz-checkpoint.json
    ├── mm2-wbads-to-wbbz-heartbeat.json   # 可选
    ├── mm2-wbbz-to-wbads-heartbeat.json   # 可选
    └── apply.sh                   # 直接提交完整 JSON
```

### 4.4 Kafka 客户端配置

`wbads-client.properties` 与 `wbbz-client.properties` 内容相同（同一 admin 账号），即 4.2 中的 SASL 三行，分别保存为两个文件即可。

### 4.5 预创建 Connect 内部 topic（建议，非必须）

Connect 管理 Kafka 使用 wbbz，以下三个 topic 位于 wbbz。手工预创建不是硬性要求：如果 topic 不存在，Connect 会尝试自动创建，并按 Worker 配置设置 partition 数、副本数和 `cleanup.policy=compact`。这里的“自动创建”是 Connect 通过 AdminClient 显式执行 `createTopics`，不是依赖 broker 的 `auto.create.topics.enable`。较新的 Kafka 版本还提供 `internal.topics.automatic.creation.enable`，默认开启；如果显式关闭该开关，或者 Worker 账号没有创建 topic 的权限，才必须手工预创建。手工预创建的好处是不依赖启动期的创建权限，也能避免已有 topic 配置不符合要求。

如果选择手工预创建，可执行：

```bash
WBBZ_CLIENT=/opt/kafka/config/wbbz-client.properties

kafka-topics.sh --bootstrap-server "$WBBZ_BS" \
  --command-config "$WBBZ_CLIENT" \
  --create --if-not-exists \
  --topic connect-mm2-wbads-wbbz-configs \
  --partitions 1 --replication-factor 3 \
  --config cleanup.policy=compact

kafka-topics.sh --bootstrap-server "$WBBZ_BS" \
  --command-config "$WBBZ_CLIENT" \
  --create --if-not-exists \
  --topic connect-mm2-wbads-wbbz-offsets \
  --partitions 25 --replication-factor 3 \
  --config cleanup.policy=compact

kafka-topics.sh --bootstrap-server "$WBBZ_BS" \
  --command-config "$WBBZ_CLIENT" \
  --create --if-not-exists \
  --topic connect-mm2-wbads-wbbz-status \
  --partitions 5 --replication-factor 3 \
  --config cleanup.policy=compact
```

要求：

- config topic 必须只有一个 partition。
- 三个 topic 都应使用 `cleanup.policy=compact`。
- topic 名应专用于该 Connect 集群，不与其他 Connect 集群共用。
- `config.storage.topic`、`offset.storage.topic`、`status.storage.topic` 这三个 Worker 配置项本身是必需的；无论 topic 是手工创建还是自动创建，都不能省略。
- 如果 topic 已经存在，Worker 配置中的 partition 数和副本数不会反向修改已有 topic。
- `--if-not-exists` 不会修正已存在 topic 的配置；上线前要用 `--describe` 确认 partition、副本和 `cleanup.policy`。

MM2 自己的 `mm2-offset-syncs.wbbz.internal`、`wbads.checkpoints.internal` 和 `heartbeats` 由 Connector 自动创建，副本数由 Connector 配置控制。

### 4.6 Worker 配置

`/opt/kafka/config/mm2-connect-wbbz.properties`（地址与凭据即 4.2 约定值）：

```properties
# Connect 管理 Kafka：wbbz
bootstrap.servers=10.75.12.95:9111,10.75.12.96:9111,10.75.12.97:9111,10.52.140.33:9111,10.52.140.34:9111

group.id=mm2-wbads-to-wbbz-connect

config.storage.topic=connect-mm2-wbads-wbbz-configs
config.storage.replication.factor=3

offset.storage.topic=connect-mm2-wbads-wbbz-offsets
offset.storage.replication.factor=3
offset.storage.partitions=25

status.storage.topic=connect-mm2-wbads-wbbz-status
status.storage.replication.factor=3
status.storage.partitions=5

# MM2 按字节透传
key.converter=org.apache.kafka.connect.converters.ByteArrayConverter
value.converter=org.apache.kafka.connect.converters.ByteArrayConverter
header.converter=org.apache.kafka.connect.converters.ByteArrayConverter

# REST。每台 Worker 使用自己的监听和 advertised 地址。
listeners=http://10.52.139.55:18088
rest.advertised.host.name=10.52.139.55
rest.advertised.port=18088
rest.advertised.listener=HTTP

# 允许 Connector 覆盖 SourceTask producer 的目标集群（反向心跳、独立管理集群场景依赖此项）
connector.client.config.override.policy=All

offset.flush.interval.ms=10000
offset.flush.timeout.ms=30000
scheduled.rebalance.max.delay.ms=300000

# Worker 管理客户端访问 wbbz
security.protocol=SASL_PLAINTEXT
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="admin" password="d4a12dfe3f97e641edd9f206eca5ae92";

# Source Task producer 的 Worker 层默认值；单 Connector 可用 producer.override.* 覆盖
producer.security.protocol=SASL_PLAINTEXT
producer.sasl.mechanism=PLAIN
producer.sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="admin" password="d4a12dfe3f97e641edd9f206eca5ae92";
producer.max.request.size=11457280
producer.batch.size=897152
producer.compression.type=snappy

# Connect 内部 consumer 访问 wbbz
consumer.security.protocol=SASL_PLAINTEXT
consumer.sasl.mechanism=PLAIN
consumer.sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="admin" password="d4a12dfe3f97e641edd9f206eca5ae92";

# Connect 内部 AdminClient 访问 wbbz
admin.security.protocol=SASL_PLAINTEXT
admin.sasl.mechanism=PLAIN
admin.sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="admin" password="d4a12dfe3f97e641edd9f206eca5ae92";

# 原配置未开启 exactly-once
exactly.once.source.support=disabled
```

说明：

- 原配置中的 `dedicated.mode.enable.internal.rest` 只属于专用模式，普通 Connect Worker 不使用该参数；`listeners` 仍然有效，但它现在是标准 Connect REST API。
- 多节点部署时，每台 Worker 的 `listeners` 和 `rest.advertised.host.name` 必须使用本机可达地址。
- 原配置中被注释的 `producer.acks=1` 不生效。建议保持 Connect 默认的可靠性设置，不要改成 `acks=1`。
- 不需要配置 `internal.key.converter` 和 `internal.value.converter`。

配置必要性：

- 必需：`bootstrap.servers`、`group.id`、三个 storage topic 名称，以及 Byte passthrough 用的 key/value/header converter（Worker 层或 Connector 层至少配置一处）；各 Connector 还需要 `source.cluster.alias` 和 `target.cluster.alias`，Source/Checkpoint 另需要 `source.cluster.bootstrap.servers` 和 `target.cluster.bootstrap.servers`。
- 业务语义必需：`replication.policy.class=org.apache.kafka.connect.mirror.IdentityReplicationPolicy`；如果改回默认策略，目标 topic 会带上源集群前缀。
- 条件必需：多 Worker 场景的 REST listener 和 advertised 地址；反向心跳或独立管理集群场景的 `producer.override.*`；如果 Connect 管理 Kafka 与目标 Kafka 不同，Source/Checkpoint/正向心跳还需要 `producer.override.*` 指向目标集群。
- 安全集群必需：`producer.*`、`consumer.*`、`admin.*` 前缀下的 SASL/SSL 参数，用于让不同客户端访问对应 Kafka。
- 建议但非必需：`connector.client.config.override.policy=All`（Kafka 3.x 默认就是 `All`，显式写出可避免被全局改成 `None` 或 `Allowlist`）、`exactly.once.source.support=disabled`（默认值）、`offset.flush.*`、`scheduled.rebalance.max.delay.ms`、producer 批量和压缩参数。
- Worker 顶层安全配置用于 Connect 管理 Kafka；`producer.*`、`consumer.*`、`admin.*` 分别用于 SourceTask producer、内部 consumer 和 AdminClient。安全集群上这些前缀通常都需要配置，不能只依赖顶层配置。

### 4.7 启动 Worker

建议至少部署三个 Worker。每台机器：

```bash
export KAFKA_HEAP_OPTS='-Xms4g -Xmx4g'

bin/connect-distributed.sh -daemon \
  /opt/kafka/config/mm2-connect-wbbz.properties
```

检查：

```bash
curl -fsS "$CONNECT/"

curl -fsS "$CONNECT/connector-plugins" |
  grep 'org.apache.kafka.connect.mirror'
```

应至少看到：

```text
org.apache.kafka.connect.mirror.MirrorSourceConnector
org.apache.kafka.connect.mirror.MirrorCheckpointConnector
org.apache.kafka.connect.mirror.MirrorHeartbeatConnector
```

## 5. Connector 配置

每个 JSON 都是提交给 `PUT /connectors/{name}/config` 的完整请求体，不再拆公共片段，也不依赖外部 JSON 处理工具。凭据在多个 JSON 中重复出现是当前写法的代价；生产环境可用配置模板或 secret 管理工具生成这些文件。

### 5.1 mm2-wbads-to-wbbz-source.json

```json
{
  "connector.class": "org.apache.kafka.connect.mirror.MirrorSourceConnector",
  "tasks.max": "200",
  "replication.policy.class": "org.apache.kafka.connect.mirror.IdentityReplicationPolicy",
  "key.converter": "org.apache.kafka.connect.converters.ByteArrayConverter",
  "value.converter": "org.apache.kafka.connect.converters.ByteArrayConverter",
  "header.converter": "org.apache.kafka.connect.converters.ByteArrayConverter",

  "source.cluster.alias": "wbads",
  "source.cluster.bootstrap.servers": "10.26.28.41:9111,10.26.28.29:9111,10.26.28.28:9111,10.78.18.47:9111,10.78.18.46:9111",
  "source.cluster.security.protocol": "SASL_PLAINTEXT",
  "source.cluster.sasl.mechanism": "PLAIN",
  "source.cluster.sasl.jaas.config": "org.apache.kafka.common.security.plain.PlainLoginModule required username=\"admin\" password=\"d4a12dfe3f97e641edd9f206eca5ae92\";",
  "target.cluster.alias": "wbbz",
  "target.cluster.bootstrap.servers": "10.75.12.95:9111,10.75.12.96:9111,10.75.12.97:9111,10.52.140.33:9111,10.52.140.34:9111",
  "target.cluster.security.protocol": "SASL_PLAINTEXT",
  "target.cluster.sasl.mechanism": "PLAIN",
  "target.cluster.sasl.jaas.config": "org.apache.kafka.common.security.plain.PlainLoginModule required username=\"admin\" password=\"d4a12dfe3f97e641edd9f206eca5ae92\";",

  "source.consumer.max.partition.fetch.bytes": "8388608",

  "topics": ".*",
  "refresh.topics.enabled": "true",
  "refresh.topics.interval.seconds": "60",

  "replication.factor": "3",

  "sync.topic.configs.enabled": "false",
  "sync.topic.acls.enabled": "false",

  "emit.offset-syncs.enabled": "true",
  "offset-syncs.topic.location": "source",
  "offset-syncs.topic.replication.factor": "3"
}
```

`tasks.max=200` 只是上限，实际 Task 数不会超过匹配到的源 topic partition 数。

### 5.2 mm2-wbads-to-wbbz-checkpoint.json


```json
{
  "connector.class": "org.apache.kafka.connect.mirror.MirrorCheckpointConnector",
  "tasks.max": "200",
  "replication.policy.class": "org.apache.kafka.connect.mirror.IdentityReplicationPolicy",
  "key.converter": "org.apache.kafka.connect.converters.ByteArrayConverter",
  "value.converter": "org.apache.kafka.connect.converters.ByteArrayConverter",
  "header.converter": "org.apache.kafka.connect.converters.ByteArrayConverter",

  "source.cluster.alias": "wbads",
  "source.cluster.bootstrap.servers": "10.26.28.41:9111,10.26.28.29:9111,10.26.28.28:9111,10.78.18.47:9111,10.78.18.46:9111",
  "source.cluster.security.protocol": "SASL_PLAINTEXT",
  "source.cluster.sasl.mechanism": "PLAIN",
  "source.cluster.sasl.jaas.config": "org.apache.kafka.common.security.plain.PlainLoginModule required username=\"admin\" password=\"d4a12dfe3f97e641edd9f206eca5ae92\";",
  "target.cluster.alias": "wbbz",
  "target.cluster.bootstrap.servers": "10.75.12.95:9111,10.75.12.96:9111,10.75.12.97:9111,10.52.140.33:9111,10.52.140.34:9111",
  "target.cluster.security.protocol": "SASL_PLAINTEXT",
  "target.cluster.sasl.mechanism": "PLAIN",
  "target.cluster.sasl.jaas.config": "org.apache.kafka.common.security.plain.PlainLoginModule required username=\"admin\" password=\"d4a12dfe3f97e641edd9f206eca5ae92\";",

  "topics": ".*",
  "groups": ".*",

  "refresh.groups.enabled": "true",
  "refresh.groups.interval.seconds": "60",

  "emit.checkpoints.enabled": "true",
  "emit.checkpoints.interval.seconds": "60",
  "checkpoints.topic.replication.factor": "3",

  "sync.group.offsets.enabled": "true",
  "sync.group.offsets.interval.seconds": "60",

  "offset-syncs.topic.location": "source"
}
```

`sync.group.offsets.enabled=true` 写 wbbz `__consumer_offsets` 的风险和前提见 2.1。

### 5.3 mm2-wbads-to-wbbz-heartbeat.json

心跳只写目标集群，不需要 `source.cluster.bootstrap.servers`，但 `source.cluster.alias` 是必需配置，用于生成心跳内容和识别 `IdentityReplicationPolicy` 下的心跳链路。

```json
{
  "connector.class": "org.apache.kafka.connect.mirror.MirrorHeartbeatConnector",
  "tasks.max": "1",
  "replication.policy.class": "org.apache.kafka.connect.mirror.IdentityReplicationPolicy",
  "key.converter": "org.apache.kafka.connect.converters.ByteArrayConverter",
  "value.converter": "org.apache.kafka.connect.converters.ByteArrayConverter",
  "header.converter": "org.apache.kafka.connect.converters.ByteArrayConverter",
  "source.cluster.alias": "wbads",
  "target.cluster.alias": "wbbz",
  "target.cluster.bootstrap.servers": "10.75.12.95:9111,10.75.12.96:9111,10.75.12.97:9111,10.52.140.33:9111,10.52.140.34:9111",
  "target.cluster.security.protocol": "SASL_PLAINTEXT",
  "target.cluster.sasl.mechanism": "PLAIN",
  "target.cluster.sasl.jaas.config": "org.apache.kafka.common.security.plain.PlainLoginModule required username=\"admin\" password=\"d4a12dfe3f97e641edd9f206eca5ae92\";",

  "emit.heartbeats.enabled": "true",
  "emit.heartbeats.interval.seconds": "5",
  "heartbeats.topic.replication.factor": "3"
}
```

### 5.4 mm2-wbbz-to-wbads-heartbeat.json

`source.cluster.alias` 为 `wbbz`、`target.cluster.alias` 为 `wbads`；通过 `producer.override.*` 把 Source Task producer 指向 wbads，依赖 Worker 的 `connector.client.config.override.policy=All`。

`producer.override.*` 不能省略。省略后该 Connector 不会写 wbads，而会写 Worker 默认集群 wbbz；`target.cluster.*` 只影响 MM2 自己创建的 AdminClient，不影响 Connect 框架的 Source Task producer。

```json
{
  "connector.class": "org.apache.kafka.connect.mirror.MirrorHeartbeatConnector",
  "tasks.max": "1",
  "replication.policy.class": "org.apache.kafka.connect.mirror.IdentityReplicationPolicy",
  "key.converter": "org.apache.kafka.connect.converters.ByteArrayConverter",
  "value.converter": "org.apache.kafka.connect.converters.ByteArrayConverter",
  "header.converter": "org.apache.kafka.connect.converters.ByteArrayConverter",
  "source.cluster.alias": "wbbz",
  "target.cluster.alias": "wbads",
  "target.cluster.bootstrap.servers": "10.26.28.41:9111,10.26.28.29:9111,10.26.28.28:9111,10.78.18.47:9111,10.78.18.46:9111",
  "target.cluster.security.protocol": "SASL_PLAINTEXT",
  "target.cluster.sasl.mechanism": "PLAIN",
  "target.cluster.sasl.jaas.config": "org.apache.kafka.common.security.plain.PlainLoginModule required username=\"admin\" password=\"d4a12dfe3f97e641edd9f206eca5ae92\";",

  "producer.override.bootstrap.servers": "10.26.28.41:9111,10.26.28.29:9111,10.26.28.28:9111,10.78.18.47:9111,10.78.18.46:9111",
  "producer.override.security.protocol": "SASL_PLAINTEXT",
  "producer.override.sasl.mechanism": "PLAIN",
  "producer.override.sasl.jaas.config": "org.apache.kafka.common.security.plain.PlainLoginModule required username=\"admin\" password=\"d4a12dfe3f97e641edd9f206eca5ae92\";",

  "emit.heartbeats.enabled": "true",
  "emit.heartbeats.interval.seconds": "5",
  "heartbeats.topic.replication.factor": "3"
}
```

### 5.5 提交与校验

`/opt/kafka/config/mm2/apply.sh`：

```bash
#!/usr/bin/env bash
set -euo pipefail

CONNECT=${CONNECT:-http://10.52.139.55:18088}
cd "$(dirname "$0")"

apply() {
  local name=$1 file=$2
  echo "Applying $name"
  curl -fsS -X PUT "$CONNECT/connectors/$name/config" \
    -H 'Content-Type: application/json' \
    --data-binary @"$file"
}

# 必需：业务数据复制
apply mm2-wbads-to-wbbz-source mm2-wbads-to-wbbz-source.json

# 如需 consumer group offset 同步
apply mm2-wbads-to-wbbz-checkpoint mm2-wbads-to-wbbz-checkpoint.json

# 心跳监控：正向验证目标集群可写性，反向验证端到端复制路径
apply mm2-wbads-to-wbbz-heartbeat mm2-wbads-to-wbbz-heartbeat.json
apply mm2-wbbz-to-wbads-heartbeat mm2-wbbz-to-wbads-heartbeat.json
```

执行：

```bash
chmod +x /opt/kafka/config/mm2/apply.sh
/opt/kafka/config/mm2/apply.sh
```

PUT 相同配置是幂等的，未发生变化的 Connector 不会重启，日常修改任一 JSON 后重新执行 apply.sh 即可。

提交前可调用配置校验 API（以 Source 为例）。返回体中的 `error_count` 应为 `0`：

```bash
cd /opt/kafka/config/mm2

curl -fsS -X PUT \
  "$CONNECT/connector-plugins/MirrorSourceConnector/config/validate" \
  -H 'Content-Type: application/json' \
  --data-binary @mm2-wbads-to-wbbz-source.json
```

## 6. 上线验收

### 6.1 Connector 和 Task 状态

```bash
curl -fsS "$CONNECT/connectors?expand=status"
```

所有 Connector 和预期 Task 应为 `RUNNING`。Checkpoint 在尚未发现符合条件的 consumer group 时可能暂时没有 Task，这不等同于 Connector 失败。

### 6.2 内部 topic

```bash
kafka-topics.sh --bootstrap-server "$WBADS_BS" \
  --command-config /opt/kafka/config/wbads-client.properties \
  --list | grep -E 'mm2-offset-syncs\.wbbz\.internal|heartbeats'

kafka-topics.sh --bootstrap-server "$WBBZ_BS" \
  --command-config /opt/kafka/config/wbbz-client.properties \
  --list | grep -E 'wbads\.checkpoints\.internal|heartbeats|connect-mm2'
```

### 6.3 业务 topic

目标 topic 与源 topic 同名：

```bash
kafka-topics.sh --bootstrap-server "$WBADS_BS" \
  --command-config /opt/kafka/config/wbads-client.properties \
  --describe --topic <业务topic>

kafka-topics.sh --bootstrap-server "$WBBZ_BS" \
  --command-config /opt/kafka/config/wbbz-client.properties \
  --describe --topic <业务topic>
```

目标 topic 由 MM2 自动创建，并随源 topic 扩分区，与 `sync.topic.configs.enabled` 无关（见 2.1）。

### 6.4 checkpoint 和 group offset

```bash
kafka-console-consumer.sh \
  --bootstrap-server "$WBBZ_BS" \
  --consumer.config /opt/kafka/config/wbbz-client.properties \
  --topic wbads.checkpoints.internal \
  --from-beginning --max-messages 5

kafka-consumer-groups.sh \
  --bootstrap-server "$WBBZ_BS" \
  --command-config /opt/kafka/config/wbbz-client.properties \
  --describe --group <业务consumer-group>
```

注意用业务 consumer group 查询，不要用 Connect Worker 的 `group.id`（见 2.3）。

## 7. 日常运维

### 7.1 修改同步 topic

当前 `"topics": ".*"` 会自动发现所有非默认排除 topic，新增普通业务 topic 后最长约 60 秒进入复制。如改为白名单，编辑 `mm2-wbads-to-wbbz-source.json`：

```text
"topics": "topic-a,topic-b,order-.*"
```

然后重新执行 apply.sh。`topics` 中每一项都是 Java 正则并使用整串匹配：`order` 只匹配 topic `order`；`order-.*` 匹配 `order-a`、`order-2026`；JSON 中匹配字面量点号需写成 `\\.`。显式配置 `topics.exclude` 时必须保留默认排除项（见 2.1）。

### 7.2 修改同步 group

编辑 `mm2-wbads-to-wbbz-checkpoint.json` 的 `groups` 后重新执行 apply.sh：

```text
"groups": "group-a,group-b,order-service-.*"
```

### 7.3 常用 REST 操作

```bash
NAME=mm2-wbads-to-wbbz-source

curl -fsS "$CONNECT/connectors/$NAME/status"

curl -fsS -X POST \
  "$CONNECT/connectors/$NAME/restart?includeTasks=true&onlyFailed=true"

curl -fsS -X PUT "$CONNECT/connectors/$NAME/pause"
curl -fsS -X PUT "$CONNECT/connectors/$NAME/resume"
curl -fsS -X PUT "$CONNECT/connectors/$NAME/stop"

curl -fsS "$CONNECT/connectors/$NAME/offsets"
```

删除 Connector 不等于清除 Source offset。需要重跑时：先停止 Connector，再用 offset REST API 删除或修改 offset，最后恢复 Connector。执行 offset 删除会导致重新复制，必须先评估下游重复数据。

### 7.4 扩缩 Worker

扩容：

- 在新机器安装相同 Kafka 版本和插件。
- 使用相同 `group.id` 及 config/offset/status topic。
- 使用新机器自己的 REST listener 和 advertised 地址。
- 启动后 Connect 自动重新分配 Task。

缩容：

- 一次停止一台 Worker。
- 等待 rebalance 完成并确认 Task 全部恢复 RUNNING。
- 再停止下一台。

## 8. 监控

| 对象 | 指标或检查 |
|---|---|
| Connector/Task | REST `/status` 中是否存在 `FAILED` |
| 数据复制 | `kafka.connect.mirror` 下的 record age、replication latency、record count |
| Source 吞吐 | source record poll/write rate |
| Worker | rebalance、JVM heap、GC pause、线程数 |
| 内部 topic | Connect config/offset/status topic 是否可写 |
| checkpoint | `wbads.checkpoints.internal` 是否持续更新 |
| group offset | wbbz 目标 group offset 是否按预期更新 |
| heartbeat | 可选：wbads 的 `heartbeats`，以及 wbbz 的 `heartbeats` 和 `wbads.heartbeats` 是否持续更新 |

复制 lag 应按业务 topic partition 比较 wbads end offset 与 wbbz end offset，或使用 MM2 自身指标；Connect Worker 的 `group.id` 不是源端消费组（见 2.3）。

## 9. 变体

### 9.1 独立 Connect 管理集群

Connect 管理 Kafka 可以使用第三个独立集群 M，不要求必须是 wbbz。Worker 配置差异：

```properties
bootstrap.servers=<M集群地址>
config.storage.topic=<M上的config topic>
offset.storage.topic=<M上的offset topic>
status.storage.topic=<M上的status topic>
```

`group.id`、`connector.client.config.override.policy=All` 等其余配置不变。

此时 Worker 默认 Source Task producer 会写 M，必须为 Source、Checkpoint、正向心跳三个 Connector 增加指向 wbbz 的 `producer.override.*`（写法与 5.4 反向心跳相同，地址和凭据换成 wbbz）；反向心跳维持覆盖到 wbads 不变。

这里的规则是：Source Task producer 由 Connect Worker 创建，初始 `bootstrap.servers` 取 Worker 的 `bootstrap.servers`，随后合并 Worker 的 `producer.*` 默认值，最后再合并 Connector 配置中的 `producer.override.*`。因此“Source Connector 写 Worker 集群”是默认行为，不是不可覆盖的硬约束；能否覆盖取决于 `connector.client.config.override.policy`。本方案使用 `All`，所以可以把 Source/Checkpoint/正向心跳的 producer 指到 wbbz，把反向心跳的 producer 指到 wbads。如果 Connect 集群的策略是 `None`，或 `Allowlist` 未放行 `bootstrap.servers` 和 SASL 参数，则独立管理集群 M 的方案不能按本文运行。

`target.cluster.*` 只影响 MM2 自己创建的 AdminClient、Consumer 等客户端，不能控制 Connect 框架的 Source Task producer；控制后者必须使用 `producer.override.*`。

普通 at-least-once 模式下，Connector 的 Source offset 可以继续存放在 M 的 Worker 全局 offset topic。M 不可用时，Connect 无法完成配置管理、offset 提交和任务恢复，因此 M 仍是关键依赖。

### 9.2 Exactly-once

原配置没有真正开启 exactly-once，注释中的 `wbbz.exactly.once.wbads.support` 不是有效参数。如需开启，至少需要：

Worker：

```properties
exactly.once.source.support=enabled
```

Source Connector：

```text
"exactly.once.support": "required",
"source.consumer.isolation.level": "read_committed"
```

如果 Connect 管理 Kafka 与目标 wbbz 分离，还需要让业务记录和 Connector 专属 Source offset 位于 wbbz：

```text
"offsets.storage.topic": "connect-mm2-wbads-wbbz-source-offsets",
"producer.override.bootstrap.servers": "<wbbz>",
"consumer.override.bootstrap.servers": "<wbbz>",
"admin.override.bootstrap.servers": "<wbbz>"
```

并为 producer、consumer、admin override 配齐 SASL 参数。

现网集群从 disabled 升级 exactly-once 时，应先将所有 Worker 配为 `preparing` 并滚动重启，再改为 `enabled` 进行第二轮滚动重启。不要直接在部分 Worker 上启用。

## 10. 从专用模式迁移

### 10.1 风险

如果直接创建新的 Connect group、内部 topic 和 Connector 名称，普通 Connect 看不到专用模式原来的 Source offset。由于 MirrorSourceTask 默认从 earliest 开始，没有迁移 offset 时可能全量重复复制。

切换前必须停止所有 `connect-mirror-maker.sh` 进程，禁止专用模式和新 Connect Connector 同时向相同目标 topic 写数据。

### 10.2 推荐方案：复用专用模式状态

专用模式本身使用 `DistributedHerder` 和 Kafka config/offset/status topic。其默认内部 topic 名包含 source alias 而不是 target alias。对于 `wbads -> wbbz`，默认值为：

```properties
group.id=wbads-mm2
config.storage.topic=mm2-configs.wbads.internal
offset.storage.topic=mm2-offsets.wbads.internal
status.storage.topic=mm2-status.wbads.internal
```

这些 topic 位于目标 wbbz。如要原位接管：

1. 停止全部专用模式节点。
2. 确认 wbbz 上存在上述 topic。
3. 普通 Connect Worker 使用相同 `group.id` 和三个内部 topic。
4. Connector 名称保持专用模式名称：`MirrorSourceConnector`、`MirrorCheckpointConnector`、`MirrorHeartbeatConnector`。
5. 启动一个 Worker 验证现有配置和 offset 被正确加载。
6. 确认没有回放后再扩至多个 Worker。
7. 通过 REST 更新原有 Connector 配置。

不要在复用旧 config topic 时同时创建本文的新 Connector 名称，否则会产生两套 Source Connector 并双写。

如果专用模式未使用 `--clusters wbbz` 限制目标，wbads 上还可能存在反向 herder：

```text
group.id=wbbz-mm2
mm2-configs.wbbz.internal
mm2-offsets.wbbz.internal
mm2-status.wbbz.internal
```

其中通常只有反向 Heartbeat 有活动 Task。可以在 wbads 侧启动第二套普通 Connect Worker 复用这些状态，或者不复用反向 herder、改为本文的 `mm2-wbbz-to-wbads-heartbeat.json`。两种方式只能选一种，不能同时运行。

### 10.3 新建 Connect 集群

如果选择第 4 节的新 group 和新内部 topic，必须接受从 earliest 重新复制，或者在停机窗口使用 Connect offset API 设置每个源 topic partition 的起始 offset。

在没有完成 offset 验证前，不要删除旧专用模式内部 topic。

## 11. 故障速查

| 现象 | 可能原因 | 处理 |
|---|---|---|
| REST 请求在 Worker 间转发失败 | advertised 地址不可达 | 修正 `rest.advertised.*` 并重启 Worker |
| Heartbeat Connector 启动失败 | 缺少 `source.cluster.alias` | 按方向补齐 `wbads` 或 `wbbz` |
| Source Task FAILED | wbads 认证、权限或网络错误 | 查看 `/status` 中的 trace |
| Connector RUNNING 但无数据 | `topics` 正则未匹配，或源 topic 无新数据 | 检查配置及源端 end offset |
| wbbz 没有目标 topic | target AdminClient 权限不足 | 检查 `target.cluster.*` 认证和 CREATE 权限 |
| wbads 无 offset-sync topic | source 端无 CREATE/WRITE 权限 | 授权或改为 `offset-syncs.topic.location=target` |
| checkpoint 无数据 | 没有符合条件的 group，或 offset-sync 尚未形成 | 检查 groups、业务消费进度及 offset-sync |
| wbbz group offset 不更新 | wbbz 上该 group 有活跃成员 | 停止目标消费者后等待下一同步周期 |
| 数据重复 | 重建了 Connector 名称、清空 offset、迁移未复用旧 offset，或有两套 Source 同时运行 | 停止重复链路并核对 Source offset |
| 目标 topic partition 少 | refresh 尚未执行或 target ALTER 权限不足 | 检查 refresh 和目标 Admin 权限 |
| 反向心跳写到 wbbz 而不是 wbads | 缺少 `producer.override.bootstrap.servers` | 修正 reverse heartbeat 配置 |
| Connect 内部 topic 不可用 | wbbz 故障或 Worker 安全配置错误 | 检查 Worker 顶层及 producer/consumer/admin 配置 |

## 12. 上线检查清单

- [ ] 所有 Worker 使用相同 `group.id` 和内部 topic。
- [ ] 每台 Worker 使用自己的可路由 REST advertised 地址。
- [ ] Connect config topic 只有一个 partition。
- [ ] 三个 Connect 内部 topic 均为 compact。
- [ ] 已确认内部 topic 是手工预创建还是依赖 Connect 自动创建，并核对对应权限。
- [ ] Worker key/value/header converter 均为 ByteArrayConverter。
- [ ] Source/Checkpoint 使用 `source.cluster.alias=wbads`、`target.cluster.alias=wbbz`。
- [ ] Source 写入 wbbz；如启用 Checkpoint，Checkpoint 也写入 wbbz。
- [ ] 已确认是否启用 Heartbeat。启用时，正向 Heartbeat 写入 wbbz，反向 Heartbeat 通过 producer override 写入 wbads。
- [ ] 已确认正向 Heartbeat 的 `source.cluster.alias=wbads`，反向 Heartbeat 的 `source.cluster.alias=wbbz`。
- [ ] `mm2-offset-syncs.wbbz.internal` 位于 wbads。
- [ ] `wbads.checkpoints.internal` 位于 wbbz。
- [ ] 业务 topic 在 wbbz 保持原名。
- [ ] `sync.group.offsets.enabled=true` 的风险已由业务确认。
- [ ] 同一个 consumer group 不会同时在 wbads 和 wbbz 活跃消费。
- [ ] 已确认专用模式进程全部停止，不存在双写。
- [ ] 已确认迁移方案是否复用旧 Source offset。
- [ ] 已配置 Connector/Task、复制延迟、JVM 和内部 topic 告警。
