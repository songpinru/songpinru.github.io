# 总览

```shell
spark.shuffle.io.preferDirectBufs #shuffle的堆外内存(netty数据传输),默认的堆外内存就是这个+spark固定的部分(vm,一部分对象创建)
spark.network.io.preferDirectBufs #3.0之后加入,心跳等网络通讯也可以使用堆外内存
spark.memory.offHeap.enabled #默认是false,部分算子可以使用堆外内存
spark.memory.offHeap.size
spark.executor.memoryOverhead #堆外内存,默认的10%
```



rdd：

* 序列化和压缩
* 内存：调整和调优
* shuffle
* 并行度（资源）

streaming：

* 处理时间
* 批处理间隔（分区数）
* 其他

# submit

1. 离线任务

   看队列所能获得的最大资源，

   如：总资源300核，800G内存，

   ​	executor-core=3，设置为2-5，为什么：单HDFS客户端多线程操作会降低吞吐量，造成网络IO降低

   ​	num-executors=300/3=100，

   ​	executor-memroy=800/100=8G，

   ​	driver-memory=4G，设置为1-6G

   ​	master：yarn-cluster

   ​	--conf spark.default.parallelism=600，总核数的2-3倍

   ​	--conf spark.yarn.executor.memoryOverhead=2048 ，堆外内存

   ​	--conf spark.core.connection.ack.wait.timeout=300 ，连接等待时长

   ​	--conf spark.storage.memoryFraction=0.4 ，cache内存占比

   > --conf spark.driver.extraJavaOptions="-XX:PermSize=128M -XX:MaxPermSize=256M"(如果cluster运行不了sql)
   >
   > spark.sql.autoBroadcastJoinThreshold 10M
   >
   > spark.sql.shuffle.partitions  200  


   ```bash
   /usr/local/spark/bin/spark-submit \
   --class com.spring.spark.WordCount \
   --num-executors 80 \
   --driver-memory 6g \
   --executor-memory 6g \
   --executor-cores 3 \
   --master yarn-cluster \
   --queue root.default \
   --conf spark.yarn.executor.memoryOverhead=2048 \
   --conf spark.core.connection.ack.wait.timeout=300 \
   /usr/local/spark/spark.jar
   ```

   

2. 实时任务

   总核数和kafka的分区数一致

   内存一般为4G（按照每批次数据8倍）

   **批处理时间应小于批处理间隔**

   spark.streaming.blockInterval=200ms    ==Partition个数 = BatchInterval / blockInterval==（并行度）（核心数2-5倍）

# 代码中

1. rdd.cache
2. sc.broadcast
3. kryo序列化
4. 本地化等待时间

```scala
val conf = new SparkConf()
  .set("spark.locality.wait", "6")
  .set("spark.scheduler.mode", "FAIR")//多job（行动算子）时使用

SET spark.sql.thriftserver.scheduler.pool=accounting;//变成数据库长联的时候使用，hiveserver2
```

## 算子调优：

1. mapPartition代替map
2. foreachPartition优化数据库连接
3. filter后使用coalesce
4. sql之后使用repartition
5. 多用reduceByKey

## ==Shuffle调优==：

1. map端缓冲区大小，默认32k

```scala
val conf = new SparkConf()
  .set("spark.shuffle.file.buffer", "64")
```

2. reduce端缓冲区大小，默认48M

```scala
val conf = new SparkConf()
  .set("spark.reducer.maxSizeInFlight", "96")
```

3. reduce拉取重试次数，默认3

```scala
val conf = new SparkConf()
  .set("spark.shuffle.io.maxRetries", "6")
```

4. reduce拉取失败等待时长，默认5s

```scala
val conf = new SparkConf()
  .set("spark.shuffle.io.retryWait", "60s")
```

5. ==shufflePartition==

   ```scala
   val conf = new SparkConf()
     .set("spark.sql.shuffle.partitions", "400")
   ```

6. sortShuffle阙值，默认200

```scala
val conf = new SparkConf()
  .set("spark.shuffle.sort.bypassMergeThreshold", "400")
```

## ==环境设置==：

1. 动态分区，非严格模式

   ```scala
   spark.sql("set hive.exec.dynamic.partition=true")
   spark.sql("set hive.exec.dynamic.partition.mode=nonstrict")
   ```

2. 背压，最大消费速率

3. 优雅关闭

4. 首次消费earliest

5. 手动维护偏移量

Kafka：

1. 网络和io的内存、核数
2. producer缓冲区大小
3. producer落盘event条数、时间间隔
4. 压缩
5. 副本数
6. 总内存

Hive：

1. 分区
2. 压缩
3. 列式存储
4. JVM重用
5. MapJoin
6. 小文件合并
7. 动态分区
8. 不用*

yarn：

1. maptask、reducetask的内存、核数
2. contain内存、核数
3. 缓冲区大小，两个
4. 任务失败重试次数、重试间隔