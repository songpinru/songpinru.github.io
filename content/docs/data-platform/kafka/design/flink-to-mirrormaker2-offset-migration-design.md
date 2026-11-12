---
title: "Flink 同步程序迁移到 MirrorMaker 2 设计"
description: "将历史 Flink 集群间复制作业迁移到 MirrorMaker 2，并以 Connect source offset 接管 Flink consumer group 位点的设计方案。"
---

**状态**：设计中

**更新时间**：2026-10-15

## 1. 目标与结论

历史 Flink 作业仅承担 Kafka 集群间的数据复制，计划使用 MirrorMaker 2（MM2）替代。迁移目标是：

1. MM2 从 Flink 已完成同步的位置继续读取源 Kafka，避免数据遗漏；
2. 源、目标 Kafka 的 offset **不要求一致**，目标端 offset 不参与迁移计算；
3. 只复制业务数据，仅部署 `MirrorSourceConnector`；
4. MM2 运行在 Kubernetes 上，通过统一的配置、发布、日志和监控入口维护，不再登录分散的主机操作。

核心结论：Flink consumer group 中已提交的 offset 表示下一条待读取记录，记为 `C`；普通模式下 MM2/Connect 保存的是上一条已处理的源 offset，记为 `M`。因此迁移关系为：

```text
C > 0 时：M = C - 1
MM2 启动后：seek(M + 1) = seek(C)
```

例如 Flink 对 `orders-0` 提交了 `C=1000`，则向 Connect 导入 `M=999`，MM2 将从源 offset 1000 继续复制。

迁移的是**源端读取进度**，不是源、目标 offset 的数值映射。`offset.consistency.enabled` 保持 `false`，不使用 MM2 offset-sync、checkpoint 或 consumer group offset 同步能力。

## 2. 设计范围

### 2.1 目标架构

```text
源 Kafka
   │
   │ 业务数据
   ▼
Kubernetes 中的 distributed Kafka Connect（`docker/mm2` 镜像）
   └── MirrorSourceConnector
             │
             ▼
          目标 Kafka

Connect 配置、状态和 source offset ──► 目标 Kafka 中的 Connect 内部 topic
运维操作 ──► GitOps/CI-CD + Kubernetes Job + Connect REST API
日志/指标 ──► 集中日志平台 + Prometheus/Grafana
```

### 2.2 本次包含

- 将 Flink 最终消费位置转换成 Connect source offset；
- 使用 distributed Kafka Connect 运行 `MirrorSourceConnector`；
- 在 Kubernetes 上部署、发布、监控和集中维护；
- 切换前校验、切换后验证及回滚准备。

### 2.3 本次不包含

- 不要求源、目标 Kafka offset 一致；
- 不部署 `MirrorCheckpointConnector` 和 `MirrorHeartbeatConnector`；
- 不同步业务 consumer group offset、topic 配置或 ACL；
- 不生成源、目标 offset 映射，关闭 `emit.offset-syncs.enabled`；
- 不迁移 Flink 算子状态、窗口状态或业务处理逻辑；
- 不承诺 Flink 与 MM2 之间端到端 exactly-once。切换优先保证不丢数据，边界处少量重复需要业务可接受或可去重。

如果原 Flink 作业包含过滤、转换、聚合、重分区等逻辑，则不能直接由 MM2 替代，必须先确认这些逻辑已经不再需要。

## 3. Offset 迁移设计

### 3.1 Offset 含义

| 位置 | 含义 | 本次用途 |
|---|---|---|
| Flink checkpoint/savepoint | Flink 作业的恢复状态 | 确认最终状态和回滚依据 |
| 源 Kafka consumer group offset `C` | 下一条待读取的源 offset | 迁移输入 |
| Connect source offset `M` | MM2 上一条已处理的源 offset | 迁移写入目标 |
| 目标 Kafka offset | 目标分区内的记录位置 | 仅用于数据验证，不参与换算 |

Connect 中 MM2 的 source partition 结构为：

```json
{"cluster":"source","topic":"orders","partition":0}
```

其中 `cluster` 必须等于 `source.cluster.alias`，`topic` 和 `partition` 都是源 Kafka 的信息。Flink group ID 不进入这个 key。

