---
title: "ads-bz 备份集群升级 3.9.2 操作手册"
description: "ads-bz 从 Kafka 1.1.1 平行部署并切换到 3.9.2 KRaft 的实施流程"
date: 2026-09-14
categories: ["operations-guide"]
tags: ["kafka", "kraft", "mirrormaker2", "高可用", "数据迁移"]
---

## 1. 背景与目标

当前生产链路为 `ads -> ads-bz`：

- `ads` 是主集群，承载正常业务流量。
- `ads-bz` 是备份集群，通过 MM2 从 `ads` 单向同步数据，平时没有业务流量。
- 两个集群当前均为 Kafka 1.1.1，且 Kafka 与 MM2 均包含内部修改，不能直接替换为社区二进制包。

本次先把 `ads-bz` 升级到 Kafka 3.9.2。升级方式不是原地迁移 ZooKeeper 元数据，而是平行部署一套新的 3.9.2 KRaft 集群：

- 新集群使用独立数据目录：`/data*/kafka-adbz/data`。
- 新集群临时使用独立端口：broker 间通信 8111，客户端访问 8110。
- KRaft Controller 部署在 `kraft-adbz-controller01.kafka.data.sina.com.cn`、`kraft-adbz-controller02.kafka.data.sina.com.cn`、`kraft-adbz-controller03.kafka.data.sina.com.cn`、`kraft-adbz-controller04.kafka.data.sina.com.cn`、`kraft-adbz-controller05.kafka.data.sina.com.cn`，Controller 监听端口为 8993。
- 旧 `ads-bz` 1.1.1 集群在验证期内保持运行，继续作为当前可用备份。
- 新集群通过 MM2 从 `ads` 同步数据并完成验证。
- 验证通过后，停止旧 `ads-bz (1.1.1)`，释放 9110 和 9111，再将新集群端口切换为 9110 和 9111，成为新的 `ads-bz (3.9.2)` 备份集群。
- 最终上线前不复用验证期数据：清理并重新初始化 Controller 与 broker 数据，broker 数据目录切回原来的 `/data*/kafka/data`，再重新同步 topic 等数据。
- 切换后保留旧版本 Kafka 部署包和配置，以及原来的 MM2 3.8.2 独立模式部署包和配置，用于新版本异常时回滚。

本方案的核心原则是：验证期不占用旧备份集群端口，不修改旧集群数据目录；只有新集群完成稳定性、数据、权限和兼容性验证后，才执行最终上线切换。最终切换会按计划清理并重建数据目录，属于上线初始化的一部分。

## 2. 目标架构

| 组件 | 数量 | 临时端口 | 数据目录 | 说明 |
|---|---:|---:|---|---|
| KRaft Controller | 5 | 8993 | `/data1/kafka-controller-adbz/data` | 验证期和最终上线前均先于 broker 部署；最终上线前重新初始化 |
| Kafka Broker | 53 | 8111 | 验证期 `/data*/kafka-adbz/data`，最终 `/data*/kafka/data` | broker 间监听 8111；最终上线前切回原数据目录 |
| 客户端监听 | 53 | 8110 | 不适用 | MM2、管理工具和验证客户端访问 8110 |
| MM2 | 按现有规模 | 不适用 | 按现有部署 | 新增 `ads -> ads-bz (3.9.2)` 单向链路 |

验证期拓扑：

```text
ads 1.1.1
  |
  +--> MM2 旧链路 --> ads-bz (1.1.1)（9110/9111，继续保留）
  |
  +--> MM2 新链路 --> ads-bz (3.9.2)（8110/8111，KRaft）
```

最终拓扑：

```text
ads 1.1.1
  |
  +--> MM2 --> ads-bz (3.9.2)（9110/9111，KRaft）
```

Controller 不需要切换到旧 `ads-bz (1.1.1)` 的 9193 端口。旧配置里的 `CONTROLLER` 是旧部署的控制/管理通道，与 KRaft quorum 的 Controller listener 不是同一概念。只要 8993 不冲突且网络策略明确，可以继续使用。

## 3. 前置条件

### 3.1 版本与代码

- Kafka 3.9.2 发布包必须基于内部代码分支重新构建，不能替换为社区发行包。
- MM2 如需使用 3.9.2，也必须基于内部修改重新构建并完成回归。
- 发布包需要记录 Git commit、构建参数、JDK 版本、变更点和 SHA256。
- 在测试环境完成以下回归后再进入生产：
  - Kafka 3.9.2 KRaft Controller 和 broker 启停。
  - 自研配置项解析，尤其是 `client.api.version.cap.*` 和旧配置中被移除的 replica 自定义参数。
  - SASL PLAIN、ACL、produce、fetch、consumer group、事务消息。
  - MM2 topic、partition 和 consumer group offset 同步。
  - Kafka 1.1.1 客户端访问 Kafka 3.9.2 broker。

### 3.2 运行环境

- JDK 版本按 3.9.2 构建和测试结论统一，建议使用 JDK 17，并固定到具体小版本。
- 53 台 broker 的磁盘容量必须同时容纳旧目录和新目录。新目录不是额外磁盘，只是同盘不同路径，不能只看目录是否为空，必须看文件系统剩余空间。
- 端口检查必须覆盖 53 台 broker 的 8110、8111、9893、10013，以及 5 台 Controller 的 8993、9893、10013。
- 新集群 JVM 堆内存、GC 参数、文件句柄数、磁盘 IO 和网络带宽不能低于现有生产标准。
- 监控需要覆盖 Controller quorum、broker 存活、ISR、offline partition、磁盘、网络、请求延迟、日志错误和 MM2 lag。

### 3.3 元数据与权限

Kafka 3.9.2 使用 `StandardAuthorizer`，ACL 保存在 KRaft 元数据日志中，不会读取旧 ZooKeeper 集群中的 ACL。新集群初始只有 `User:admin` 这类 super user 权限。

ACL 不依赖 MM2 同步：

