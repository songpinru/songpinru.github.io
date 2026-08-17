---
title: "MirrorMaker2 On Connect Distributed 部署运维手册"
---

# MirrorMaker 2 on Kafka Connect Distributed 部署运维手册

> 文档版本：v2.0
>
> 更新时间：2026-08-14
>
> 适用链路：`wbads -> wbbz`
>
> 适用模式：Kafka Connect Distributed 直接运行 MM2 Connector

## 1. 目标与结论

本文将现有 `connect-mirror-maker.sh` 专用模式配置转换为普通 Kafka Connect Distributed 部署。

转换后具备以下能力：

- 通过 Connect REST API 动态修改同步 topic 和 group，无需重启 Worker。
- 多个 Worker 自动分配 Connector 和 Task。
- 保持现有单向复制、topic 不加前缀、consumer group offset 自动同步等语义。
- 保持专用模式的正向和反向心跳行为。

推荐拓扑：

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

本文部署四个活动 Connector：

| Connector 名称 | 类 | 作用 |
|---|---|---|
| `mm2-wbads-to-wbbz-source` | `MirrorSourceConnector` | 复制业务数据、分区信息和 offset-sync |
| `mm2-wbads-to-wbbz-checkpoint` | `MirrorCheckpointConnector` | 生成 checkpoint，并同步目标 group offset |
| `mm2-wbads-to-wbbz-heartbeat` | `MirrorHeartbeatConnector` | 向 wbbz 写正向心跳 |
| `mm2-wbbz-to-wbads-heartbeat` | `MirrorHeartbeatConnector` | 向 wbads 写反向心跳，监控完整复制链路 |

## 2. 原专用配置的准确解释

原配置的有效语义如下：

| 原配置 | 实际含义 |
|---|---|
| `wbads->wbbz.enabled=true` | 开启业务数据从 wbads 到 wbbz 的复制 |
| `wbbz->wbads.enabled=false` | 不复制业务数据和 checkpoint 到 wbads |
| `emit.heartbeats.enabled=true` | 两个方向都会创建 Heartbeat Task |
| `wbads->wbbz.topics=.*` | 复制所有未被默认排除规则排除的 topic |
| `wbads->wbbz.groups=.*` | 处理所有未被默认排除规则排除的 consumer group |
| `IdentityReplicationPolicy` | wbbz 上的目标 topic 与 wbads 同名 |
| `sync.group.offsets.enabled=true` | 将翻译后的 offset 写入 wbbz 的 `__consumer_offsets` |
| 未设置 `offset-syncs.topic.location` | 使用默认值 `source`，offset-sync topic 位于 wbads |

### 2.1 反向 Heartbeat 不能遗漏

专用模式中，即使：

```properties
wbbz->wbads.enabled=false
```

只要全局心跳开启，`MirrorMakerConfig.clusterPairs()`仍会创建 `wbbz -> wbads` herder。原因是反向心跳先写入 wbads，然后由正向 Source Connector 复制到 wbbz，用于观察完整复制路径。

普通 Connect 模式不会自动完成这一步，因此本文显式部署 `mm2-wbbz-to-wbads-heartbeat`。

### 2.2 默认内部 topic 位置

本手册保持原配置默认行为：

| topic | 所在 Kafka 集群 | 作用 |
|---|---|---|
| `mm2-offset-syncs.wbbz.internal` | wbads | 源 offset 到目标 offset 的映射 |
| `wbads.checkpoints.internal` | wbbz | consumer group checkpoint |
| `heartbeats` | wbads 和 wbbz | 链路心跳 |
| Connect config/offset/status topic | wbbz | Connect 集群管理和 Source Task 进度 |

如需将 offset-sync topic 改到 wbbz，必须同时在 Source 和 Checkpoint Connector 设置：

```text
"offset-syncs.topic.location": "target"
```

这是行为变更，不属于原配置的等价转换。

## 3. 重要修正

原手册中以下说法不准确，本版本已修正：