### 3.2 最终迁移边界

Kafka consumer group offset 只有在能够代表 Flink 最终成功输出时才能直接使用。切换时必须满足：

1. Flink 已优雅停止，并生成最终成功的 checkpoint/savepoint；
2. Flink Sink 缓冲已刷出，相关事务已经提交，目标端数据可见；
3. 已关闭 Flink 自动恢复，确认没有旧实例继续写目标 Kafka；
4. 停止后导出的 group offset 保持稳定，并与最终 checkpoint/savepoint 的源进度一致；
5. 源 Kafka 中从 `C` 开始的数据仍在保留期内。

如果 group offset 落后于最终 checkpoint，可以从更早位置重放，但需要接受重复；如果 group offset 超前、来源不明，或目标端提交状态无法确认，则停止迁移，不能直接导入。

### 3.3 转换规则

| Flink next offset `C` | Connect 写入 | 处理方式 |
|---|---|---|
| `C > 0` | `{"offset": C-1}` | MM2 从 `C` 开始读取 |
| `C = 0` | 清除该分区已有 Connect offset | 配合 `auto.offset.reset=earliest` 从头读取 |
| offset 缺失 | 不写入 | 默认阻断迁移，不能静默使用 latest |
| `C` 小于源 log start offset | 不写入 | 所需源数据已经过期，需要补数或明确接受缺失 |
| `C` 大于源 log end offset | 不写入 | 位点无效，需要重新核查 |

每个迁移分区都必须满足：

```text
source log start offset <= C <= source log end offset
```

导入前冻结 topic 和 partition 变更，并保存以下清单：

- source cluster ID、topic、partition；
- Flink group ID 和原始 `C`；
- 转换后的 `M`；
- 源 log start/end offset；
- connector name、source alias 和配置摘要。

### 3.4 通过 Connect REST API 导入

禁止直接向 `offset.storage.topic` 写入手工 JSON。统一使用 Connect REST API，让 Connect 负责 namespace、序列化和实际 offset 存储。

先以 `STOPPED` 状态创建 connector：

```http
POST /connectors
Content-Type: application/json

{
  "name": "flink-to-mm2-source",
  "initial_state": "STOPPED",
  "config": {
    "connector.class": "org.apache.kafka.connect.mirror.MirrorSourceConnector",
    "tasks.max": "4",
    "source.cluster.alias": "source",
    "target.cluster.alias": "target",
    "source.cluster.bootstrap.servers": "SOURCE_BOOTSTRAP",
    "target.cluster.bootstrap.servers": "TARGET_BOOTSTRAP",
    "topics": "^(orders|payments)$",
    "replication.policy.class": "org.apache.kafka.connect.mirror.IdentityReplicationPolicy",
    "offset.consistency.enabled": "false",
    "emit.offset-syncs.enabled": "false",
    "refresh.topics.enabled": "true",
    "refresh.topics.interval.seconds": "60",
    "sync.topic.configs.enabled": "false",
    "sync.topic.acls.enabled": "false",
    "source.consumer.auto.offset.reset": "earliest",
    "source.consumer.isolation.level": "read_committed",
    "key.converter": "org.apache.kafka.connect.converters.ByteArrayConverter",
    "value.converter": "org.apache.kafka.connect.converters.ByteArrayConverter",
    "header.converter": "org.apache.kafka.connect.converters.ByteArrayConverter"
  }
}
```

`topics`、replication policy 和读隔离级别必须按原 Flink 作业的实际行为确认。如果历史数据写入目标同名 topic，可使用经过验证的 `IdentityReplicationPolicy`；目标 topic 及 partition 应提前创建并检查。

Connector 处于 `STOPPED` 后，写入转换后的 offset：

```http
PATCH /connectors/flink-to-mm2-source/offsets
Content-Type: application/json

{
  "offsets": [
    {
      "partition": {"cluster":"source","topic":"orders","partition":0},
      "offset": {"offset":999}
    },
    {
      "partition": {"cluster":"source","topic":"orders","partition":1},
      "offset": null
    }
  ]
}
```

导入后必须调用以下接口全量回读，确认每个分区与迁移计划一致：