1. 验证期使用内部 ACL 同步接口的本地执行模式，手动向新集群写入 ACL，并用普通业务账号验证授权和拒绝行为。
2. 最终上线前会重新初始化 Controller 和数据目录，ACL 一并清空；初始化完成后使用线上 ACL 同步接口处理。
3. 线上接口执行后必须核对执行结果，并抽样验证 ACL 数量和授权语义。

由于 `allow.everyone.if.no.acl.found=false`，ACL 缺失会直接表现为业务无权限。线上接口执行记录必须归档。

## 4. 实施步骤

### 4.1 部署 KRaft Controller

Controller 必须先部署。

#### 4.1.1 修改 Controller 启动脚本

在修改前，先在五台 Controller 节点创建日志目录：

```bash
mkdir -p /data0/logs/kafka-adbz
```

每台 Controller 的 `bin/kafka-server-start.sh` 修改后完整内容如下（与 broker 启动脚本的区别已在注释中标注）：

```bash
#!/bin/bash
# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#    http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

if [ $# -lt 1 ];
then
        echo "USAGE: $0 [-daemon] server.properties [--override property=value]*"
        exit 1
fi
base_dir=$(dirname $0)

if [ "x$KAFKA_LOG4J_OPTS" = "x" ]; then
    export KAFKA_LOG4J_OPTS="-Dlog4j.configuration=file:$base_dir/../config/log4j.properties"
fi

if [ "x$KAFKA_HEAP_OPTS" = "x" ]; then
    export KAFKA_HEAP_OPTS="-Xmx6G -Xms6G"          # Controller 使用 6G，broker 使用 16G
fi

EXTRA_ARGS=${EXTRA_ARGS-'-name kafkaServer -loggc'}

COMMAND=$1
case $COMMAND in
  -daemon)
    EXTRA_ARGS="-daemon "$EXTRA_ARGS
    shift
    ;;
  *)
    ;;
esac

IP=`ifconfig | grep 'netmask 255.255.255' | grep inet | awk '{print $2}' | head -n1`
export JMX_PORT="9893"                               # 与 broker 保持一致
export PROMETHEUS_PORT="10013"                       # 与 broker 保持一致
export HADOOP_USER_NAME="hadoop_kraft"
export JAVA_HOME="/usr/local/jdk-17.0.7"
export LOG_DIR=/data0/logs/kafka-adbz                # Controller 与 broker 统一使用 kafka-adbz 日志目录
export KAFKA_OPTS="-Djava.security.auth.login.config=$base_dir/../config/kafka_server_jaas.conf"
export KAFKA_JMX_OPTS="-Dcom.sun.management.jmxremote=true -Dcom.sun.management.jmxremote.authenticate=true -Dcom.sun.management.jmxremote.local.only=false -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.host=0.0.0.0 -Djava.rmi.server.hostname=$IP -Dcom.sun.management.jmxremote.password.file=$base_dir/../config/jmxremote.password -Dcom.sun.management.jmxremote.access.file=$base_dir/../config/jmxremote.access -javaagent:$base_dir/../libs/jmx_prometheus_javaagent-0.20.0.jar=${PROMETHEUS_PORT:-19999}:$base_dir/../config/jmx.yaml"

exec $base_dir/kafka-run-class.sh $EXTRA_ARGS kafka.Kafka "$@"
```

#### 4.1.2 修改配置文件

创建 Controller 元数据目录：

```bash
mkdir -p /data1/kafka-controller-adbz/data
```

每台 Controller 需生成独立的 `server.properties`，以下为完整配置模板。五台 Controller 必须修改以下内容：

- **`node.id`**：依次为 `1000`、`1001`、`1002`、`1003`、`1004`，与 `--initial-controllers` 中的编号对应，且必须等于该节点配置中的 `node.id`。
- **`listeners`**：固定为 `CONTROLLER://:8993`。
- **`advertised.listeners`**：将 `<本机IP>` 替换为该 Controller 的实际 IP，不使用域名。
- **`log.dirs`**：使用 `/data1/kafka-controller-adbz/data`。

