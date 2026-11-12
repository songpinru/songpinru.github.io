---
title: "MirrorMaker2 On Connect Distributed 操作手册"
---

## 1. 准备

以下步骤默认 Kafka 安装目录为 `/usr/local/kafka/kafka-release`。

```bash
KAFKA_HOME=/usr/local/kafka/kafka-release
CONNECT=http://10.79.40.119:18088
```

## 2. 修改启动脚本

修改 `$KAFKA_HOME/bin/connect-distributed.sh`，替换为以下内容：

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
        echo "USAGE: $0 [-daemon] connect-distributed.properties"
        exit 1
fi

base_dir=$(dirname $0)

if [ "x$KAFKA_LOG4J_OPTS" = "x" ]; then
    export KAFKA_LOG4J_OPTS="-Dlog4j.configuration=file:$base_dir/../config/connect-log4j.properties"
fi

if [ "x$KAFKA_HEAP_OPTS" = "x" ]; then
  export KAFKA_HEAP_OPTS="-Xms256M -Xmx2G"
fi

EXTRA_ARGS=${EXTRA_ARGS-'-name connectDistributed'}

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
export JMX_PORT="8888"
export PROMETHEUS_PORT="18888"
export JAVA_HOME="/usr/local/jdk-17.0.7"
export LOG_DIR=/data0/logs/mm2
export KAFKA_JMX_OPTS="-Dcom.sun.management.jmxremote=true -Dcom.sun.management.jmxremote.authenticate=true -Dcom.sun.management.jmxremote.local.only=false -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.host=0.0.0.0 -Djava.rmi.server.hostname=$IP -Dcom.sun.management.jmxremote.password.file=$base_dir/../config/jmxremote.password -Dcom.sun.management.jmxremote.access.file=$base_dir/../config/jmxremote.access -javaagent:$base_dir/../libs/jmx_prometheus_javaagent-0.20.0.jar=${PROMETHEUS_PORT:-19999}:$base_dir/../config/jmx.yaml"

exec $(dirname $0)/kafka-run-class.sh $EXTRA_ARGS org.apache.kafka.connect.cli.ConnectDistributed "$@"
```


## 3. 修改日志配置

修改 `$KAFKA_HOME/config/connect-log4j.properties`：

```properties
log4j.rootLogger=INFO, connectAppender

log4j.appender.connectAppender=org.apache.log4j.SinaDailyRollingFileAppender
log4j.appender.connectAppender.DatePattern='.'yyyy-MM-dd
log4j.appender.connectAppender.File=${kafka.logs.dir}/connect-mm2.log
log4j.appender.connectAppender.layout=org.apache.log4j.PatternLayout
log4j.appender.connectAppender.layout.ConversionPattern=[%d] %p %X{connector.context}%m (%c:%L)%n
log4j.appender.connectAppender.MaxBackupIndex=20

log4j.logger.org.reflections=ERROR
```

## 4. 配置 Worker

创建 `$KAFKA_HOME/config/connect-mm2.properties`：

```properties
# REST。每台 Worker 使用自己的监听和 advertised 地址。
listeners=http://<本机IP>:18080

# Connect 管理 Kafka：wbbz
bootstrap.servers=10.52.139.55:8110,10.52.136.56:8110,10.52.139.41:8110,10.52.140.40:8110,10.52.141.42:8110

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



# 不开启 exactly-once
exactly.once.source.support=disabled