```http
GET /connectors/flink-to-mm2-source/offsets
```

只有 100% 校验通过后才能启动：

```http
PUT /connectors/flink-to-mm2-source/resume
```

`PAUSED` 不等于 `STOPPED`。修改 offset 前必须确认 connector 和所有 task 已停止。Connector 一旦运行，不得再次执行旧迁移计划，否则会将读取位置回退。

## 4. MirrorSourceConnector 配置原则

| 配置项 | 建议值 | 说明 |
|---|---|---|
| `offset.consistency.enabled` | `false` | 普通复制模式，不要求两端 offset 一致 |
| `emit.offset-syncs.enabled` | `false` | 不生成源、目标 offset 映射 |
| `sync.topic.configs.enabled` | `false` | 目标 topic 配置独立管理 |
| `sync.topic.acls.enabled` | `false` | ACL 由平台统一管理 |
| `refresh.topics.enabled` | `true` | 周期感知源端新增 topic/partition，并重新分配 task |
| `refresh.topics.interval.seconds` | `60` | 控制上游变化的感知时延 |
| `topics` | 明确的业务 allowlist | 不使用过宽的正则 |
| `replication.policy.class` | 按历史目标 topic 命名确认 | 必须继续写入正确的目标 topic |
| `source.consumer.isolation.level` | 与 Flink 一致 | 避免读取可见性发生变化 |

还需要确认：

- key、value、header、timestamp 和 tombstone 的兼容性；
- 目标 topic 必须提前创建并冻结 partition count；
- 目标端只有一个复制写入者，不能让 Flink 与 MM2 长时间并行写入同一数据；
- Worker 的业务数据 producer 实际连接目标 Kafka，不能只依赖 connector 中的 `target.cluster.bootstrap.servers`。

`MirrorCheckpointConnector` 不进入本次部署范围，本方案不增加 checkpoint 运行时保护。

## 5. Kubernetes 部署设计

### 5.1 是否可以按无状态服务部署

**结论：worker Pod 可以被替换且通常不需要 PVC，但不能完全当作普通无状态 HTTP 微服务。**

Connect 的持久状态保存在 Kafka，Pod 本地没有必须保留的业务状态，因此可以使用 Kubernetes 重建、漂移和滚动发布。但是 connector/task 是有归属的长运行任务，Pod 增减会触发 Connect 集群 rebalance；Pod 异常退出后，新 task 会从最近一次已提交的 Connect source offset 恢复，普通 source 模式可能重放少量数据。

所以应区分：

- **存储层面无状态**：Pod 可替换、无需共享本地磁盘；
- **运行层面有成员和任务归属**：不能随意频繁扩缩容、杀 Pod 或缩容到 0，也不能只检查 HTTP 端口就认为复制健康。

### 5.2 Connect worker 之间的通信

源码确认 distributed Connect 集群存在两类内部通信：

| 通信路径 | 作用 |
|---|---|
| worker ↔ Kafka group coordinator | 使用相同 `group.id` 加入 worker group、发送 heartbeat、选举 leader 并分配 connector/task |
| worker ↔ Connect 内部 topic | 读取 connector 配置和状态，提交/恢复 source offset |
| worker → worker REST | 非 leader 收到管理请求后转发给 leader/任务 owner；connector 动态生成 task 配置时也可能转发到 leader |

worker 间 REST 属于控制面通信；业务数据不会在 Pod 之间转发。运行 `MirrorSourceTask` 的 worker 直接从源 Kafka 读取，并由该 worker 的 source producer 直接写入目标 Kafka。

每个 worker 会把自己的 REST URL 放进 group metadata，leader 根据这些 URL 定位 connector/task owner。管理请求可以先发给任意 worker，但接收请求的 worker 可能继续转发。因此 Kubernetes 上必须满足：

1. 每个 Pod 都有**唯一且其他 Pod 可直接访问**的 advertised REST 地址；
2. 不要让所有 worker 都把同一个负载均衡 `Service` 地址作为 `rest.advertised.host.name`，否则转发可能再次随机落到错误 worker；
3. `NetworkPolicy` 必须允许 Connect Pod 之间访问 REST 端口；
4. 如果使用 HTTPS，worker 间客户端也必须信任 advertised 地址对应的证书；
5. 所有 Pod 都必须能访问源 Kafka、目标 Kafka 和 Connect 内部 topic，因为 connector/task 可能迁移到任意 worker。