```properties
# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#    http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

#
# This configuration file is intended for use in KRaft mode, where
# Apache ZooKeeper is not present.
#

############################# Server Basics #############################

# The role of this server. Setting this puts us in KRaft mode
process.roles=controller

# The node id associated with this instance's roles
node.id=1000

# List of controller endpoints used connect to the controller cluster
controller.quorum.bootstrap.servers=kraft-adbz-controller01.kafka.data.sina.com.cn:8993,kraft-adbz-controller02.kafka.data.sina.com.cn:8993,kraft-adbz-controller03.kafka.data.sina.com.cn:8993,kraft-adbz-controller04.kafka.data.sina.com.cn:8993,kraft-adbz-controller05.kafka.data.sina.com.cn:8993

# inter.broker.protocol.version=3.9

# The max time that the client waits to establish a connection to ZooKeeper for authorizer
# authorizer.zookeeper.connection.timeout.ms=15000

# Zookeeper session timeout 
# authorizer.zookeeper.session.timeout.ms=60000

# Name of listener used for communication between brokers
# inter.broker.listener.name=SASL_PLAINTEXT

# SASL mechanism used for inter-broker communication.
# sasl.mechanism.inter.broker.protocol=PLAIN

# The fully qualified name of a class which is used by the broker for authorization.
authorizer.class.name=org.apache.kafka.metadata.authorizer.StandardAuthorizer
allow.everyone.if.no.acl.found=false
kafka.authorizer.wait.timeout.ms=20000

############################# Socket Server Settings #############################

# The address the socket server listens on.
listeners=CONTROLLER://:8993

# Listener name, hostname and port the broker or the controller will advertise to clients.
# If not set, it uses the value for "listeners".
advertised.listeners=CONTROLLER://<本机IP>:8993

sasl.enabled.mechanisms=PLAIN

# A comma-separated list of the names of the listeners used by the controller.
# This is required if running in KRaft mode.
controller.listener.names=CONTROLLER

sasl.mechanism.controller.protocol=PLAIN

client.api.version.cap.enable=true

# Maps listener names to security protocols, the default is for them to be the same.
listener.security.protocol.map=CONTROLLER:SASL_PLAINTEXT,PLAINTEXT:PLAINTEXT,SSL:SSL,SASL_PLAINTEXT:SASL_PLAINTEXT,SASL_SSL:SASL_SSL

# super user 
super.users=User:admin

# The number of threads that the server uses for receiving requests from the network and sending responses to the network
num.network.threads=20

# The number of threads that the server uses for processing requests, which may include disk I/O
num.io.threads=20

# The send buffer (SO_SNDBUF) used by the socket server
socket.send.buffer.bytes=1048576

# The receive buffer (SO_RCVBUF) used by the socket server
socket.receive.buffer.bytes=1048576

# The maximum size of a request that the socket server will accept (protection against OOM)
socket.request.max.bytes=104857600

# The number of queued requests allowed before blocking the network threads
queued.max.requests=200

############################# Log Basics #############################

# A comma separated list of directories under which to store log files
log.dirs=/data1/kafka-controller-adbz/data

# The default number of log partitions per topic.
num.partitions=3

# The number of threads per data directory to be used for log recovery at startup and flushing at shutdown.
num.recovery.threads.per.data.dir=1

############################# Internal Topic Settings  #############################
# The replication factor for the group metadata internal topics "__consumer_offsets" and "__transaction_state"
offsets.topic.replication.factor=3

# The replication factor for the transaction topic 
transaction.state.log.replication.factor=3

# Overridden min.insync.replicas config for the transaction topic
transaction.state.log.min.isr=1

# Enable auto creation of topic on the server.
auto.create.topics.enable=false

# Enables auto leader balancing.
auto.leader.rebalance.enable=false

############################# Log Retention Policy #############################

# The following configurations control the disposal of log segments. The policy can
# be set to delete segments after a period of time, or after a given size has accumulated.
# A segment will be deleted whenever *either* of these criteria are met. Deletion always happens
# from the end of the log.

# A size-based retention policy for logs.
log.retention.bytes=536870912000

# The maximum size of a log segment file.
log.segment.bytes=1073741824

# The minimum age of a log file to be eligible for deletion
log.retention.hours=8

# The maximum time before a new log segment is rolled out (in hours)
log.roll.hours=24

# Indicates whether to enable replicas not in the ISR set to be elected as leader as a last resort
unclean.leader.election.enable=true

# The interval at which log segments are checked to see if they can be deleted according
# to the retention policies
log.retention.check.interval.ms=300000

# ----- Enable tiered storage -----
remote.log.storage.system.enable=false
```

Controller-only 节点不需要配置 `advertised.listeners`（此处保留是用于显式指定 advertise 地址），Controller 的访问地址来自 `controller.quorum.bootstrap.servers` 和格式化时写入的初始 Controller 信息。为避免同 ID 迁移 Controller 时 KRaft 记录的 endpoint 保留旧域名，`advertised.listeners` 和 `--initial-controllers` 均使用实际 IP。

#### 4.1.3 格式化数据目录

先在发布机或任意一台 Controller 上生成一次 `KAFKA_CLUSTER_ID` 和五个 Controller directory id，并归档完整的 `--initial-controllers` 字符串。五台 Controller 必须读取同一份 cluster id 和 initial controllers 文件，不能各自执行 `random-uuid`。

`--initial-controllers` 的格式是：

```text
<node-id>@<hostname>:<port>:<directory-id>
```

其中 `directory-id` 不是磁盘路径，而是写入 `meta.properties` 的 Controller metadata directory ID。五个 Controller 的 directory id 必须分别唯一，且完整的 `--initial-controllers` 字符串必须在五台 Controller 上保持完全一致。`<hostname>` 使用 Controller 实际 IP，不使用域名，避免同 ID 迁移 Controller 时 KRaft 记录的 endpoint 保留旧域名：

```bash
# 只在发布机或任意一台 Controller 上执行一次。
# Controller IP 使用实际 IP，不使用域名。
CONTROLLER_1000_IP=<controller01-ip>
CONTROLLER_1001_IP=<controller02-ip>
CONTROLLER_1002_IP=<controller03-ip>
CONTROLLER_1003_IP=<controller04-ip>
CONTROLLER_1004_IP=<controller05-ip>

KAFKA_CLUSTER_ID=$(bin/kafka-storage.sh random-uuid)
CONTROLLER_1000_DIRECTORY_ID=$(bin/kafka-storage.sh random-uuid)
CONTROLLER_1001_DIRECTORY_ID=$(bin/kafka-storage.sh random-uuid)
CONTROLLER_1002_DIRECTORY_ID=$(bin/kafka-storage.sh random-uuid)
CONTROLLER_1003_DIRECTORY_ID=$(bin/kafka-storage.sh random-uuid)
CONTROLLER_1004_DIRECTORY_ID=$(bin/kafka-storage.sh random-uuid)

INITIAL_CONTROLLERS="1000@${CONTROLLER_1000_IP}:8993:${CONTROLLER_1000_DIRECTORY_ID},1001@${CONTROLLER_1001_IP}:8993:${CONTROLLER_1001_DIRECTORY_ID},1002@${CONTROLLER_1002_IP}:8993:${CONTROLLER_1002_DIRECTORY_ID},1003@${CONTROLLER_1003_IP}:8993:${CONTROLLER_1003_DIRECTORY_ID},1004@${CONTROLLER_1004_IP}:8993:${CONTROLLER_1004_DIRECTORY_ID}"

printf '%s\n' "$KAFKA_CLUSTER_ID" | tee ads-bz.cluster-id
printf '%s\n' "$INITIAL_CONTROLLERS" | tee ads-bz.initial-controllers
```

将 `ads-bz.cluster-id` 和 `ads-bz.initial-controllers` 分发到五台 Controller 后，每台 Controller 分别执行以下命令。配置文件替换为该节点配置：

