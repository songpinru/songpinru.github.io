---
title: "Kafka Connect / MirrorMaker 2 REST API 查询指南"
description: "Kafka Connect Distributed 与 MirrorMaker 2 的 Connector、Task、Worker、Offset、Plugin 和日志管理 REST API"
---

## 1. 说明

核心 REST 资源定义在以下类中：

- `RootResource`
- `ConnectorsResource`
- `ConnectorPluginsResource`
- `LoggingResource`
- `InternalClusterResource`

完整 OpenAPI 定义见：

- `docs/generated/connect_rest.yaml`

通用约定：

- 默认 REST 端口为 `8083`，实际以 `listeners` 配置为准。
- 接口通常使用 `Content-Type: application/json`。
- `{connector}` 表示 connector 名称。
- `{task}` 表示整数类型的 task ID。
- `?forward=` 是 Connect worker 内部转发参数，普通用户通常不需要指定。
- 配置了 `rest.extension.classes` 时，第三方扩展可能注册额外接口，本清单只包含 Kafka Connect 内建接口。

## 2. Worker 基础接口

| 方法 | 路径 | 作用 | 说明 |
|---|---|---|---|
| GET | `/` | 查询当前 Connect worker 信息 | 返回 `version`、`commit`、`kafka_cluster_id` |
| GET | `/health` | Worker 健康检查 | 健康返回 200；启动中返回 503；异常返回 500 |

## 3. Connector Plugin 接口

| 方法 | 路径 | 作用 | 说明 |
|---|---|---|---|
| GET | `/connector-plugins` | 列出已安装插件 | 默认 `connectorsOnly=true`，只返回 Source/Sink Connector；设置 `connectorsOnly=false` 也会返回 Transform、Predicate、Converter |
| GET | `/connector-plugins/{pluginName}/config` | 查询插件配置定义 | 返回 `ConfigKeyInfo` 列表 |
| PUT | `/connector-plugins/{pluginName}/config/validate` | 校验 connector 配置 | 请求体是完整配置 Map，返回 `ConfigInfos` 和逐项校验结果 |

注意：`GET /connector-plugins` 查询的是处理该请求的 worker，不是整个集群的统一视图。滚动升级或各 worker 插件不一致时，结果可能不同。

## 4. Connector 管理接口

| 方法 | 路径 | 作用 | 请求/响应及说明 |
|---|---|---|---|
| GET | `/connectors` | 列出 connector | 默认返回 connector 名称数组；支持 `?expand=status&expand=info`，返回带状态和详情的 Map |
| POST | `/connectors` | 创建 connector | 成功返回 201；请求体包含 `name`、`config`、可选 `initial_state` |
| GET | `/connectors/{connector}` | 查询 connector 详情 | 返回名称、配置、task ID 列表、connector 类型 |
| DELETE | `/connectors/{connector}` | 删除 connector | 停止并删除 connector/task 配置，成功通常返回 204 |
| GET | `/connectors/{connector}/config` | 查询 connector 配置 | 返回配置 Map |
| PUT | `/connectors/{connector}/config` | 创建或完整替换配置 | 已存在返回 200；新建返回 201 |
| PATCH | `/connectors/{connector}/config` | 局部更新配置 | 请求体中值为 `null` 表示删除该配置项，成功返回 200 |
| GET | `/connectors/{connector}/status` | 查询 connector 和所有 task 状态 | 返回 connector 状态、worker ID、异常信息及全部 task 状态 |

### 4.1 创建 connector

```http
POST /connectors
Content-Type: application/json
```

```json
{
  "name": "flink-to-mm2-source",
  "config": {
    "connector.class": "org.apache.kafka.connect.mirror.MirrorSourceConnector",
    "tasks.max": "4"
  },
  "initial_state": "STOPPED"
}
```

`initial_state` 可选值：

- `RUNNING`：默认值
- `PAUSED`
- `STOPPED`

### 4.2 查看 connector 列表

```http
GET /connectors
```

返回 connector 名称数组。

```http
GET /connectors?expand=status&expand=info
```

返回带状态和详情的 Map，适合一次查询所有 connector 的运行状态。

## 5. Connector 生命周期接口

| 方法 | 路径 | 作用 | 说明 |
|---|---|---|---|
| PUT | `/connectors/{connector}/stop` | 停止 connector | 设置目标状态为 `STOPPED`，关闭 task 并释放 Kafka Client、SMT 等资源；幂等 |
| PUT | `/connectors/{connector}/pause` | 暂停 connector | task 停止处理数据，但继续持有资源；目标状态变为 `PAUSED`；返回 202 |
| PUT | `/connectors/{connector}/resume` | 恢复 connector | 将 `PAUSED` 或 `STOPPED` 状态恢复为 `STARTED`；返回 202 |
| POST | `/connectors/{connector}/restart` | 重启 connector/task | 支持 `includeTasks`、`onlyFailed` 两个查询参数 |