对外管理仍可以使用一个 ClusterIP Service，因为任意 worker 都能接收并转发管理请求；这个 Service 与每个 worker 对内公布的唯一地址用途不同。

### 5.3 仓库内 MM2 镜像

Kubernetes 部署统一使用仓库中的 `docker/mm2` 镜像实现，不再另行维护主机安装包或在 Pod 启动时下载插件。

| 文件 | 作用 |
|---|---|
| `docker/mm2/Dockerfile` | 从指定 `KAFKA_URL` 构建 Java 17 镜像，保留完整 Kafka `libs`，使用 UID/GID `10001` 非 root 运行 |
| `docker/mm2/connect-mm2.properties` | distributed worker、内部 topic、REST advertised 地址、认证和 source producer 默认配置 |
| `docker/mm2/docker-entrypoint.sh` | 校验端口、按需加载 JMX exporter，并使用 `exec` 启动 Connect 以正确接收 `SIGTERM` |
| `docker/mm2/connect-log4j.properties` | 日志只输出 stdout，由 Kubernetes 日志系统集中收集 |
| `docker/mm2/jmx-exporter.yaml` | 可选 Prometheus JMX exporter 配置 |
| `docker/mm2/README.md` | 镜像构建、环境变量和 Pod IP 配置说明 |

镜像约定：

- 镜像只启动 Kafka Connect distributed worker，不自动创建任何 connector；`MirrorSourceConnector` 仍通过 REST API 以 `STOPPED` 状态创建；
- 默认 REST 端口为 `8088`，与 Dockerfile `EXPOSE 8088`、Kubernetes Service 和探针保持一致；
- worker 配置使用 `EnvVarConfigProvider` 在进程内解析 `${env:...}`，不会把包含凭据的新 properties 文件写到容器文件系统；
- Kafka/Connect/MM2 版本由构建参数 `KAFKA_URL` 决定。生产构建必须固定制品版本，发布后使用不可变镜像 tag/digest，所有 worker 使用同一镜像；
- Prometheus exporter 默认关闭；设置 `JMX_EXPORTER_PORT` 后才以 Java Agent 方式启用。如果 Kafka 制品不包含对应 agent，入口脚本会拒绝启动；
- 当前 worker 配置明确设置 `exactly.once.source.support=disabled`，与本方案普通 source 模式一致。

### 5.4 推荐 Kubernetes 拓扑

采用专用的 distributed Kafka Connect 集群，初始建议 3 个 worker Pod：

| Kubernetes 资源 | 用途 |
|---|---|
| `Deployment` | 使用 `docker/mm2` 镜像运行 worker，统一滚动发布和扩缩容 |
| `ConfigMap` | 注入非敏感环境变量，必要时覆盖 JMX 配置 |
| `Secret` | 注入 `CONNECT_SASL_JAAS_CONFIG` 等敏感配置 |
| `Service` | 暴露统一管理入口 `8088`，不作为所有 worker 共用的 advertised 地址 |
| `Job` | 执行一次性的 offset `export/plan/apply/verify` 和 connector 发布操作 |
| `PodDisruptionBudget` | 避免维护时同时驱逐过多 worker |
| `NetworkPolicy` | 放行 worker 间 REST、Kafka、DNS、管理 Job 和监控所需流量 |
| `ServiceMonitor` | 在启用 `JMX_EXPORTER_PORT` 后采集 Connect/JVM/MM2 指标 |

镜像需要以下运行参数：

```text
CONNECT_BOOTSTRAP_SERVERS
CONNECT_GROUP_ID
CONNECT_CONFIG_STORAGE_TOPIC
CONNECT_OFFSET_STORAGE_TOPIC
CONNECT_STATUS_STORAGE_TOPIC
CONNECT_SASL_JAAS_CONFIG
POD_IP
```

其中非敏感变量通过 ConfigMap 注入，`CONNECT_SASL_JAAS_CONFIG` 通过 Secret 注入，`POD_IP` 必须使用 Downward API：