```bash
KAFKA_CLUSTER_ID=$(cat ads-bz.cluster-id)
INITIAL_CONTROLLERS=$(cat ads-bz.initial-controllers)

bin/kafka-storage.sh format \
  --initial-controllers "$INITIAL_CONTROLLERS" \
  -t "$KAFKA_CLUSTER_ID" \
  -c config/kraft/server.properties
```

如果 `format` 提示不识别 `--initial-controllers`，说明当前包不是预期的 3.9.2 动态 KRaft 配置，或团队选择了静态 quorum 模式。此时停止操作，确认发布包和配置模式；不要混用 `controller.quorum.bootstrap.servers` 与静态 `controller.quorum.voters`。

首次部署时目录必须为空。`--ignore-formatted` 只允许用于失败重试前确认已有格式化结果，不能用来跳过目录检查。

#### 4.1.4 启动 Controller

格式化完成后按顺序启动五台 Controller：

```bash
bin/kafka-server-start.sh -daemon config/kraft/server.properties
```

检查项：

- 至少三台（共五台）进程存活，日志无 `FATAL` 或持续 `ERROR`。
- quorum 能选出 leader。
- 8993 只暴露给 Controller、broker 和运维网络。
- Controller 数据目录有元数据文件，磁盘监控正常。

### 4.2 部署 broker

#### 4.2.1 修改 broker 启动脚本

在修改前，先在所有 broker 节点创建日志目录：

```bash
mkdir -p /data0/logs/kafka-adbz
```

每台 broker 的 `bin/kafka-server-start.sh` 修改后完整内容如下（对比旧集群启动脚本，修改处已用注释标注）：

```bash
#!/bin/bash
# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#    http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

if [ $# -lt 1 ];
then
        echo "USAGE: $0 [-daemon] server.properties [--override property=value]*"
        exit 1
fi
base_dir=$(dirname $0)

if [ "x$KAFKA_LOG4J_OPTS" = "x" ]; then
    export KAFKA_LOG4J_OPTS="-Dlog4j.configuration=file:$base_dir/../config/log4j.properties"
fi

if [ "x$KAFKA_HEAP_OPTS" = "x" ]; then
    export KAFKA_HEAP_OPTS="-Xmx16G -Xms16G"
fi

EXTRA_ARGS=${EXTRA_ARGS-'-name kafkaServer -loggc'}

COMMAND=$1
case $COMMAND in
  -daemon)
    EXTRA_ARGS="-daemon "$EXTRA_ARGS
    shift
    ;;
  *)
    ;;
esac

IP=`ifconfig | grep 'netmask 255.255.255' | grep inet | awk '{print $2}' | head -n1`
export JMX_PORT="9893"
export PROMETHEUS_PORT="10013"
export HADOOP_USER_NAME="hadoop_kraft"
export JAVA_HOME="/usr/local/jdk-17.0.7"
export LOG_DIR=/data0/logs/kafka-adbz
export KAFKA_OPTS="-Djava.security.auth.login.config=$base_dir/../config/kafka_server_jaas.conf"
export KAFKA_JMX_OPTS="-Dcom.sun.management.jmxremote=true -Dcom.sun.management.jmxremote.authenticate=true -Dcom.sun.management.jmxremote.local.only=false -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.host=0.0.0.0 -Djava.rmi.server.hostname=$IP -Dcom.sun.management.jmxremote.password.file=$base_dir/../config/jmxremote.password -Dcom.sun.management.jmxremote.access.file=$base_dir/../config/jmxremote.access -javaagent:$base_dir/../libs/jmx_prometheus_javaagent-0.20.0.jar=${PROMETHEUS_PORT:-19999}:$base_dir/../config/jmx.yaml"

exec $base_dir/kafka-run-class.sh $EXTRA_ARGS kafka.Kafka "$@"
```

修改说明（共 3 处）：

| 参数 | 旧集群默认值 | 新集群修改值 |
|---|---|---|
| `JMX_PORT` | `9999` | `9893` |
| `PROMETHEUS_PORT` | 不适用（旧集群未启用） | `10013` |
| `LOG_DIR` | `/data0/logs/kafka` | `/data0/logs/kafka-adbz` |

以上端口和日志目录修改适用于 53 台新集群 broker。Controller 节点仅 5 台，同样使用 JMX `9893` 和 Prometheus `10013`，保持管理端口一致。

旧 `ads-bz` 1.1.1 的 JMX 端口为 `9999`，且未启用 Prometheus。新集群统一使用 `9893` 和 `10013`，最终上线端口切换为 `9110` 和 `9111` 时，这两个管理端口保持不变；`9999` 在升级后的新集群中不再使用。

#### 4.2.2 修改配置文件

在修改前，先在所有 broker 节点创建数据目录：

```bash
mkdir -p /data1/kafka-adbz/data /data2/kafka-adbz/data /data3/kafka-adbz/data /data4/kafka-adbz/data /data5/kafka-adbz/data /data6/kafka-adbz/data /data7/kafka-adbz/data /data8/kafka-adbz/data /data9/kafka-adbz/data /data10/kafka-adbz/data
```

每台 broker 需生成独立的 `server.properties`，以下为完整配置模板。每台 broker 必须修改以下内容：

- **`node.id`**：与旧集群 `broker.id` 保持一致，确保全局唯一（含 Controller）。
- **`listeners`** 和 **`advertised.listeners`**：将 `<本机IP>` 替换为该节点实际本机 IP，将 `<客户端-advertised-域名>` 替换为实际客户端访问域名。
- **`log.dirs`**：按本机实际挂载盘列出完整路径，不支持通配符展开。