### 5.1 Restart 参数

| 参数 | 默认值 | 含义 |
|---|---:|---|
| `includeTasks` | `false` | 是否同时重启 task |
| `onlyFailed` | `false` | 是否只重启 `FAILED` 的实例 |

行为：

- 默认 `includeTasks=false&onlyFailed=false`：只重启 connector，返回 204。
- 指定 `includeTasks=true` 或 `onlyFailed=true`：返回 202 和 connector 状态信息。
- 单个 task 重启使用 `/connectors/{connector}/tasks/{task}/restart`。

### 5.2 STOPPED 与 PAUSED 的区别

| 状态 | 是否处理数据 | 是否保留 task 和资源 | 是否允许修改 offset |
|---|---|---|---|
| `RUNNING` | 是 | 是 | 否 |
| `PAUSED` | 否 | 是 | 否 |
| `STOPPED` | 否 | 否 | 是，且要求 taskCount=0 |

修改 connector offset 前必须是 `STOPPED`，`PAUSED` 不够。

`PUT /connectors/{connector}/stop` 返回后，状态变化和 task 清理仍可能有短暂延迟。建议通过以下接口确认 connector 和所有 task 均已停止：

```http
GET /connectors/{connector}/status
```

## 6. Task 管理接口

| 方法 | 路径 | 作用 | 说明 |
|---|---|---|---|
| GET | `/connectors/{connector}/tasks` | 查询 task 列表和配置 | 返回 `TaskInfo[]`，每项包含 task ID 和配置 |
| GET | `/connectors/{connector}/tasks-config` | 查询 task 配置 | 已废弃，建议使用 `/tasks`；返回 Map |
| GET | `/connectors/{connector}/tasks/{task}/status` | 查询单个 task 状态 | `{task}` 是整数 task ID |
| POST | `/connectors/{connector}/tasks/{task}/restart` | 重启单个 task | 通常用于重启失败 task，成功通常返回 204 |

## 7. Offset 管理接口

| 方法 | 路径 | 作用 | 说明 |
|---|---|---|---|
| GET | `/connectors/{connector}/offsets` | 查询当前 offset | 运行、暂停、停止状态均可查询 |
| PATCH | `/connectors/{connector}/offsets` | 修改指定分区 offset | connector 必须是 `STOPPED`，且 taskCount=0；不是标准 JSON Patch |
| DELETE | `/connectors/{connector}/offsets` | 重置全部 offset | connector 必须是 `STOPPED`，且 taskCount=0；写入 null/tombstone |

### 7.1 修改 offset

```http
PATCH /connectors/flink-to-mm2-source/offsets
Content-Type: application/json
```

```json
{
  "offsets": [
    {
      "partition": {
        "cluster": "source",
        "topic": "orders",
        "partition": 0
      },
      "offset": {
        "offset": 999
      }
    },
    {
      "partition": {
        "cluster": "source",
        "topic": "orders",
        "partition": 1
      },
      "offset": null
    }
  ]
}
```

其中 `offset: null` 表示单独重置该分区。

### 7.2 状态约束

| Connector 状态 | PATCH/DELETE offsets 结果 |
|---|---|
| `RUNNING` | 400 Bad Request |
| `PAUSED` | 400 Bad Request |
| `STOPPED`，但 task 尚未清理完 | 可能仍返回 400 Bad Request |
| `STOPPED`，且 taskCount=0 | 允许修改 |

错误响应示例：

```json
{
  "error_code": 400,
  "message": "Connectors must be in the STOPPED state before their offsets can be modified. This can be done for the specified connector by issuing a 'PUT' request to the '/connectors/flink-to-mm2-source/stop' endpoint"
}
```

## 8. Topic Tracking 接口

| 方法 | 路径 | 作用 | 说明 |
|---|---|---|---|
| GET | `/connectors/{connector}/topics` | 查询 connector 使用的 topic 集合 | `topic.tracking.enable=false` 时返回 403 |
| PUT | `/connectors/{connector}/topics/reset` | 清空该 connector 的 active topic 集合 | 依赖 `topic.tracking.enable` 和 `topic.tracking.allow.reset`；成功返回 202 |

## 9. 日志级别管理接口

这些接口属于 admin REST API：

- 默认情况下使用普通 `listeners`。
- 配置了 `admin.listeners` 后，只暴露在 admin listener 上。
- 当前版本使用 Log4j 1.x 风格级别。