```yaml
env:
  - name: POD_IP
    valueFrom:
      fieldRef:
        fieldPath: status.podIP
  - name: CONNECT_SASL_JAAS_CONFIG
    valueFrom:
      secretKeyRef:
        name: mm2-connect-auth
        key: sasl-jaas-config
ports:
  - name: rest
    containerPort: 8088
  - name: metrics
    containerPort: 18088
```

`metrics` 端口只有在设置 `JMX_EXPORTER_PORT=18088` 时才会实际监听。Service、ServiceMonitor 和 NetworkPolicy 必须使用同一组端口。

镜像已经直接使用环境配置生成每个 worker 的 advertised 地址，不需要启动脚本二次渲染配置文件：

```properties
listeners=http://0.0.0.0:${env:CONNECT_REST_PORT}
rest.advertised.host.name=${env:POD_IP}
rest.advertised.port=${env:CONNECT_REST_PORT}
rest.advertised.listener=HTTP
```

如果 HTTPS 证书不能覆盖 Pod IP，或希望使用稳定 DNS，需要扩展当前镜像配置，并可改用无 PVC 的 `StatefulSet + Headless Service`。稳定 DNS 有利于寻址，但不会消除 worker group rebalance。

### 5.5 Worker 状态、安全与统一配置

同一 Connect 集群的所有 Pod 必须使用相同且长期稳定的以下环境变量：

```text
CONNECT_BOOTSTRAP_SERVERS=TARGET_BOOTSTRAP
CONNECT_GROUP_ID=flink-mm2-connect
CONNECT_CONFIG_STORAGE_TOPIC=flink-mm2-connect-configs
CONNECT_OFFSET_STORAGE_TOPIC=flink-mm2-connect-offsets
CONNECT_STATUS_STORAGE_TOPIC=flink-mm2-connect-status
```

要求：

- Connect 配置、状态和 source offset 保存在 Kafka，worker Pod 不需要 PVC；
- 内部 topic 提前创建并启用 compact，副本数与镜像中的 `replication.factor=3` 一致；`config.storage.topic` 使用单分区；
- 内部 topic 名称、`group.id` 和管理 Kafka 地址不随 Pod 名称或发布批次变化；
- 镜像保留完整 Kafka `libs`，所有 Pod 使用相同镜像 digest，不在运行时安装或替换 JAR；
- 3 个 worker 是可用性配置，不代表一定有 3 份复制任务；实际并行度受源 partition 数和 `tasks.max` 限制；
- `CONNECT_BOOTSTRAP_SERVERS` 指向目标 Kafka，确保 Connect 内部 topic 和 source producer 都落到预期目标集群；
- 当前镜像默认使用 `SASL_PLAINTEXT + PLAIN`。若生产环境要求 TLS/SASL_SSL，必须通过经过评审的配置文件覆盖或扩展镜像，不能只改 Kubernetes Service；
- 镜像设置 `connector.client.config.override.policy=All`，允许 connector 覆盖 producer/consumer/admin 客户端参数。Connect REST 必须限制访问，connector JSON 必须版本化和评审，避免借此注入任意客户端配置；
- 推荐设置 `runAsNonRoot: true`、`runAsUser: 10001`、`runAsGroup: 10001`，并在环境验证后启用只读根文件系统。

### 5.6 Pod 生命周期与滚动发布

Connect 不支持 worker static membership，Pod 加入、退出或更换都会触发 group rebalance，需要特别处理：

- 使用 pod anti-affinity 或 topology spread 将 worker 分散到不同节点/可用区，并配置 PDB；
- 滚动发布一次只替换一个 worker，避免多个 worker 同时退出；
- 不使用基于 CPU 的激进 HPA，不缩容到 0；扩缩容应作为受控变更，并观察 rebalance 和 task 恢复；
- 镜像设置 `scheduled.rebalance.max.delay.ms=300000`。worker 离开后，task 可能等待该延迟再重新分配，应根据“滚动升级稳定性”和“故障恢复时长”目标调优并实测；
- 镜像设置 `offset.flush.interval.ms=10000`、`offset.flush.timeout.ms=30000`，而 `task.shutdown.graceful.timeout.ms` 仍使用框架默认 5000。`terminationGracePeriodSeconds` 必须覆盖 task 停止、producer flush、offset 提交及余量；
- 镜像入口脚本最终使用 `exec connect-distributed.sh`，可以把 Kubernetes 的 `SIGTERM` 传给 Connect；部署不能再包一层不转发信号的 shell；
- 普通 source 模式在进程崩溃时会从最后成功提交的 source offset 恢复，未提交窗口内的数据可能重复。