```properties
# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#    http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
# see kafka.server.KafkaConfig for additional details and defaults

############################# Server Basics #############################

# The id of the broker. This must be set to a unique integer for each broker.
node.id=1
process.roles=broker

############################# Socket Server Settings #############################
# List of comma-separated URIs the REST API will listen on.
listeners=SASL_PLAINTEXT_BROKER://:8111,SASL_PLAINTEXT://:8110
advertised.listeners=SASL_PLAINTEXT_BROKER://<本机IP>:8111,SASL_PLAINTEXT://<客户端-advertised-域名>:8110
# Map between listener names and security protocols.
listener.security.protocol.map=CONTROLLER:SASL_PLAINTEXT,PLAINTEXT:PLAINTEXT,SSL:SSL,SASL_PLAINTEXT:SASL_PLAINTEXT,SASL_SSL:SASL_SSL,SASL_PLAINTEXT_BROKER:SASL_PLAINTEXT
# Name of listener used for communication between brokers
inter.broker.listener.name=SASL_PLAINTEXT_BROKER
# SASL mechanism used for inter-broker communication.
sasl.mechanism.inter.broker.protocol=PLAIN
# The list of SASL mechanisms enabled in the Kafka server.
sasl.enabled.mechanisms=PLAIN
# The super user does not require ACL authentication and has the highest permission
super.users=User:admin

############################# Broker Basic ###########################
# Rack of the broker.
broker.rack=rack-A
# The number of threads handling network requests
num.network.threads=40
# The number of threads doing disk I/O
num.io.threads=40
# The send buffer (SO_SNDBUF) used by the socket server
socket.send.buffer.bytes=1048576
# The receive buffer (SO_RCVBUF) used by the socket server
socket.receive.buffer.bytes=1048576
# The maximum size of a request that the socket server will accept (protection against OOM)
socket.request.max.bytes=104857600
## The number of queued requests allowed before blocking the network threads
queued.max.requests=200
# The purge interval (in number of requests) of the fetch request purgatory
fetch.purgatory.purge.interval.requests=100
# The purge interval (in number of requests) of the producer request purgatory	
producer.purgatory.purge.interval.requests=100
# The amount of time the group coordinator will wait for more consumers to join a new group before performing the first rebalance
group.initial.rebalance.delay.ms=3000
# When set to true, the broker caps the maximum advertised and accepted version of the APIs listed in client.api.version.cap.overrides for client-facing listeners only.
client.api.version.cap.enable=true
# use the default configuration
# client.api.version.cap.overrides=METADATA:6,FETCH:8,LIST_OFFSETS:3,OFFSET_COMMIT:5,OFFSET_FETCH:4,OFFSET_FOR_LEADER_EPOCH:1

############################# acl for broker -- zookeeper #############################
# zookeeper.set.acl=true
allow.everyone.if.no.acl.found=false
authorizer.class.name=org.apache.kafka.metadata.authorizer.StandardAuthorizer
# authorizer.zookeeper.connection.timeout.ms=15000
# authorizer.zookeeper.session.timeout.ms=60000
kafka.authorizer.wait.timeout.ms=20000

############################# Topic Basic ###########################
auto.create.topics.enable=false
auto.leader.rebalance.enable=false
## The frequency with which the partition rebalance check is triggered by the controller. set 600s
leader.imbalance.check.interval.seconds=3600
## The ratio of leader imbalance allowed per broker. The controller would trigger a leader balance if it goes above this value per broker. The value is specified in percentage.
leader.imbalance.per.broker.percentage=20
# Enables delete topic. Delete topic through the admin tool will have no effect if this config is turned off
delete.topic.enable=true
## Specify the final compression type for a given topic. This configuration accepts the standard compression codecs ('gzip', 'snappy', lz4).
## It additionally accepts 'uncompressed' which is equivalent to no compression; and 'producer' which means retain the original compression codec set by the producer.
compression.type=producer
## Any consumer distinguished by clientId/consumer group will get throttled if it fetches more bytes than this value per-second
#quota.consumer.default=20971520
default.replication.factor=3
min.insync.replicas=2

############################# Replication configurations ################################
# Number of fetcher threads used to replicate messages from a source broker.
num.replica.fetchers=8
# The number of bytes of messages to attempt to fetch for each partition. 
replica.fetch.max.bytes=1048576
# Minimum bytes expected for each fetch response. If not enough bytes, wait up to replicaMaxWaitTimeMs
replica.fetch.min.bytes=102400
# max wait time for each fetcher request issued by follower replicas.
replica.fetch.wait.max.ms=300
# The frequency with which the high watermark is saved out to disk
replica.high.watermark.checkpoint.interval.ms=3000
## If a follower hasn't sent any fetch requests or hasn't consumed up to the leaders log end offset for at least this time, the leader will remove the follower from isr
replica.lag.time.max.ms=10000
# The socket receive buffer for network requests
replica.socket.receive.buffer.bytes=65536
# The socket timeout for network requests. Its value should be at least replica.fetch.wait.max.ms	
replica.socket.timeout.ms=30000

############################# offset Basics ################################
# 	The number of partitions for the offset commit topic
offsets.topic.num.partitions=30
# The replication factor for the offsets topic
offsets.topic.replication.factor=3
# Offsets older than this retention period will be discarded
offsets.retention.minutes=2880
## The offsets topic segment bytes should be kept relatively small in order to facilitate faster log compaction and cache loads
offsets.topic.segment.bytes=67108864

############################# Log Basics #############################
# A comma seperated list of directories under which to store log files
log.dirs=/data1/kafka-adbz/data,/data2/kafka-adbz/data,/data3/kafka-adbz/data,/data4/kafka-adbz/data,/data5/kafka-adbz/data,/data6/kafka-adbz/data,/data7/kafka-adbz/data,/data8/kafka-adbz/data,/data9/kafka-adbz/data,/data10/kafka-adbz/data
# A comma seperated list of directories under which to store log files
log.cleanup.policy = delete
# Enable the log cleaner process to run on the server.
log.cleaner.enable=true
# The default number of log partitions per topic. More partitions allow greater
# parallelism for consumption, but this will also result in more files across
# the brokers.
num.partitions=3
# The largest record batch size allowed by Kafka.
message.max.bytes=1000012
# The number of threads per data directory to be used for log recovery at startup and flushing at shutdown.
# This value is recommended to be increased for installations with data dirs located in RAID array.
num.recovery.threads.per.data.dir=1

############################# Log Flush Policy #############################

# Messages are immediately written to the filesystem but by default we only fsync() to sync
# the OS cache lazily. The following configurations control the flush of data to disk.
# There are a few important trade-offs here:
#    1. Durability: Unflushed data may be lost if you are not using replication.
#    2. Latency: Very large flush intervals may lead to latency spikes when the flush does occur as there will be a lot of data to flush.
#    3. Throughput: The flush is generally the most expensive operation, and a small flush interval may lead to exceessive seeks.
# The settings below allow one to configure the flush policy to flush data after a period of time or
# every N messages (or both). This can be done globally and overridden on a per-topic basis.

# The number of messages to accept before forcing a flush of data to disk
# log.flush.interval.messages=100000
#log.flush.scheduler.interval.ms=5000

# The maximum amount of time a message can sit in a log before we force a flush
#log.flush.interval.ms=5000
log.index.interval.bytes=4096
log.index.size.max.bytes=10485760

############################# Log Retention Policy #############################

# The following configurations control the disposal of log segments. The policy can
# be set to delete segments after a period of time, or after a given size has accumulated.
# A segment will be deleted whenever *either* of these criteria are met. Deletion always happens
# from the end of the log.

# The minimum age of a log file to be eligible for deletion
log.retention.hours=72
# A size-based retention policy for logs. Segments are pruned from the log as long as the remaining
# segments don't drop below log.retention.bytes.
#log.retention.bytes=1073741824
log.retention.bytes=536870912000
# The maximum size of a log segment file. When this size is reached a new log segment will be created.
log.segment.bytes=1610612736
# The maximum time before a new log segment is rolled out (in hours), secondary to log.roll.ms property
log.roll.hours=24
# The interval at which log segments are checked to see if they can be deleted according
# to the retention policies
log.retention.check.interval.ms=300000
# The number of background threads to use for log cleaning
log.cleaner.threads=2

############################# Zookeeper #############################

# Zookeeper connection string (see zookeeper docs for details).
# This is a comma separated host:port pairs, each corresponding to a zk
# server. e.g. "127.0.0.1:3001,127.0.0.1:3002".
# You can also append an optional chroot string to the urls to specify the
# root directory for all kafka znodes.
# zookeeper.connect=zk1.data.sina.com.cn:2181,zk6.data.sina.com.cn:2181,zk2.data.sina.com.cn:2181,zk5.data.sina.com.cn:2181,zk4.data.sina.com.cn:2181,zk3.data.sina.com.cn:2181,zk7.data.sina.com.cn:2181/weibo_ads_cap_test
# Timeout in ms for connecting to zookeeper
# zookeeper.connection.timeout.ms=10000
##session timeout also depend on the zookeeper config of minSessionTimeout and maxSessionTimeout
# zookeeper.session.timeout.ms=30000

############################# controller #############################
# Name of listener used for communication between controller and brokers
# control.plane.listener.name=CONTROLLER
# The socket timeout for controller-to-broker channels
controller.socket.timeout.ms=30000
# Enable controlled shutdown of the server	
controlled.shutdown.enable=true
controller.quorum.bootstrap.servers=kraft-adbz-controller01.kafka.data.sina.com.cn:8993,kraft-adbz-controller02.kafka.data.sina.com.cn:8993,kraft-adbz-controller03.kafka.data.sina.com.cn:8993,kraft-adbz-controller04.kafka.data.sina.com.cn:8993,kraft-adbz-controller05.kafka.data.sina.com.cn:8993
controller.listener.names=CONTROLLER
sasl.mechanism.controller.protocol=PLAIN
# zookeeper.metadata.migration.enable=true

############################# client #############################
client.fetchOrProduce.correlation.interval=1000
client.fetchOrProduce.correlation.init=20
client.heartbeat.correlation.interval=1000
client.heartbeat.correlation.init=20

############################# transaction #############################
# transaction.state.log.min.isr=1
# transaction.state.log.replication.factor=3
unclean.leader.election.enable=true
# 远程存储配置
############################# Log Tiered Storage ############################
# ----- Enable tiered storage -----
remote.log.storage.system.enable=false
remote.log.reader.threads=100
remote.log.reader.max.pending.tasks=10000
# Retention of remote log metadata topic in milliseconds.
# Notes:To avoid any data loss, this value should be more than the maximum retention period of any topic enabled with tiered storage in the cluster.
remote.log.metadata.topic.retention.ms=2592000000
# The maximum number of bytes that can be copied from local storage to remote storage per second
remote.log.manager.copy.max.bytes.per.second=209715200

# ----- Configure the remote log manager -----
# This is the default, but adding it for explicitness:
remote.log.metadata.manager.class.name=org.apache.kafka.server.log.remote.metadata.storage.TopicBasedRemoteLogMetadataManager

# Put the real listener name you'd like to use here:
remote.log.metadata.manager.listener.name=SASL_PLAINTEXT
rlmm.config.remote.log.metadata.common.client.sasl.mechanism=PLAIN
rlmm.config.remote.log.metadata.common.client.sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username='admin' password='d4a12dfe3f97e641edd9f206eca5ae92';
rlmm.config.remote.log.metadata.producer.security.protocol=SASL_PLAINTEXT
rlmm.config.remote.log.metadata.producer.sasl.mechanism=PLAIN
rlmm.config.remote.log.metadata.producer.sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username='admin' password='d4a12dfe3f97e641edd9f206eca5ae92';
rlmm.config.remote.log.metadata.consumer.security.protocol=SASL_PLAINTEXT
rlmm.config.remote.log.metadata.consumer.sasl.mechanism=PLAIN
rlmm.config.remote.log.metadata.consumer.sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username='admin' password='d4a12dfe3f97e641edd9f206eca5ae92';

# You may need to set this if you're experimenting with a single broker setup:
#rlmm.config.remote.log.metadata.topic.replication.factor=1

# ----- Configure the remote storage manager -----
# Here you need either one or two directories depending on what you did in Step 1:
remote.log.storage.manager.class.path=/usr/local/kafka/kafka-release/tired-storage/*
remote.log.storage.manager.class.name=io.aiven.kafka.tieredstorage.RemoteStorageManager
# 8 MiB is the current recommended chunk size:
rsm.config.chunk.size=8388608

# ----- Configure the storage backend -----
# Using HDFS backend as an example:
rsm.config.storage.backend.class=io.arenadata.kafka.tieredstorage.storage.hdfs.HdfsStorage
rsm.config.storage.hdfs.root=/user/hadoop_kraft/weibo_ads_cap_test
rsm.config.storage.hdfs.core-site.path=/usr/local/kafka/kafka-release/config/hadoop/core-site.xml
rsm.config.storage.hdfs.hdfs-site.path=/usr/local/kafka/kafka-release/config/hadoop/hdfs-site.xml
# The prefix can be skipped:
#rsm.config.storage.key.prefix: "some/prefix/"

# ----- Configure the fetch chunk cache -----
rsm.config.fetch.chunk.cache.class=io.aiven.kafka.tieredstorage.fetch.cache.DiskChunkCache
rsm.config.fetch.chunk.cache.path=/data11/cache
# fetch chunk rate
rsm.config.fetch.chunk.cache.write.rate.limit.bytes.per.second=524288000
# Pick some cache size, 300 GiB here:
rsm.config.fetch.chunk.cache.size=322122547200
# Prefetching size, 16 MiB here:
rsm.config.fetch.chunk.cache.prefetch.max.size=16777216
# Cache retention time ms, where -1 represents infinite retention
rsm.config.fetch.chunk.cache.retention.ms=600000
```