1. Connect 管理 Kafka 不强制等于下游 Kafka。可以使用独立管理集群，但必须用 `producer.override.*`指定 SourceRecord 的目标集群。
2. `group.id`是 Connect Worker 集群的组，不是 MM2 在 wbads 上消费数据的 consumer group。
3. MirrorSourceTask 使用手工 partition assignment，不能用 `kafka-consumer-groups --group <Connect group.id>`监控复制 lag。
4. `sync.topic.configs.enabled=false`只关闭 topic 配置属性同步，不会关闭目标 topic 创建和分区扩容。
5. 删除 Connector 默认不会自动删除其 Source offset；不要把“删除 Connector”等同于“清空 offset”。
6. `refresh.topics`会重新发现源 topic/partition 并重配 Task；源 topic 消失后会停止处理，但不会删除目标 topic。
7. 专用模式 A 到 B 的默认 Connect 内部 topic 名包含 source alias，即 `mm2-*.A.internal`，不是 target alias。
8. 原注释中的 `wbbz.exactly.once.wbads.support`不是有效配置。

## 4. 上线前检查

### 4.1 IdentityReplicationPolicy 冲突

当前策略让普通业务 topic 在 wbads 和 wbbz 上同名。上线前必须确认：

- wbbz 上的同名 topic 不是另一条独立生产链路的写入目标。
- 不存在另一套 `wbbz -> wbads`业务复制。
- 目标同名 topic 的 partition 数不大于源 topic；Kafka 不支持缩减 partition。
- 下游应用能够接受镜像数据直接进入现有同名 topic。

### 4.2 消息大小

当前参数：

```properties
wbads source consumer max.partition.fetch.bytes=8388608
wbbz Connect producer max.request.size=11457280
```

还应确认：

- wbads broker 允许 consumer 拉取对应大小的 record batch。
- wbbz broker 的 `message.max.bytes`或 topic `max.message.bytes`足够大。
- wbbz follower 的复制 fetch 限制足够大。

否则 Task 可能持续重试或失败。

### 4.3 权限

当前账号为 admin，通常具备完整权限。如果后续改为最小权限账号，至少需要：

| 集群 | 权限 |
|---|---|
| wbads | 读取和描述源业务 topic；列出/描述 consumer group；创建、写入和读取 `mm2-offset-syncs.wbbz.internal`；创建和写入 `heartbeats` |
| wbbz | Connect Worker group 权限；读写 config/offset/status topic；创建、写入和描述目标业务 topic；扩分区；创建/读写 checkpoint 和 heartbeat topic；在启用 group offset 同步时 ALTER consumer group offset |

`sync.topic.configs.enabled=false`和`sync.topic.acls.enabled=false`降低了 topic 配置及 ACL 管理权限需求。

### 4.4 REST 安全

Broker 的 SASL 配置不会保护 Connect REST API。当前使用明文 HTTP：

```properties
listeners=http://10.52.139.55:18088
```

必须通过防火墙、反向代理或专用管理网络限制访问。任何能访问 REST API 的主体都可能修改、停止或删除 Connector。

## 5. 文件规划

建议目录：

```text
/opt/kafka/config/
├── mm2-connect-wbbz.properties
├── wbads-client.properties
├── wbbz-client.properties
└── mm2/
    ├── source.json
    ├── checkpoint.json
    ├── heartbeat-forward.json
    ├── heartbeat-reverse.json
    └── apply.sh
```

## 6. Kafka 客户端配置

### 6.1 wbads-client.properties

```properties
security.protocol=SASL_PLAINTEXT
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="admin" password="d4a12dfe3f97e641edd9f206eca5ae92";
```

### 6.2 wbbz-client.properties

```properties
security.protocol=SASL_PLAINTEXT
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="admin" password="d4a12dfe3f97e641edd9f206eca5ae92";
```

生产环境建议使用 `FileConfigProvider`或其他 Secret 管理方式，避免凭据长期明文存放。

## 7. 创建 Connect 内部 topic

本文使用 wbbz 作为 Connect 管理 Kafka。以下三个 topic 位于 wbbz：