探针建议：

- liveness 只检查 worker 进程/基础 REST 是否存活，不要因 connector FAILED、lag 上升或短暂 rebalance 重启 Pod；
- readiness 控制管理 Service 是否向该 worker 转发流量，但不能代替 connector/task 健康检查；
- connector/task 状态、offset 是否推进和复制 lag 使用独立监控告警。

### 5.7 集中管理方式

为避免后期分散登录维护，统一采用以下管理边界：

1. **基础设施配置 GitOps 化**：Deployment/StatefulSet、Service、ConfigMap、Secret 引用、PDB、NetworkPolicy 和监控规则全部进入版本库，通过 CI/CD 发布；
2. **Connector 配置版本化**：connector JSON 保存在配置仓库，由受控 Job 或流水线调用 Connect REST API，不在 Pod 内手工修改；
3. **REST 只在集群内开放**：通过 ClusterIP Service 暴露，必要操作由 Kubernetes Job、受控运维入口或临时 port-forward 完成，不在每台机器开放管理端口；
4. **日志和指标集中化**：日志输出 stdout，由集群日志系统收集；通过 JMX Exporter/Prometheus 统一采集，不登录 Pod 查长期日志；
5. **单一生命周期控制方**：如果使用 Operator/CRD，connector 的 STOPPED/RUNNING 状态必须由同一个控制器管理，不能同时由 Operator 和手工 REST 相互覆盖。

Offset 导入 Job 必须是一次性任务，不能放在每个 Pod 的 `initContainer` 中。Job 默认只执行 `apply + verify`，不自动 `resume`；人工审批后由唯一管理入口启动 connector，避免 Job 重试时重新导入旧 offset。

### 5.8 监控与告警

至少监控：

- connector/task 状态，FAILED 或长时间无 task 时告警；
- 源端消费 lag、复制吞吐和积压是否持续下降；
- Connect source offset 是否持续推进；
- worker group rebalance 次数、耗时和 task 未分配时长；
- worker 间 REST 转发失败、leader 不可达和 HTTP 409/5xx；
- producer/consumer error、认证、超时、消息过大和序列化错误；
- Pod 重启次数、终止耗时以及 JVM 内存/GC；
- 切换后的目标数据量、边界记录和业务消费是否正常。

Kubernetes `Ready` 只代表 worker 服务可用，不代表 connector 正常复制；必须同时检查 connector/task 和复制指标。

## 6. 上线步骤

1. **预检查**：确认 Flink 只有复制逻辑；核对 topic 映射、partition、读隔离、目标权限和源数据保留时间；
2. **构建并部署 worker**：使用固定 `KAFKA_URL` 构建、发布 `docker/mm2` 镜像并锁定 digest；创建 Connect Deployment、Service、内部 topic、Secret 和监控，暂不启动 source connector；
3. **预演**：在测试环境验证 `C → C-1`、`C=0`、Pod 重建续传及 REST 导入流程；
4. **冻结变更**：冻结迁移 topic 的新增、删除、重建和扩分区；暂停 Connect 扩缩容及滚动发布；
5. **停止 Flink**：优雅停止，取得最终 checkpoint/savepoint，等待 Sink 提交完成并禁用自动拉起；
6. **导出位点**：导出全部迁移分区的 `C`，校验范围、分区全集及目标提交状态；
7. **创建 STOPPED connector**：确认 task 未运行，备份 connector 已有 offset（如果存在）；
8. **转换和导入**：生成 `M=C-1` 的完整计划，通过 REST PATCH 导入；
9. **回读校验**：逐分区 GET offsets，校验 connector name、alias、topic、partition 和 offset；
10. **启动 MM2**：人工审批后 resume，观察实际首批记录、lag 和目标数据；
11. **稳定性验证**：积压追平后进行一次受控滚动重启，确认能够从 Connect offset 正确续传；
12. **收尾**：进入稳定运行后恢复常规发布策略；保留 Flink savepoint、迁移清单和审计记录至回滚窗口结束。