#### 4.2.3 格式化数据目录

每台 broker 执行以下命令，读取同一份 `ads-bz.cluster-id`，使用 Controller 部署时生成的 `KAFKA_CLUSTER_ID` 格式化数据目录：

```bash
KAFKA_CLUSTER_ID=$(cat ads-bz.cluster-id)
bin/kafka-storage.sh format \
  -t "$KAFKA_CLUSTER_ID" \
  -c config/kraft/server.properties
```

格式化前确认 `log.dirs` 中所有目录均为空。`--ignore-formatted` 只允许用于失败重试前确认已有格式化结果，不能用来跳过目录检查。

#### 4.2.4 启动 broker

格式化完成后启动 broker：

```bash
bin/kafka-server-start.sh -daemon config/kraft/server.properties
```

建议按批次启动，例如每批 5 到 10 台：

1. 启动一批 broker。
2. 确认进程、端口、日志、Controller 连接和磁盘 IO 正常。
3. 确认前一批 broker 均注册成功，再启动下一批。
4. 53 台全部启动后，确认 broker 数量、副本分布和 leader 分布。

不要一次性启动 53 台。虽然新集群没有业务流量，但同时启动会集中产生日志恢复、注册和 Controller 通信压力，也会与旧集群共享磁盘和网络。

### 4.3 预处理 ACL 并生成 topic