```bash
WBBZ_BS='10.75.12.95:9111,10.75.12.96:9111,10.75.12.97:9111,10.52.140.33:9111,10.52.140.34:9111'
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

MM2 自己的 `mm2-offset-syncs.wbbz.internal`、`wbads.checkpoints.internal`和`heartbeats`可由 Connector 自动创建，副本数由 Connector 配置控制。

## 8. Connect Worker 配置

`/opt/kafka/config/mm2-connect-wbbz.properties`：

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

# 允许 reverse heartbeat 覆盖 SourceTask producer 的目标集群
connector.client.config.override.policy=All

offset.flush.interval.ms=10000
offset.flush.timeout.ms=30000
scheduled.rebalance.max.delay.ms=300000

# Worker 管理客户端访问 wbbz
security.protocol=SASL_PLAINTEXT
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="admin" password="d4a12dfe3f97e641edd9f206eca5ae92";

# SourceTask producer 写 wbbz
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

- 原配置中的 `dedicated.mode.enable.internal.rest`只属于专用模式，普通 Connect Worker 不使用该参数。
- `listeners`仍然有效，但它现在是标准 Connect REST API。
- 多节点部署时，每台 Worker 的 `listeners`和`rest.advertised.host.name`必须使用本机可达地址。
- 原配置中被注释的 `producer.acks=1`不生效。建议保持 Connect 默认的可靠性设置，不要改成 `acks=1`。
- 不需要配置 `internal.key.converter`和`internal.value.converter`。

## 9. 启动 Worker

建议至少部署三个 Worker。每台机器：

```bash
export KAFKA_HEAP_OPTS='-Xms4g -Xmx4g'

bin/connect-distributed.sh -daemon \
  /opt/kafka/config/mm2-connect-wbbz.properties
```

检查：

```bash
CONNECT=http://10.52.139.55:18088

curl -fsS "$CONNECT/" | jq

curl -fsS "$CONNECT/connector-plugins" |
  jq -r '.[].class' |
  grep 'org.apache.kafka.connect.mirror'