## 7. 验收与回滚

### 7.1 验收标准

- 所有计划分区均成功导入并全量回读一致；
- MM2 从每个分区的 `C` 或该位置后的第一条可见记录开始读取；
- 没有发现迁移边界后的数据遗漏；
- connector/task 持续 RUNNING，source offset 持续推进，lag 最终收敛；
- `docker/mm2` 镜像以非 root 用户运行，REST advertised Pod IP、stdout 日志和可选 JMX exporter 符合部署配置；
- Pod 重建或滚动发布后能够继续复制，不依赖本地文件和人工登录；
- 未创建 Checkpoint/Heartbeat connector，未产生 offset-sync、group、config 或 ACL 同步行为；
- 目标 topic、partition 和消息格式符合原同步链路要求。

### 7.2 回滚策略

**MM2 启动前**：可以直接停止/删除尚未运行的 connector，恢复 Flink 最终 savepoint，不会产生新的目标写入。

**MM2 启动后**：先 STOPPED MM2 并确认 task 停止，再决定恢复方式：

- 可接受重复：从 Flink 最终 savepoint 恢复，边界后的数据可能重复；
- 不可接受重复：根据 MM2 已处理的最新源 offset 重新生成 Flink 恢复边界后再启动，不能直接使用旧 savepoint；
- 任何情况下都不能让 Flink 和 MM2 同时长期写入同一目标数据。

在回滚窗口结束前，不删除 Flink 最终 savepoint、原 group offset 快照、Connect offset 备份和迁移审计记录。

## 8. 实施前待确认项

- Flink/Kafka connector 版本、最终 checkpoint 与 group commit 的对应关系；
- Flink Sink 的交付语义及最终 flush/事务提交证据；
- 源/目标集群地址、cluster ID、topic/partition 清单；
- 目标 topic 命名是否需要 `IdentityReplicationPolicy`；
- `docker/mm2` 生产构建使用的 Kafka 制品版本/URL、最终镜像 tag/digest、资源规格、worker 数量和内部 topic 副本数；
- Kubernetes namespace、Secret/config provider、NetworkPolicy、日志和监控接入方式；
- 可接受的切换停顿、重复范围、源 retention 和回滚窗口。

## 9. 源码依据

- `MirrorSourceTask.loadOffset/initializeConsumer`：普通模式从 Connect source offset `M+1` 恢复；
- `MirrorSourceTask.convertRecord`：Connect source offset 保存源记录 offset；
- `MirrorUtils.wrapPartition/unwrapOffset`：source partition 由 cluster/topic/partition 组成；
- `MirrorSourceConnector.alterOffsets`：校验 REST 导入的 source offset；
- `ConnectorsResource.getOffsets/alterConnectorOffsets`：Connect REST offset 查询和修改入口；
- `DistributedHerder`：distributed 模式下修改 offset 前要求 connector 处于 STOPPED；非 leader 会通过 REST 向 leader 提交 task 配置；
- `WorkerCoordinator`、`IncrementalCooperativeConnectProtocol`：worker 使用 Kafka group 协议选主、分配任务，并在成员 metadata 中携带各自 REST URL；
- `HerderRequestHandler`、`RestClient`：管理请求会在 worker 之间转发；
- `RestServer.advertisedUrl`：`rest.advertised.*` 决定其他 worker 使用的直连地址；
- `KafkaConfigBackingStore`、`KafkaOffsetBackingStore`、`KafkaStatusBackingStore`：配置、source offset 和状态持久化到 Kafka；
- `WorkerConfig`：task 优雅停止和 source offset flush 的超时/周期配置；
- `docker/mm2/Dockerfile`、`docker/mm2/connect-mm2.properties`、`docker/mm2/docker-entrypoint.sh`：本方案实际镜像、worker 默认配置、REST advertised 地址和容器生命周期实现。