1. 使用内部 ACL 同步接口的本地执行模式向新集群写入 ACL。
2. 执行 ACL 数量对比和抽样语义对比，并用普通业务账号验证授权和拒绝行为。
3. 创建验证 topic，副本数为 3：

```bash
bin/kafka-topics.sh \
  --bootstrap-server "$BOOTSTRAP" \
  --command-config "$ADMIN" \
  --create \
  --topic ads-bz-upgrade-canary \
  --partitions 3 \
  --replication-factor 3
```

验证 topic 不使用 `__` 前缀，避免与 Kafka 内部 topic 或 MM2 默认排除规则混淆。最终上线前会重新初始化数据目录，验证 topic 不迁移到最终集群；如需保留同名监控能力，应在最终集群重建后重新创建。

### 4.4 部署 MM2 新链路

保持旧 `ads -> ads-bz (1.1.1)` MM2 链路不变，新增一条 `ads -> ads-bz (3.9.2)` 链路：

- 源集群：`ads` 1.1.1，地址和凭据沿用现有 MM2 配置。
- 目标集群：`ads-bz (3.9.2)`，bootstrap 使用 8110。
- 方向必须保持单向，不得开启 `ads-bz (3.9.2) -> ads` 的业务数据复制。
- 如果现有灾备语义要求目标 topic 与源 topic 同名，继续使用 Identity Replication Policy。
- topic、partition、consumer group offset 和 topic 配置的同步范围必须与旧备份链路对齐；ACL 不通过 MM2 同步。
- 新旧链路使用不同的 target alias 和独立 MM2 部署，源端 offset-sync topic 与 checkpoint 状态互不共享；最终 target 重建后不迁移验证期 offset，group offset 由 checkpoint connector 重新同步。
- 新链路首次启动应从源 topic 当前最早保留数据开始回放，并显式确认 MM2 的起始 offset 策略。
- 具体命令和配置按现有 MM2 操作手册执行；旧链路保持原来的 3.8.2 独立模式，新链路使用本次验证版本。

部署后检查：

1. MM2 Worker 和 Connector 状态为 RUNNING。
2. 目标集群开始创建 topic 和分区。
3. 心跳或 canary 数据能从 `ads` 到达 `ads-bz (3.9.2)`。
4. MM2 复制 lag 逐步下降，不存在长期卡住的 partition。
5. checkpoint connector 能生成 group offset，并能将目标 group offset 写入新集群。

MM2 通常只会在目标 consumer group 没有活跃成员时同步 group offset。验证 offset 同步时，应先停止测试 consumer，再检查新集群上的 group offset，避免把活跃消费组误判为同步失败。

## 5. 最终上线切换

### 5.1 切换前检查

1. 新集群已完成验证，并经过评审确认进入最终上线窗口。
2. 旧 `ads-bz` 1.1.1 和旧 MM2 3.8.2 独立模式链路当前健康。
3. 新集群 MM2 lag 已追平，并记录切换前源端和目标端 offset。
4. 旧版本回滚资产已经归档并设置为只读，至少包含：
   - Kafka 1.1.1 安装包、`server.properties`、JAAS、启停脚本和服务配置。
   - MM2 3.8.2 安装包、独立模式配置、JAAS、启停脚本和服务配置。