```

应至少看到：

```text
org.apache.kafka.connect.mirror.MirrorSourceConnector
org.apache.kafka.connect.mirror.MirrorCheckpointConnector
org.apache.kafka.connect.mirror.MirrorHeartbeatConnector
```

## 10. Connector 配置

### 10.1 source.json

```json
{
  "connector.class": "org.apache.kafka.connect.mirror.MirrorSourceConnector",
  "tasks.max": "200",

  "source.cluster.alias": "wbads",
  "target.cluster.alias": "wbbz",

  "source.cluster.bootstrap.servers": "10.26.28.41:9111,10.26.28.29:9111,10.26.28.28:9111,10.78.18.47:9111,10.78.18.46:9111",
  "source.cluster.security.protocol": "SASL_PLAINTEXT",
  "source.cluster.sasl.mechanism": "PLAIN",
  "source.cluster.sasl.jaas.config": "org.apache.kafka.common.security.plain.PlainLoginModule required username=\"admin\" password=\"d4a12dfe3f97e641edd9f206eca5ae92\";",

  "target.cluster.bootstrap.servers": "10.75.12.95:9111,10.75.12.96:9111,10.75.12.97:9111,10.52.140.33:9111,10.52.140.34:9111",
  "target.cluster.security.protocol": "SASL_PLAINTEXT",
  "target.cluster.sasl.mechanism": "PLAIN",
  "target.cluster.sasl.jaas.config": "org.apache.kafka.common.security.plain.PlainLoginModule required username=\"admin\" password=\"d4a12dfe3f97e641edd9f206eca5ae92\";",

  "source.consumer.max.partition.fetch.bytes": "8388608",

  "topics": ".*",
  "refresh.topics.enabled": "true",
  "refresh.topics.interval.seconds": "60",

  "replication.policy.class": "org.apache.kafka.connect.mirror.IdentityReplicationPolicy",
  "replication.factor": "3",

  "sync.topic.configs.enabled": "false",
  "sync.topic.acls.enabled": "false",

  "emit.offset-syncs.enabled": "true",
  "offset-syncs.topic.location": "source",
  "offset-syncs.topic.replication.factor": "3",

  "key.converter": "org.apache.kafka.connect.converters.ByteArrayConverter",
  "value.converter": "org.apache.kafka.connect.converters.ByteArrayConverter",
  "header.converter": "org.apache.kafka.connect.converters.ByteArrayConverter"
}
```

注意：

- 不显式设置 `topics.exclude`，继续使用 MM2 默认排除规则：

  ```text
  .*[\-\.]internal, .*\.replica, __.*
  ```

- `IdentityReplicationPolicy`会让普通业务 topic 在 wbbz 上与 wbads 同名。
- heartbeat 是例外：wbads 上的 `heartbeats`复制到 wbbz 后名为 `wbads.heartbeats`，用于观察端到端路径；wbbz 本地正向 Heartbeat Connector 写入的仍是 `heartbeats`。
- 当前是严格单向复制。以后如果开启 `wbbz -> wbads`业务复制，必须重新评估 Identity 策略，否则会形成复制循环。
- `tasks.max=200`只是上限，实际 Task 数不会超过匹配到的源 topic partition 数。

### 10.2 checkpoint.json

```json
{
  "connector.class": "org.apache.kafka.connect.mirror.MirrorCheckpointConnector",
  "tasks.max": "200",

  "source.cluster.alias": "wbads",
  "target.cluster.alias": "wbbz",

  "source.cluster.bootstrap.servers": "10.26.28.41:9111,10.26.28.29:9111,10.26.28.28:9111,10.78.18.47:9111,10.78.18.46:9111",
  "source.cluster.security.protocol": "SASL_PLAINTEXT",
  "source.cluster.sasl.mechanism": "PLAIN",
  "source.cluster.sasl.jaas.config": "org.apache.kafka.common.security.plain.PlainLoginModule required username=\"admin\" password=\"d4a12dfe3f97e641edd9f206eca5ae92\";",

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

  "offset-syncs.topic.location": "source",
  "replication.policy.class": "org.apache.kafka.connect.mirror.IdentityReplicationPolicy",

  "key.converter": "org.apache.kafka.connect.converters.ByteArrayConverter",
  "value.converter": "org.apache.kafka.connect.converters.ByteArrayConverter",
  "header.converter": "org.apache.kafka.connect.converters.ByteArrayConverter"
}
```

重要风险：

- `sync.group.offsets.enabled=true`会写 wbbz 的 `__consumer_offsets`。
- MM2 只会同步当前在 wbbz 没有活跃成员的 consumer group。
- 灾备切换前必须确保同一个 group 不会同时在 wbads 和 wbbz 消费，避免两个站点分别推进 offset。
- 默认 `groups.exclude`仍会排除 `console-consumer-.*`、`connect-.*`和`__.*`。

### 10.3 heartbeat-forward.json

```json
{
  "connector.class": "org.apache.kafka.connect.mirror.MirrorHeartbeatConnector",
  "tasks.max": "1",

  "source.cluster.alias": "wbads",
  "target.cluster.alias": "wbbz",

  "target.cluster.bootstrap.servers": "10.75.12.95:9111,10.75.12.96:9111,10.75.12.97:9111,10.52.140.33:9111,10.52.140.34:9111",
  "target.cluster.security.protocol": "SASL_PLAINTEXT",
  "target.cluster.sasl.mechanism": "PLAIN",
  "target.cluster.sasl.jaas.config": "org.apache.kafka.common.security.plain.PlainLoginModule required username=\"admin\" password=\"d4a12dfe3f97e641edd9f206eca5ae92\";",

  "emit.heartbeats.enabled": "true",
  "emit.heartbeats.interval.seconds": "5",
  "heartbeats.topic.replication.factor": "3",
  "replication.policy.class": "org.apache.kafka.connect.mirror.IdentityReplicationPolicy",

  "key.converter": "org.apache.kafka.connect.converters.ByteArrayConverter",
  "value.converter": "org.apache.kafka.connect.converters.ByteArrayConverter",
  "header.converter": "org.apache.kafka.connect.converters.ByteArrayConverter"
}
```

### 10.4 heartbeat-reverse.json

该 Connector 在同一 Connect 集群中运行，但通过 `producer.override.*`向 wbads 写入心跳。

```json
{
  "connector.class": "org.apache.kafka.connect.mirror.MirrorHeartbeatConnector",
  "tasks.max": "1",

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
  "heartbeats.topic.replication.factor": "3",
  "replication.policy.class": "org.apache.kafka.connect.mirror.IdentityReplicationPolicy",

  "key.converter": "org.apache.kafka.connect.converters.ByteArrayConverter",
  "value.converter": "org.apache.kafka.connect.converters.ByteArrayConverter",
  "header.converter": "org.apache.kafka.connect.converters.ByteArrayConverter"
}
```

## 11. 提交 Connector

`/opt/kafka/config/mm2/apply.sh`：

```bash
#!/usr/bin/env bash
set -euo pipefail

