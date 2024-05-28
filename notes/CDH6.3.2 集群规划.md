# CDH6.3.2 集群规划

# 基础

## 硬件信息

|                   | 内存 | 硬盘 | 核数 |
| ----------------- | ---- | ---- | ---- |
| bjuc49.optaim.com | 128G | 10T  | 32   |
| bjuc50.optaim.com | 128G | 10T  | 32   |
| bjuc51.optaim.com | 128G | 10T  | 32   |
| bjuc52.optaim.com | 128G | 10T  | 32   |
| bjuc53.optaim.com | 128G | 10T  | 32   |
| bjuc54.optaim.com | 128G | 10T  | 32   |
| bjuc55.optaim.com | 128G | 10T  | 32   |

## 环境

**系统:**

ubuntu18.04

**jdk:**

oracle-j2sdk1.8-1.8.0+update181-1.x86_64.rpm

**mysql:**

Mariadb-10.1.45

username=root

password=Qwaszx@zxc

> 49和50上的hive库互为主从

**Cloudera Manager:**

Cloudera Express 6.3.1

**CDH:**

CDH6.3.2

```
systemctl restart cloudera-scm-server
```



## CDH包含框架

| Component        | Version |
| :--------------- | :------ |
| Apache Avro      | 1.8.2   |
| Apache Flume     | 1.9.0   |
| Apache Hadoop    | 3.0.0   |
| Apache HBase     | 2.1.4   |
| HBase Indexer    | 1.5     |
| Apache Hive      | 2.1.1   |
| Hue              | 4.3.0   |
| Apache Impala    | 3.2.0   |
| Apache Kafka     | 2.2.1   |
| Kite SDK         | 1.0.0   |
| Apache Kudu      | 1.10.0  |
| Apache Solr      | 7.4.0   |
| Apache Oozie     | 5.1.0   |
| Apache Parquet   | 1.9.0   |
| Parquet-format   | 2.4.0   |
| Apache Pig       | 0.17.0  |
| Apache Sentry    | 2.1.0   |
| Apache Spark     | 2.4.0   |
| Apache Sqoop     | 1.4.7   |
| Apache ZooKeeper | 3.4.5   |

# 集群规划

| 49        | 50       | 51            | 52       | 53        | 54        | 55        |
| --------- | -------- | ------------- | -------- | --------- | --------- | --------- |
| mariadb   | mariadb  |               |          |           |           |           |
| cm_server |          |               |          |           |           |           |
| cm_agent  | cm_agent | cm_agent      | cm_agent | cm_agent  | cm_agent  | cm_agent  |
| namenode  | namenode |               |          |           |           |           |
| datanode  | datanode | datanode      | datanode | datanode  | datanode  | datanode  |
|           |          | RM            | RM       |           |           |           |
| NM        | NM       | NM            | NM       | NM        | NM        | NM        |
|           |          |               |          | zookeeper | zookeeper | zookeeper |
|           |          |               |          | kafka     | kafka     | kafka     |
| hive      | hive     |               |          |           |           |           |
| spark     | spark    | spark_history |          |           |           |           |
| flink     | flink    |               |          |           |           |           |
| sqoop     | sqoop    |               |          |           |           |           |
| kudu      | kudu     |               |          |           |           |           |
|           | impala   |               |          |           |           |           |
| flume     |          |               |          |           |           |           |
| hue       |          |               |          |           |           |           |
| Oozie     |          |               |          |           |           |           |
| sentry    |          |               |          |           |           |           |

# 数据迁移

HDFS数据迁移:

```sh
hadoop distcp -skipcrccheck -update hdfs://10.11.90.12:8020/zyz/ifansb/ hdfs://10.11.20.52:8020/zyz/ifansb/

hadoop distcp -skipcrccheck -update hdfs://10.11.90.12:8020/user/hive/warehouse/ifansb.db hdfs://10.11.20.52:8020/user/hive/warehouse/ifansb.db
```

hive 元数据迁移:

```
第一步先把旧集群得元数据导出来
然后导入新集群的mysql对应的库里
查询升级所需的脚本
./schematool -dbType mysql -upgradeSchemaFrom 1.10 -dryRun -passWord scm -userName scm
再mysql中依次执行所需的脚本,最后补全缺少的table
在CDH中更新hive metaStore namenode
```

hive分区补全

```
msck repair table table_name;
```

权限刷新

```
hdfs dfsadmin -refreshUserToGroupsMappings
```

Hive server2 HA

```xml
<property>
	<name>hive.server2.support.dynamic.service.discovery</name>
	<value>true</value>
</property>
```

JDBC

```
#hive session
hive.spark.client.future.timeout=60s
hive.spark.client.connect.timeout=600s
hive.spark.client.server.connect.timeout=600s

!connect jdbc:hive2://bjuc55.optaim.com:2181,bjuc54.optaim.com:2181,bjuc53.optaim.com:2181/;serviceDiscoveryMode=zooKeeper;zooKeeperNamespace=hiveserver2

beeline --hiveconf hive.server2.logging.operation.level=NONE  -u "jdbc:hive2://10.11.20.55:2181,10.11.20.54:2181,10.11.20.53:2181/;serviceDiscoveryMode=zooKeeper;zooKeeperNamespace=hiveserver2"  -n hive
```