5. 53 台新 broker 和 5 台 Controller 的验证期配置、最终上线配置和 node id 映射均已备份。
6. 最终上线配置已完成评审：端口为 9110 和 9111，broker 数据目录为原来的 `/data*/kafka/data`。
7. 已确认 9110 和 9111 只由旧 `ads-bz (1.1.1)` 占用。
8. 数据目录清理和 Controller 重新初始化已获得变更审批，且确认没有进程继续写入相关目录。
9. 已通知相关人员变更窗口、影响范围、验证命令和回滚条件。

### 5.2 清理、重新初始化与切换

按顺序执行：

1. 停止 `ads -> ads-bz (3.9.2)` 的 MM2 链路和非必要验证客户端，记录源端与目标端 offset。
2. 停止旧 `ads -> ads-bz (1.1.1)` 的 MM2 3.8.2 独立模式链路，记录源端与旧目标端 offset。
3. 对旧 `ads-bz (1.1.1)` 和新 `ads-bz (3.9.2)` 执行 controlled shutdown，确认 9110 和 9111 已释放。
4. 归档验证期配置、日志、offset 快照、ACL 执行记录和问题记录。
5. 按审批清理 Controller 元数据目录 `/data1/kafka-controller-adbz/data`、验证期 broker 目录 `/data*/kafka-adbz/data` 和原 broker 目录 `/data*/kafka/data`。
6. 基于验证期配置生成最终上线配置：broker 端口改为 `9110/9111`，数据目录改回 `/data*/kafka/data`，日志目录改回 `/data0/logs/kafka`；JMX `9893` 和 Prometheus `10013` 保持不变。
7. 参考第 4 章将 Controller 和 broker 完整重新部署一遍：重新生成 `KAFKA_CLUSTER_ID`、五个 Controller directory id 和 `--initial-controllers`，五台 Controller 重新 format 后启动，broker 使用同一个新 cluster id format；配置文件分别使用 `controller-final.properties` 和 `broker-final.properties`。
8. 使用线上 ACL 同步接口向最终集群写入 ACL，并核对执行结果、ACL 数量和抽样授权语义。
9. 将 MM2 目标集群地址改为 `9110`，恢复 `ads -> ads-bz (3.9.2)` 单向同步。
10. 等待 MM2 追平后，使用 1.1.1 客户端和 3.9.2 客户端执行 produce、fetch 和 group offset 验证。
11. 更新服务发现、监控、告警和运维文档中的 `ads-bz (3.9.2)` 地址。

最终上线是一个重新初始化过程，不复用验证期 Controller 元数据、broker 数据、topic 数据和 ACL。重新初始化使用的 cluster id 必须归档；不要把验证期 `meta.properties` 或数据文件复制到最终目录。

从停止旧 MM2 到新 MM2 恢复期间，`ads` 没有可用备份集群。该窗口必须纳入变更时长预算，并提前明确最大可接受停顿时间；超过阈值时应执行回滚。

### 5.3 切换后观察

端口切换后至少观察 24 小时：

- Controller quorum 状态。
- 53 台 broker 存活和 ISR。
- MM2 lag 与 checkpoint。
- 9110 客户端连接数。
- 磁盘空间、IO util、网络带宽、请求延迟。
- broker 与 Controller 日志。
- 旧集群端口没有被其他进程占用。

观察期内旧 Kafka 1.1.1 和旧 MM2 3.8.2 的部署包、配置和服务脚本继续保留，不得删除或覆盖。原 broker 数据目录已被新集群使用，回滚时不能假设旧数据仍可直接复用，需要按回滚方案重新初始化并重新同步。

## 6. 回滚方案

### 6.1 业务切换前回滚

如果最终上线切换前发现问题：

1. 停止 `ads-bz (3.9.2)` 上的 MM2 新链路。
2. 停止新集群 broker 和 Controller。
3. 保持旧 `ads-bz` 1.1.1 原端口、原数据和旧 MM2 3.8.2 独立模式链路不变。
4. 保留新集群日志、配置和 Core dump 供排查。
5. 问题修复并重新验证后，再申请下一次变更窗口。

这是本方案的主要回滚路径。只要尚未执行最终上线切换，旧备份集群没有被动过，回滚影响最小。

### 6.2 端口切换后回滚

端口切换后，如果新集群无法稳定工作：

1. 停止业务或演练流量对新集群的写入。
2. 保留新集群日志、配置、offset、topic 和 ACL 快照。
3. 确认 9110 和 9111 已停止。
4. 使用已归档的 Kafka 1.1.1 部署包和配置重新初始化旧 `ads-bz (1.1.1)` 集群。由于原数据目录已在新上线时清理并被新集群使用，不能直接假设旧数据仍可复用。
5. 使用已归档的 MM2 3.8.2 独立模式部署包和配置恢复 `ads -> ads-bz (1.1.1)` 链路，重新同步数据。
6. 将业务或演练客户端切回旧集群。

如果已经有数据只写入新集群，回滚前必须明确处理方式：重放到旧集群、保留在新集群等待恢复，或经过业务确认后接受丢弃。不能默认认为两个集群数据完全一致。

### 6.3 Controller 故障处理

- 单台 Controller 故障不影响 quorum，可修复后重启。
- 两台 Controller 故障时仍有 3 台存活，quorum 仍可用，但应停止变更操作并优先恢复 Controller。
- 三台及以上 Controller 故障时集群元数据无法形成多数派，应停止变更操作并优先恢复 Controller。
- 修复 Controller 前不要格式化或清空元数据目录。
- 如需重建 Controller，必须基于已有元数据备份和 Kafka 3.9.2 恢复流程单独评审，不能按首次部署流程直接重新 format。
- 本节适用于运行期故障；第 5.2 节中经过审批的最终上线前重新初始化是计划内操作。