CONNECT=${CONNECT:-http://10.52.139.55:18088}
BASE_DIR=$(cd "$(dirname "$0")" && pwd)

apply() {
  local name=$1
  local file=$2

  echo "Applying $name from $file"
  curl -fsS -X PUT \
    "$CONNECT/connectors/$name/config" \
    -H 'Content-Type: application/json' \
    --data-binary "@$BASE_DIR/$file" |
    jq
}

apply mm2-wbads-to-wbbz-heartbeat heartbeat-forward.json
apply mm2-wbbz-to-wbads-heartbeat heartbeat-reverse.json
apply mm2-wbads-to-wbbz-source source.json
apply mm2-wbads-to-wbbz-checkpoint checkpoint.json
```

执行：

```bash
chmod +x /opt/kafka/config/mm2/apply.sh
/opt/kafka/config/mm2/apply.sh
```

提交前可调用配置校验 API：

```bash
CONNECT=http://10.52.139.55:18088

curl -fsS -X PUT \
  "$CONNECT/connector-plugins/MirrorSourceConnector/config/validate" \
  -H 'Content-Type: application/json' \
  --data-binary @/opt/kafka/config/mm2/source.json |
  jq '.error_count, [.configs[] | select(.value.errors | length > 0)]'
```

## 12. 上线验收

### 12.1 Connector 和 Task 状态

```bash
CONNECT=http://10.52.139.55:18088

curl -fsS "$CONNECT/connectors?expand=status" |
  jq -r 'to_entries[] |
    "\(.key) connector=\(.value.status.connector.state) tasks=\([.value.status.tasks[].state] | join(","))"'
```

所有 Connector 和预期 Task 应为 `RUNNING`。

Checkpoint 在尚未发现符合条件的 consumer group 时可能暂时没有 Task，这不等同于 Connector 失败。

### 12.2 检查内部 topic

```bash
WBADS_BS='10.26.28.41:9111,10.26.28.29:9111,10.26.28.28:9111,10.78.18.47:9111,10.78.18.46:9111'
WBBZ_BS='10.75.12.95:9111,10.75.12.96:9111,10.75.12.97:9111,10.52.140.33:9111,10.52.140.34:9111'

kafka-topics.sh --bootstrap-server "$WBADS_BS" \
  --command-config /opt/kafka/config/wbads-client.properties \
  --list | grep -E 'mm2-offset-syncs\.wbbz\.internal|heartbeats'

kafka-topics.sh --bootstrap-server "$WBBZ_BS" \
  --command-config /opt/kafka/config/wbbz-client.properties \
  --list | grep -E 'wbads\.checkpoints\.internal|heartbeats|connect-mm2'
```

### 12.3 检查业务 topic

由于使用 IdentityReplicationPolicy，目标 topic 与源 topic 同名：

```bash
kafka-topics.sh --bootstrap-server "$WBADS_BS" \
  --command-config /opt/kafka/config/wbads-client.properties \
  --describe --topic <业务topic>

kafka-topics.sh --bootstrap-server "$WBBZ_BS" \
  --command-config /opt/kafka/config/wbbz-client.properties \
  --describe --topic <业务topic>
```

即使 `sync.topic.configs.enabled=false`，MM2 仍会创建目标 topic，并在源 topic 增加 partition 后扩展目标 partition。该开关只控制 retention、cleanup policy 等 topic 配置属性同步。

### 12.4 检查 checkpoint 和 group offset

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

不要使用 Connect Worker 的 `group.id`查询 MM2 业务复制 lag。它只用于 Worker 成员管理。

## 13. 日常运维

### 13.1 修改同步 topic

当前配置：

```text
"topics": ".*"
```

会自动发现所有非默认排除 topic。新增普通业务 topic 后，最长约 60 秒进入复制。

如改为白名单：

```text
"topics": "topic-a,topic-b,order-.*"
```

更新：

```bash
curl -fsS -X PUT \
  http://10.52.139.55:18088/connectors/mm2-wbads-to-wbbz-source/config \
  -H 'Content-Type: application/json' \
  --data-binary @/opt/kafka/config/mm2/source.json |
  jq
```

`topics`中的每一项都是 Java 正则，并使用整串匹配：

- `order`只匹配 topic `order`。
- `order-.*`匹配 `order-a`、`order-2026`。
- JSON 中匹配字面量点号需要写成 `\\.`。

如果显式配置 `topics.exclude`，必须保留默认排除项，例如：

```text
"topics.exclude": ".*[\\-\\.]internal,.*\\.replica,__.*,临时topic-.*"
```

移除 topic 只会停止后续复制，不会删除 wbbz 上已经存在的 topic 和数据。

### 13.2 修改同步 group

编辑 `checkpoint.json`：

```text
"groups": "group-a,group-b,order-service-.*"
```

然后更新 Checkpoint Connector：

```bash
curl -fsS -X PUT \
  http://10.52.139.55:18088/connectors/mm2-wbads-to-wbbz-checkpoint/config \
  -H 'Content-Type: application/json' \
  --data-binary @/opt/kafka/config/mm2/checkpoint.json |
  jq
```

### 13.3 常用 REST 操作

```bash
CONNECT=http://10.52.139.55:18088
NAME=mm2-wbads-to-wbbz-source

curl -fsS "$CONNECT/connectors/$NAME/status" | jq

curl -fsS -X POST \
  "$CONNECT/connectors/$NAME/restart?includeTasks=true&onlyFailed=true" |
  jq

curl -fsS -X PUT "$CONNECT/connectors/$NAME/pause"
curl -fsS -X PUT "$CONNECT/connectors/$NAME/resume"
curl -fsS -X PUT "$CONNECT/connectors/$NAME/stop"

curl -fsS "$CONNECT/connectors/$NAME/offsets" | jq
```

删除 Connector 不等于清除 Source offset。需要重跑时：

1. 先停止 Connector。
2. 使用 offset REST API 删除或修改 offset。
3. 再恢复 Connector。

执行 offset 删除会导致重新复制，必须先评估下游重复数据。

### 13.4 扩缩 Worker

扩容：

- 在新机器安装相同 Kafka 版本和插件。
- 使用相同 `group.id`及 config/offset/status topic。
- 使用新机器自己的 REST listener 和 advertised 地址。
- 启动后 Connect 自动重新分配 Task。

缩容：

- 一次停止一台 Worker。
- 等待 rebalance 完成并确认 Task 全部恢复 RUNNING。
- 再停止下一台。

## 14. 监控

重点监控：

| 对象 | 指标或检查 |
|---|---|
| Connector/Task | REST `/status`中是否存在 `FAILED` |
| 数据复制 | `kafka.connect.mirror`下的 record age、replication latency、record count |
| Source 吞吐 | source record poll/write rate |
| Worker | rebalance、JVM heap、GC pause、线程数 |
| 内部 topic | Connect config/offset/status topic 是否可写 |
| checkpoint | `wbads.checkpoints.internal`是否持续更新 |
| group offset | wbbz 目标 group offset 是否按预期更新 |
| heartbeat | wbads 和 wbbz 的 `heartbeats`是否持续更新 |

复制 lag 应按业务 topic partition 比较 wbads end offset 与 wbbz end offset，或使用 MM2 自身指标。不要把 Connect Worker `group.id`当作源端消费组。

## 15. 独立 Connect 管理集群

Connect 管理 Kafka 可以使用第三个独立集群 M，不要求必须是 wbbz。

此时 Worker 配置：

```properties
bootstrap.servers=<M集群地址>
group.id=mm2-wbads-to-wbbz-connect
config.storage.topic=<M上的config topic>
offset.storage.topic=<M上的offset topic>
status.storage.topic=<M上的status topic>
connector.client.config.override.policy=All
```

由于 Worker 默认 SourceTask producer 会写 M，以下三个 Connector 必须增加：

```text
"producer.override.bootstrap.servers": "10.75.12.95:9111,10.75.12.96:9111,10.75.12.97:9111,10.52.140.33:9111,10.52.140.34:9111",
"producer.override.security.protocol": "SASL_PLAINTEXT",
"producer.override.sasl.mechanism": "PLAIN",
"producer.override.sasl.jaas.config": "org.apache.kafka.common.security.plain.PlainLoginModule required username=\"admin\" password=\"d4a12dfe3f97e641edd9f206eca5ae92\";"
```

适用 Connector：

- `mm2-wbads-to-wbbz-source`
- `mm2-wbads-to-wbbz-checkpoint`
- `mm2-wbads-to-wbbz-heartbeat`

反向 Heartbeat 继续覆盖到 wbads。

普通 at-least-once 模式下，Connector 的 Source offset 可以继续存放在 M 的 Worker 全局 offset topic。M 不可用时，Connect 无法完成配置管理、offset 提交和任务恢复，因此 M 仍是关键依赖。

## 16. Exactly-once

原配置没有真正开启 exactly-once。以下注释不是有效参数：

```properties
#wbbz.exactly.once.wbads.support=enabled
```

如需开启，至少需要：

Worker：

```properties
exactly.once.source.support=enabled
```

Source Connector：

```text
"exactly.once.support": "required",
"source.consumer.isolation.level": "read_committed"
```

如果 Connect 管理 Kafka 与目标 wbbz 分离，还需要让业务记录和 connector 专属 Source offset 位于 wbbz：

```text
"offsets.storage.topic": "connect-mm2-wbads-wbbz-source-offsets",
"producer.override.bootstrap.servers": "<wbbz>",
"consumer.override.bootstrap.servers": "<wbbz>",
"admin.override.bootstrap.servers": "<wbbz>"
```

并为 producer、consumer、admin override 配齐 SASL 参数。

现网集群从 disabled 升级 exactly-once 时，应先将所有 Worker 配为 `preparing`并滚动重启，再改为 `enabled`进行第二轮滚动重启。不要直接在部分 Worker 上启用。

## 17. 从现有专用模式迁移

### 17.1 风险

如果直接创建新的 Connect group、内部 topic 和 Connector 名称，普通 Connect 看不到专用模式原来的 Source offset。由于 MirrorSourceTask 默认从 earliest 开始，没有迁移 offset 时可能全量重复复制。

切换前必须停止所有 `connect-mirror-maker.sh`进程，禁止专用模式和新 Connect Connector 同时向相同目标 topic 写数据。

### 17.2 推荐方案：复用专用模式状态

专用模式本身使用 `DistributedHerder`和 Kafka config/offset/status topic。对于 `wbads -> wbbz`，默认值为：

```properties
group.id=wbads-mm2
config.storage.topic=mm2-configs.wbads.internal
offset.storage.topic=mm2-offsets.wbads.internal
status.storage.topic=mm2-status.wbads.internal
```

这些 topic 位于目标 wbbz。

如要原位接管：

1. 停止全部专用模式节点。
2. 确认 wbbz 上存在上述 topic。
3. 普通 Connect Worker 使用相同 `group.id`和三个内部 topic。
4. Connector 名称保持专用模式名称：
   - `MirrorSourceConnector`
   - `MirrorCheckpointConnector`
   - `MirrorHeartbeatConnector`
5. 启动一个 Worker 验证现有配置和 offset 被正确加载。
6. 确认没有回放后再扩至多个 Worker。
7. 通过 REST 更新原有 Connector 配置。

不要在复用旧 config topic 时同时创建本文的新 Connector 名称，否则会产生两套 Source Connector 并双写。

如果专用模式未使用 `--clusters wbbz`限制目标，wbads 上还可能存在反向 herder：

```text
group.id=wbbz-mm2
mm2-configs.wbbz.internal
mm2-offsets.wbbz.internal
mm2-status.wbbz.internal
```

其中通常只有反向 Heartbeat 有活动 Task。可以：

- 在 wbads 侧启动第二套普通 Connect Worker 复用这些状态；或者
- 不复用反向 herder，改为本文的 `heartbeat-reverse.json`。

两种方式只能选一种，不能同时运行。

### 17.3 新建 Connect 集群

如果选择本文第 8 节的新 group 和新内部 topic：

- 必须接受从 earliest 重新复制；或者
- 在停机窗口使用 Connect offset API 设置每个源 topic partition 的起始 offset。

在没有完成 offset 验证前，不要删除旧专用模式内部 topic。

## 18. 故障速查

| 现象 | 可能原因 | 处理 |
|---|---|---|
| REST 请求在 Worker 间转发失败 | advertised 地址不可达 | 修正 `rest.advertised.*`并重启 Worker |
| Source Task FAILED | wbads 认证、权限或网络错误 | 查看 `/status`中的 trace |
| Connector RUNNING 但无数据 | `topics`正则未匹配，或源 topic 无新数据 | 检查配置及源端 end offset |
| wbbz 没有目标 topic | target AdminClient 权限不足 | 检查 `target.cluster.*`认证和 CREATE 权限 |
| wbads 无 offset-sync topic | source 端无 CREATE/WRITE 权限 | 授权或改为 `offset-syncs.topic.location=target` |
| checkpoint 无数据 | 没有符合条件的 group，或 offset-sync 尚未形成 | 检查 groups、业务消费进度及 offset-sync |
| wbbz group offset 不更新 | wbbz 上该 group 有活跃成员 | 停止目标消费者后等待下一同步周期 |
| 数据重复 | 重建了 Connector 名称、清空 offset、迁移未复用旧 offset，或有两套 Source 同时运行 | 停止重复链路并核对 Source offset |
| 目标 topic partition 少 | refresh 尚未执行或 target ALTER 权限不足 | 检查 refresh 和目标 Admin 权限 |
| 反向心跳写到 wbbz 而不是 wbads | 缺少 `producer.override.bootstrap.servers` | 修正 reverse heartbeat 配置 |
| Connect 内部 topic 不可用 | wbbz 故障或 Worker 安全配置错误 | 检查 Worker 顶层及 producer/consumer/admin 配置 |

## 19. 上线检查清单

- [ ] 所有 Worker 使用相同 `group.id`和内部 topic。
- [ ] 每台 Worker 使用自己的可路由 REST advertised 地址。
- [ ] Connect config topic 只有一个 partition。
- [ ] 三个 Connect 内部 topic 均为 compact。
- [ ] Worker key/value/header converter 均为 ByteArrayConverter。
- [ ] Source、Checkpoint 和正向 Heartbeat 写入 wbbz。
- [ ] 反向 Heartbeat 通过 producer override 写入 wbads。
- [ ] `mm2-offset-syncs.wbbz.internal`位于 wbads。
- [ ] `wbads.checkpoints.internal`位于 wbbz。
- [ ] 业务 topic 在 wbbz 保持原名。
- [ ] `sync.group.offsets.enabled=true`的风险已由业务确认。
- [ ] 同一个 consumer group 不会同时在 wbads 和 wbbz 活跃消费。
- [ ] 已确认专用模式进程全部停止，不存在双写。
- [ ] 已确认迁移方案是否复用旧 Source offset。
- [ ] 已配置 Connector/Task、复制延迟、JVM 和内部 topic 告警。