| 方法 | 路径 | 作用 | 说明 |
|---|---|---|---|
| GET | `/admin/loggers` | 列出显式设置过级别的 logger | 返回 logger 和 level Map |
| GET | `/admin/loggers/{logger}` | 查询 logger 级别 | 不存在返回 404 |
| PUT | `/admin/loggers/{logger}` | 设置 logger 级别 | 请求体如 `{"level":"DEBUG"}` |

### 9.1 设置日志级别

```http
PUT /admin/loggers/org.apache.kafka.connect?scope=worker
Content-Type: application/json
```

```json
{
  "level": "DEBUG"
}
```

`scope` 可选值：

| scope | 含义 | 返回 |
|---|---|---|
| `worker` | 只修改当前 worker，默认值 | 200，返回受影响的 logger 列表 |
| `cluster` | distributed 模式下广播到整个 Connect 集群 | 204 |

## 10. Distributed 模式内部接口

以下接口是 Connect worker 之间通信使用的，不是给用户调用的，也没有列入正常 OpenAPI 文档：

| 方法 | 路径 | 作用 |
|---|---|---|
| POST | `/connectors/{connector}/tasks` | 非 leader worker 将 task 配置转发给 leader 写入 config topic |
| PUT | `/connectors/{connector}/fence` | Exactly-once Source Connector 的 zombie task fencing |

特点：

- 内部请求需要签名。
- 不保证稳定的用户调用契约。
- 可能因为 non-leader、签名错误、rebalance 返回 400/401/409 等错误。
- 运维和迁移脚本不应调用这两个接口。

## 11. Distributed 模式请求路由行为

### 11.1 Leader 转发

创建、修改配置、删除、停止、重启、修改 offset 等操作通常需要 leader 执行。请求发到 follower 时，follower 会内部转发到 leader。

### 11.2 Rebalance 冲突

如果集群正在 rebalance、配置快照落后或暂时找不到 leader，可能返回：

```http
409 Conflict
```

### 11.3 GET 查询

一般可以访问任意 worker，但部分数据具有 worker 局部性或最终一致性：

| 接口 | 行为 |
|---|---|
| `GET /connector-plugins` | 当前 worker 的插件视图 |
| `GET /connectors/{name}/status` | 来自 status topic，跨 worker 聚合并存在短暂延迟 |
| `GET /health` | 只反映当前 worker |

### 11.4 forward 参数

API 中的 `?forward=` 是内部转发参数，正常用户不需要传。接收到请求的 worker 会根据 leader 信息自动转发。

### 11.5 REST Extension

如果在 distributed worker 中配置了 `rest.extension.classes`，第三方扩展可以额外注册资源、filter 或 endpoint，因此实际对外接口可能比内建列表更多。

## 12. 运维场景速查

| 场景 | 接口 |
|---|---|
| 健康检查 | `GET /health` |
| 查看所有 connector | `GET /connectors?expand=status&expand=info` |
| 创建为 STOPPED | `POST /connectors`，`initial_state=STOPPED` |
| 停止 connector | `PUT /connectors/{name}/stop` |
| 查看是否真正停止 | `GET /connectors/{name}/status` |
| 暂停但不释放资源 | `PUT /connectors/{name}/pause` |
| 修改 offset | `PATCH /connectors/{name}/offsets` |
| 重置 offset | `DELETE /connectors/{name}/offsets` |
| 回读 offset | `GET /connectors/{name}/offsets` |
| 启动/恢复 | `PUT /connectors/{name}/resume` |
| 重启失败 task | `POST /connectors/{name}/tasks/{task}/restart` |
| 调整日志级别 | `PUT /admin/loggers/{logger}?scope=worker\|cluster` |

## 13. 参考资料

- `connect/runtime/src/main/java/org/apache/kafka/connect/runtime/rest/ConnectRestServer.java`
- `connect/runtime/src/main/java/org/apache/kafka/connect/runtime/rest/resources/RootResource.java`
- `connect/runtime/src/main/java/org/apache/kafka/connect/runtime/rest/resources/ConnectorsResource.java`
- `connect/runtime/src/main/java/org/apache/kafka/connect/runtime/rest/resources/ConnectorPluginsResource.java`
- `connect/runtime/src/main/java/org/apache/kafka/connect/runtime/rest/resources/LoggingResource.java`
- `connect/runtime/src/main/java/org/apache/kafka/connect/runtime/rest/resources/InternalClusterResource.java`
- `docs/generated/connect_rest.yaml`
- `docs/kafka-connect/user-guide.md`