```

多节点部署时，每台 Worker 只把 `listeners`、`rest.advertised.host.name`、`rest.advertised.port` 改成本机地址，其他配置保持一致。

## 5. 启动 Worker

在每台 Worker 节点执行：

```bash
cd "$KAFKA_HOME"
bin/connect-distributed.sh -daemon config/connect-mm2.properties
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
```

## 6. 配置 Connector

创建 `$KAFKA_HOME/config/mm2-wbads-to-wbbz-source.json`：

```json
{
  "connector.class": "org.apache.kafka.connect.mirror.MirrorSourceConnector",
  "offset.consistency.enabled": "true",
  "tasks.max": "10",
  "replication.policy.class": "org.apache.kafka.connect.mirror.IdentityReplicationPolicy",
  "key.converter": "org.apache.kafka.connect.converters.ByteArrayConverter",
  "value.converter": "org.apache.kafka.connect.converters.ByteArrayConverter",
  "header.converter": "org.apache.kafka.connect.converters.ByteArrayConverter",

  "source.cluster.alias": "wbads",
  "source.cluster.bootstrap.servers": "10.52.141.45:8110,10.52.143.45:8110,10.1.63.44:8110",
  "source.cluster.security.protocol": "SASL_PLAINTEXT",
  "source.cluster.sasl.mechanism": "PLAIN",
  "source.cluster.sasl.jaas.config": "org.apache.kafka.common.security.plain.PlainLoginModule required username=\"admin\" password=\"d4a12dfe3f97e641edd9f206eca5ae92\";",
  "target.cluster.alias": "wbbz",
  "target.cluster.bootstrap.servers": "10.78.31.59:8111,10.79.40.119:8111,10.75.2.189:8111",
  "target.cluster.security.protocol": "SASL_PLAINTEXT",
  "target.cluster.sasl.mechanism": "PLAIN",
  "target.cluster.sasl.jaas.config": "org.apache.kafka.common.security.plain.PlainLoginModule required username=\"admin\" password=\"d4a12dfe3f97e641edd9f206eca5ae92\";",

  "topics": ".*",
  
  "refresh.topics.enabled": "true",
  "refresh.topics.interval.seconds": "60",

  "replication.factor": "3",

  "sync.topic.configs.enabled": "true",
  "sync.topic.acls.enabled": "false",

  "emit.offset-syncs.enabled": "true",
  "offset-syncs.topic.location": "target",
  "offset-syncs.topic.replication.factor": "3",

  "source.consumer.max.partition.fetch.bytes": "8388608"
  
}
```

创建 `$KAFKA_HOME/config/mm2-wbads-to-wbbz-checkpoint.json`：

```json
{
  "connector.class": "org.apache.kafka.connect.mirror.MirrorCheckpointConnector",
  "offset.consistency.enabled": "true",
  "tasks.max": "5",
  "replication.policy.class": "org.apache.kafka.connect.mirror.IdentityReplicationPolicy",
  "key.converter": "org.apache.kafka.connect.converters.ByteArrayConverter",
  "value.converter": "org.apache.kafka.connect.converters.ByteArrayConverter",
  "header.converter": "org.apache.kafka.connect.converters.ByteArrayConverter",

  "source.cluster.alias": "wbads",
  "source.cluster.bootstrap.servers": "10.52.141.45:8110,10.52.143.45:8110,10.1.63.44:8110",
  "source.cluster.security.protocol": "SASL_PLAINTEXT",
  "source.cluster.sasl.mechanism": "PLAIN",
  "source.cluster.sasl.jaas.config": "org.apache.kafka.common.security.plain.PlainLoginModule required username=\"admin\" password=\"d4a12dfe3f97e641edd9f206eca5ae92\";",
  "target.cluster.alias": "wbbz",
  "target.cluster.bootstrap.servers": "10.78.31.59:8111,10.79.40.119:8111,10.75.2.189:8111",
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

  "offset-syncs.topic.location": "target"
}
```

## 7. 启动 Connector

提交 Source Connector：

```bash
curl -fsS -X PUT "$CONNECT/connectors/mm2-wbads-to-wbbz-source/config" \
  -H 'Content-Type: application/json' \
  --data-binary @"$KAFKA_HOME/config/mm2-wbads-to-wbbz-source.json"
```

提交 Checkpoint Connector：

```bash
curl -fsS -X PUT "$CONNECT/connectors/mm2-wbads-to-wbbz-checkpoint/config" \
  -H 'Content-Type: application/json' \
  --data-binary @"$KAFKA_HOME/config/mm2-wbads-to-wbbz-checkpoint.json"
```

## 8. 验证

查看 Connector 状态：

```bash
curl -fsS "$CONNECT/connectors/mm2-wbads-to-wbbz-source/status"
curl -fsS "$CONNECT/connectors/mm2-wbads-to-wbbz-checkpoint/status"
```

`connector.state` 和 `tasks[].state` 应为 `RUNNING`。

查看日志：

```bash
tail -f /data0/logs/mm2/connect.log
```

常用操作：

```bash
# 查看全部 Connector
curl -fsS "$CONNECT/connectors?expand=status"

NAME=mm2-wbads-to-wbbz-source

curl -fsS "$CONNECT/connectors/$NAME/status"

# 重启失败的 Connector 和 Task

curl -fsS -X POST "$CONNECT/connectors/$NAME/restart?includeTasks=true&onlyFailed=true"

curl -fsS -X PUT "$CONNECT/connectors/$NAME/pause"
curl -fsS -X PUT "$CONNECT/connectors/$NAME/resume"
# 删除 Connector（慎用）
curl -fsS -X DELETE "$CONNECT/connectors/$NAME"

curl -fsS "$CONNECT/connectors/$NAME/offsets"

```
