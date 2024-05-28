# Spark配置

##  yarn模式
### spark-env.sh
```sh
YARN_CONF_DIR=/opt/module/hadoop-2.7.2/etc/hadoop
export SPARK_HISTORY_OPTS="
-Dspark.history.ui.port=18080 
-Dspark.history.fs.logDirectory=hdfs://hadoop102:9000/directory 
-Dspark.history.retainedApplications=30"
```



### spark-defalts.conf
```bash
spark.eventLog.enabled           true
spark.eventLog.dir               hdfs://hadoop102:9000/directory
spark.yarn.historyServer.address=hadoop102:18080
spark.history.ui.port=18080

##
spark.eventLog.compress		true
spark.io.compression.codec	snappy
spark.checkpoint.compress	true
spark.rdd.compress			true
spark.serializer			org.apache.spark.serializer.KryoSerializer
## spark.kryo.unsafe  true 不安全，提高性能时用
## spark.kryoserializer.buffer.max  64M  如果在Kryo中收到“超出缓冲区限制”异常，请增加此值
## spark.scheduler.mode		FIFO/FAIR
## spark.speculation 推测执行
```
### yarn-site.xml
```xml
<property>
    <name>yarn.log.server.url</name>
    <value>http://hadoop102:19888/jobhistory/logs</value>
</property>
```
---
## standalone模式
### spark-env.sh 
```shell
-Dspark.history.ui.port=18080 
-Dspark.history.fs.logDirectory=hdfs://hadoop102:9000/directory 
-Dspark.history.retainedApplications=30"
export SPARK_DAEMON_JAVA_OPTS="
-Dspark.deploy.recoveryMode=ZOOKEEPER 
-Dspark.deploy.zookeeper.url=hadoop102,hadoop103,hadoop104 
-Dspark.deploy.zookeeper.dir=/spark"
```
### spark-defalts.conf
```
spark.master                     spark://hadoop102:7077
spark.eventLog.enabled           true
spark.eventLog.dir               hdfs://hadoop102:9000/directory
```
### slaves
```
hadoop102
hadoop103
hadoop104
```
---
## submit

```shell
# submit参数
spark-submit \
  --master local[5]  \
  --driver-cores 2   \
  --driver-memory 8g \
  --executor-cores 4 \
  --num-executors 10 \
  --executor-memory 8g \
  --class PackageName.ClassName XXXX.jar \
  --name "Spark Job Name" \
  InputPath      \
  OutputPath

# 示例
bin/spark-submit \
--class org.apache.spark.examples.SparkPi \
--master spark://hadoop102:7077 \
--executor-memory 2G \
--total-executor-cores 2 \
./examples/jars/spark-examples_2.11-2.1.1.jar \
10

```

# Spark core

POM添加：

```xml
<dependency>
        <groupId>org.apache.spark</groupId>
        <artifactId>spark-core_2.11</artifactId>
        <version>2.1.1</version>
</dependency>
```

## RDD创建

```scala
//创建SparkConf并设置App名称
val conf: SparkConf = new SparkConf().setAppName("SparkCoreTest").setMaster("local[*]")

//创建SparkContext，该对象是提交Spark App的入口
val sc: SparkContext = new SparkContext(conf)

//1.使用parallelize()创建rdd
val rdd: RDD[Int] = sc.parallelize(Array(1, 2, 3, 4, 5, 6, 7, 8))

//2.使用makeRDD()创建rdd
val rdd1: RDD[Int] = sc.makeRDD(Array(1, 2, 3, 4, 5, 6, 7, 8))

//3.读取文件。如果是集群路径：hdfs://hadoop102:9000/input
val lineWordRdd: RDD[String] = sc.textFile("input")
```

## RDD分区

> 默认分区数(local[*])是CPU的核心数

> 从集合生成RDD的分区规则是：end=(i * length/num)
>
> 从文件生成RDD的分区规则是Hadoop的切片规则：
>
> goalsize=totalsize/numPartitions，
>
> minPartitions默认是2和defalt取最小
>
> 按goalsize截取（若剩余字节大于1.1倍totalsize），剩余的放在一个分区

```scala
def makeRDD[T: ClassTag](seq: Seq[T],numSlices: Int = defaultParallelism): RDD[T] 
= withScope {
  parallelize(seq, numSlices)
}


	defaultParallelism 	= cpu核心数(local[*])


 def positions(length: Long, numSlices: Int): Iterator[(Int, Int)] = {
      (0 until numSlices).iterator.map { i =>
        val start = ((i * length) / numSlices).toInt
        val end = (((i + 1) * length) / numSlices).toInt
        (start, end)
      }
    }
```

```scala
def textFile(path: String,
			minPartitions: Int = defaultMinPartitions): RDD[String] = withScope {
	assertNotStopped()
  hadoopFile(path, classOf[TextInputFormat], classOf[LongWritable], classOf[Text],
    minPartitions).map(pair => pair._2.toString).setName(path)
}

def defaultMinPartitions: Int = math.min(defaultParallelism, 2)

while (((double) bytesRemaining)/splitSize > SPLIT_SLOP) {
            String[] splitHosts = getSplitHosts(blkLocations,
                length-bytesRemaining, splitSize, clusterMap);
            splits.add(makeSplit(path, length-bytesRemaining, splitSize,
                splitHosts));
            bytesRemaining -= splitSize;
          }

if (bytesRemaining != 0) {
            String[] splitHosts = getSplitHosts(blkLocations, length
                - bytesRemaining, bytesRemaining, clusterMap);
            splits.add(makeSplit(path, length - bytesRemaining, bytesRemaining,
                splitHosts));
          }
```

## spark算子

### Transformation 转换算子

#### Value型

| name                                    | description                              |
| --------------------------------------- | ---------------------------------------- |
| map()                                   | 映射                                     |
| mapPartitions()                         | map的批处理型，提高效率，内存压力大      |
| mapPartitionsWithIndex()                | key值为分区号的mapPartitions             |
| flatmap()                               | map后扁平化                              |
| glom()                                  | 分区变为数组                             |
| groupBy()                               | 分组(不能写\_，展开为e=>e                |
| filter()                                | 过滤(判断条件为boolean)                  |
| sample(withReplacement, fraction, seed) | 取样(是否放回，期望：大致样本比例，种子) |
| distinct()                              | 去重底层是map和reduceByKey               |
| coalesce()                              | 重分区(num,shuffle)                      |
| repartition()                           | =coalesce(num,true)                      |
| sortBy(f,ascending, numPartitions)      | 排序                                     |
| pipe(command)                           | 管道，脚本调用                           |

#### Value型交互

| name           | description                            |
| -------------- | -------------------------------------- |
| union()        | 并集，无shuffle                        |
| substract()    | 差集，无shuffle                        |
| intersection() | 交集，无shuffle                        |
| zip()          | 拉链(要求严格，每个分区的元素数要相等) |

#### Key-Value型

| name             | description                                                  |
| ---------------- | ------------------------------------------------------------ |
| partitionBy()    | 分区，可自定义分区器                                         |
| groupByKey()     | 根据key分组                                                  |
| reduceByKey()    | 根据key聚合(尽量用这个)                                      |
| flodByKey()      | 有初始值的reduceByKey                                        |
| aggregateByKey() | (零值)(分区内函数，分区间函数)                               |
| combineByKey()   | (首个值转化，分区内函数，分区间函数)                         |
| sortByKey()      | 按照key值排序，key须混入ordered特质和serializable            |
| mapValues()      | 只映射values                                                 |
| join()           | 按照key内连接(相通key的两个聚合成元组)                       |
| cogroup()        | 每个分区内按照key把value聚合为集合，然后把集合变成元组(shuffle) |

**reduceByKey()	flodByKey()	aggregateByKey()	combineByKey()**

>   这四个底层都调用了**combineByKeyWithClassTag**,
>   >**combineByKeyWithClassTag**：（首个值转化，分区内函数，分区间函数）
>   >
>   >**combineByKey**和combineByKeyWithClassTag用法一致，只是套了壳
>   >**aggregateByKey**是combineByKeyWithClassTag 将零值提出来进行了柯里化
>   >**flodByKey**，首个值转化，分区内函数=分区间函数
>   >**reducerByKey**，分区内函数=分区间函数

### Action行动算子

| name                 | description                                                  |
| -------------------- | ------------------------------------------------------------ |
| collect()            | 将RDD转换为数组返回给drive                                   |
| count()              | 返回RDD中元素个数(Long)                                      |
| first()              | 返回RDD中第一个元素                                          |
| take()               | 返回RDD的前n个元素(数组)                                     |
| takeOrder()          | 返回RDD排序后的前n个元素(数组)                               |
| top()                | takeOrdered(num)(ord.reverse)                                |
| countByKey()         | 返回每个key的value个数(数组)                                 |
| foreach(f)           | 在每个分区内分别遍历每个元素                                 |
| foreachPartition()   | 在每个分区内分别遍历                                         |
| reduce()             | 聚合(~~零值~~)(分区内函数=分区间函数)                        |
| flod()               | 聚合 (零值)(分区内函数=分区间函数)(零值在分区间聚合时还要执行一遍) |
| aggregate()          | 聚合(零值)(分区内函数，分区间函数) (零值在分区间聚合时还要执行一遍) |
| saveAsTextFile()     |                                                              |
| saveAsSequenceFile() | 只支持K-V结构                                                |
| saveAsObjectFile     |                                                              |

## RDD序列化和血缘依赖

class也需要继承**Serializable**接口

```scala
new SparkConf().setAppName("SerDemo")
               .setMaster("local[*]")
// 替换默认的序列化机制
.set("spark.serializer","org.apache.spark.serializer.KryoSerializer")
// 注册需要使用 kryo 序列化的自定义类
.registerKryoClasses(Array(classOf[Searcher]))
```

血缘关系

**rdd.toDebugString**

```
(2) ShuffledRDD[4] at reduceByKey at Lineage01.scala:27 []
 +-(2) MapPartitionsRDD[3] at map at Lineage01.scala:23 []
    |  MapPartitionsRDD[2] at flatMap at Lineage01.scala:19 []
    |  input/1.txt MapPartitionsRDD[1] at textFile at Lineage01.scala:15 []
    |  input/1.txt HadoopRDD[0] at textFile at Lineage01.scala:15 []

```

依赖关系

**rdd.dependencies**

```
List(org.apache.spark.OneToOneDependency@f2ce6b)
----------------------
List(org.apache.spark.OneToOneDependency@692fd26)
----------------------
List(org.apache.spark.OneToOneDependency@627d8516)
----------------------
List(org.apache.spark.ShuffleDependency@a518813)

```
## RDD持久化

### RDD Cache

```scala
mapRdd.cache()
def cache(): this.type = persist()
def persist(): this.type = persist(StorageLevel.MEMORY_ONLY)

object StorageLevel {
  val NONE = new StorageLevel(false, false, false, false)
  val DISK_ONLY = new StorageLevel(true, false, false, false)
  val DISK_ONLY_2 = new StorageLevel(true, false, false, false, 2)
  val MEMORY_ONLY = new StorageLevel(false, true, false, true)
  val MEMORY_ONLY_2 = new StorageLevel(false, true, false, true, 2)
  val MEMORY_ONLY_SER = new StorageLevel(false, true, false, false)
  val MEMORY_ONLY_SER_2 = new StorageLevel(false, true, false, false, 2)
  val MEMORY_AND_DISK = new StorageLevel(true, true, false, true)
  val MEMORY_AND_DISK_2 = new StorageLevel(true, true, false, true, 2)
  val MEMORY_AND_DISK_SER = new StorageLevel(true, true, false, false)
  val MEMORY_AND_DISK_SER_2 = new StorageLevel(true, true, false, false, 2)
  val OFF_HEAP = new StorageLevel(true, true, true, false, 1)

```

方法名为 **cache()**，底层调用的是 **persist()**，有多个等级（_2代表缓存数量2，提高安全性）

取消缓存方法为：**unpersist()**

### RDD Checkpoint

```scala
// 需要设置路径，否则抛异常：Checkpoint directory has not been set in the SparkContext
        sc.setCheckpointDir("./checkpoint1")
// 增加缓存,避免再重新跑一个job做checkpoint(推荐)
        wordToOneRdd.cache()
// 数据检查点：针对wordToOneRdd做检查点计算
        wordToOneRdd.checkpoint()
// spark2.2之后可以提取检查点
		sc.checkpointFile()
```

### Cache 和 Checkpoint 的区别

* cache是临时性的，app结束就删除，checkpoint是永久的，通常存在hdfs等文件系统
* cache不切断血缘，checkpoint会切断血缘，后续都从checkpoint读取
* cache和checkpoint存储内容一样，源码显示：若有cache，从cache中读取数据，不用再走一遍job

### checkpoint存到HDFS

```scala
// 设置访问HDFS集群的用户名
        System.setProperty("HADOOP_USER_NAME","atguigu")

// 需要设置路径.需要提前在HDFS集群上创建/checkpoint路径
        sc.setCheckpointDir("hdfs://hadoop102:9000/checkpoint")


```

## 分区

### Hash分区

```scala
class HashPartitioner(partitions: Int) extends Partitioner {
    require(partitions >= 0, s"Number of partitions ($partitions) cannot be negative.")
    
    def numPartitions: Int = partitions
    //分区规则
    def getPartition(key: Any): Int = key match {
        case null => 0
        case _ => Utils.nonNegativeMod(key.hashCode, numPartitions)
    }
    
    override def equals(other: Any): Boolean = other match {
        case h: HashPartitioner =>
            h.numPartitions == numPartitions
        case _ =>
            false
    }
    
    override def hashCode: Int = numPartitions
}

```



### Range分区

分区内无序，分区间有序

### 自定义分区

```scala
//继承Partitioner,重写两个方法
// 设置的分区数
override def numPartitions: Int = num

// 具体分区逻辑
override def getPartition(key: Any): Int = {}

```

##  数据保存

* TextFile
* JsonFile
* SequenceFile
* ObjectFile

### HDFS

由于Hadoop的API有新旧两个版本，Spark也提供了两套创建操作接口**hadoopRDD**和**newHadoopRDD**

### MySQL

就是JDBC

PS：建议使用 foreachPartition()来减少资源消耗

```scala
//定义连接mysql的参数
   val driver = "com.mysql.jdbc.Driver"
   val url = "jdbc:mysql://hadoop102:3306/rdd"
   val userName = "root"

//创建JdbcRDD
   val rdd = new JdbcRDD(sc, () => {
     Class.forName(driver)
     DriverManager.getConnection(url, userName, passWd)
   },
     "select * from `rddtable` where `id`>=? and `id`<=?;",
     1,
     10,
     1,
     r => (r.getInt(1), r.getString(2))
   )

```

## 累加器

累加器：分布式共享只写变量。（Executor和Executor之间不能读数据）

### 系统累加器

```scala
//声明累加器
	val sum1: LongAccumulator = sc.longAccumulator("sum1")
	val sum1: LongAccumulator = sc.doubleAccumulator("sum1")
	val sum1: LongAccumulator = sc.collectionAccumulator("sum1")
// 使用累加器
	sum1.add(count)
// 获取累加器
	sum1.value
```

### 自定义累加器

声明累加器

1. 继承AccumulatorV2,设定输入、输出泛型
2. 重新方法
3. sc注册累加器：sc.register(mc)

```scala
class MyAccumulator extends AccumulatorV2[String, mutable.Map[String, Long]] {

    // 定义输出数据集合
    var map = mutable.Map[String, Long]()

    // 是否为初始化状态，如果集合数据为空，即为初始化状态
    override def isZero: Boolean = map.isEmpty

    // 复制累加器
    override def copy(): AccumulatorV2[String, mutable.Map[String, Long]] = {
        new MyAccumulator()
    }

    // 重置累加器
    override def reset(): Unit = map.clear()

    // 增加数据
    override def add(v: String): Unit = {
        // 业务逻辑
        if (v.startsWith("H")) {
            map(v) = map.getOrElse(v, 0L) + 1L
        }
    }

    // 合并累加器
    override def merge(other: AccumulatorV2[String, mutable.Map[String, Long]]): Unit = {

        var map1 = map
        var map2 = other.value

        map = map1.foldLeft(map2)(
            (map,kv)=>{
                map(kv._1) = map.getOrElse(kv._1, 0L) + kv._2
                map
            }
        )
    }

    // 累加器的值，其实就是累加器的返回结果
    override def value: mutable.Map[String, Long] = map
}
```
>  源码中，每个task执行copyReset（copy+reset）

## 广播变量

广播变量：分布式共享只读变量

使用广播变量步骤：

1. 通过对一个类型T的对象调用SparkContext.broadcast创建出一个Broadcast[T]对象，任何可序列化的类型都可以这么实现。

2. 通过value属性访问该对象的值（在Java中为value()方法）。

3. 变量只会被发到各个节点一次，应作为只读值处理（修改这个值不会影响到别的节点）。

```scala
val list: List[(String, Int)] = List(("a", 4), ("b", 5), ("c", 6))
// 声明广播变量
val broadcastList: Broadcast[List[(String, Int)]] = sc.broadcast(list)
// 使用广播变量
fbroadcastList.value
```

```scala
// 广播文件，将文件夹传到各个节点
sc = spark.sparkContext
sc.addFile("tools/",recursive=True)
sc.addFile("rule_set/",recursive=True)
// 那会广播文件
SparkFiles.get(fileName)
```



#  Spark SQL

## 概念

**SparkSQL是spark用于结构化数据的处理模块**

特点：

1. **易整合**：无缝适应SQL
2. **统一的数据访问方式**：相同方式连接不同的数据源
3. **兼容hive**：自带hive，也可以配置使用已有hive
4. **标准的数据连接**：JDBC/ODBC

API：

* **DataFrame**：以RDD为基础的分布式数据集，添加了元信息(表头)的概念
* **DataSet**：DataFrame的扩展，添加了数据与类的联系
* **RDD**：最底层的分布式数据集

> DataFrame是DataSet的特例，泛型为Row

## Spark SQL编程

**SparkSession**：实质是SQLContext和HiveContext的组合

POM添加：

```xml
<dependency>
<groupId>org.apache.spark</groupId>
<artifactId>spark-sql_2.11</artifactId>
<version>2.1.1</version>
</dependency>
```
------------------------------------------------------------

```scala
val conf: SparkConf = new SparkConf().setMaster("local[*]").setAppName("SparkSQL01_Demo")

//创建SparkSession对象
    val spark: SparkSession = SparkSession.builder().config(conf).getOrCreate()
//RDD=>DataFrame=>DataSet转换需要引入隐式转换规则，否则无法转换
//spark不是包名，是上下文环境对象名
    import spark.implicits._

//读取json文件 创建DataFrame  {"username": "lisi","age": 18}
    val df: DataFrame = spark.read.json("test.json")

```


### DataFrame

#### 创建DataFrame

spark.read.json("path").show

Seq.toDF

#### DataFrame转换为RDD

import spark.implicits._

df.rdd.collect

#### DataFrame转换为DataSet

case class 类名()

df.as[类名]

#### SQL语法和DSL语法

**SQL:**

df.createOrReplaceTempView("people")

spark.sql(" ")

**DSL:**

df.printSchema

df.select("name").show()

df.select($"name",$"age"+1).show

df.filter($"age">19).show

df.groupBy("age").count.show

> 涉及运算时，每列前加$

----------------------------------------

### DataSet

#### 创建DataSet

case class 类名()

Seq(类).toDS

#### DataSet转换为DataFrame

ds.toDF

#### DataSet转换为RDD

import spark.implicits._

ds.rdd

--------------------------

### RDD

```scala
spark.sparkContext.makeRDD()
spark.sparkContext.textFile()
```

#### RDD转换为DataFrame

import spark.implicits._

case class 类名()

**RDD内为元组**：rdd.toDF(表头)

**RDD内为类的对象**：rdd.toDF

spark.createDataFrame(RDD, StructType)

#### RDD转换为DataSet

import spark.implicits._

case class 类名()

**RDD内为类的对象**：rdd.toDS

---------------------------

### 用户自定义函数

#### UDF

spark.udf.register("naem",function)

> function:必须有类型，如   (s:Long)=>s+1L

#### UDAF

##### 第一种方式，弱类型

* 定义类继承UserDefinedAggregateFunction，并重写其中方法

* spark.udf.register("naem",function)

```scala
class MyAveragUDAF extends UserDefinedAggregateFunction {

  // 聚合函数输入参数的数据类型
  def inputSchema: StructType = StructType(Array(StructField("age",IntegerType)))

  // 聚合函数缓冲区中值的数据类型(age,count)
  def bufferSchema: StructType = {
    StructType(Array(StructField("sum",LongType),StructField("count",LongType)))
  }

  // 函数返回值的数据类型
  def dataType: DataType = DoubleType

  // 稳定性：对于相同的输入是否一直返回相同的输出。
  def deterministic: Boolean = true

  // 函数缓冲区初始化
  def initialize(buffer: MutableAggregationBuffer): Unit = {
    // 存年龄的总和
    buffer(0) = 0L
    // 存年龄的个数
    buffer(1) = 0L
  }

  // 更新缓冲区中的数据
  def update(buffer: MutableAggregationBuffer,input: Row): Unit = {
    if (!input.isNullAt(0)) {
      buffer(0) = buffer.getLong(0) + input.getInt(0)
      buffer(1) = buffer.getLong(1) + 1
    }
  }

  // 合并缓冲区
  def merge(buffer1: MutableAggregationBuffer,buffer2: Row): Unit = {
    buffer1(0) = buffer1.getLong(0) + buffer2.getLong(0)
    buffer1(1) = buffer1.getLong(1) + buffer2.getLong(1)
  }

  // 计算最终结果
  def evaluate(buffer: Row): Double = buffer.getLong(0).toDouble / buffer.getLong(1)
}
```
##### 第二种方式，强类型

* 定义样例类，为函数的输入输出缓存类型
* 定义类继承Aggregator，实现其方法
* 需要DataSet来使用该函数，DSL风格语言
* MyAvgUDAF.toColumn
```scala
//输入数据类型
case class User(username:String,age:Long){}

//缓存类型
case class AgeBuffer(var sum:Long,var count:Long) {}

//自定义UDAF函数 强类型
//  IN  输入数据类型
//  BUF 缓存数据类型
//  OUT 输出数据类型
class MyAvgUDAF extends Aggregator[User, AgeBuffer, Double]{

  //缓存初始化
  override def zero: AgeBuffer = {
    AgeBuffer(0L,0L)
  }

  //聚合
  override def reduce(b: AgeBuffer, a: User): AgeBuffer = {
    b.sum += a.age
    b.count += 1
    b
  }

  //合并
  override def merge(b1: AgeBuffer, b2: AgeBuffer): AgeBuffer = {
    b1.sum += b2.sum
    b1.count += b2.count
    b1
  }

  //求值
  override def finish(reduction: AgeBuffer): Double = {
    reduction.sum.toDouble/reduction.count
  }

  //指定DataSet的编码方式  用于进行序列化   固定写法
  //如果是简单类型，就使用系统自带的；如果说，是自定义类型的话，那么就使用product
  override def bufferEncoder: Encoder[AgeBuffer] = {
    Encoders.product
  }

  override def outputEncoder: Encoder[Double] = {
    Encoders.scalaDouble
  }
}
```
## SparkSQL数据加载与保存

### 文件的加载保存

**加载数据的通用方法**：saprk.read.load 

```shell
scala> spark.read.

csv   format   jdbc   json   load   option   options   orc   parquet   schema   table   text   textFile
```

spark.read.format("...")[.option("...")].load("...")

* format("…")："csv"、"jdbc"、"json"、"orc"、"parquet"和"textFile"

* option("…")：JDBC参数，url、user、password和dbtable

* load("...")：文件路径

```
spark.read.format("jdbc")
      .option("url", "jdbc:mysql://hadoop202:3306/test")
      .option("driver", "com.mysql.jdbc.Driver")
      .option("user", "root")
      .option("password", "123456")
      .option("dbtable", "user")
      .load()
      .show
```

```sql
select * from json.`/opt/module/spark-local/people.json`
```

----------------------

**保存数据的通用方式**：df.write.save

````
scala> df.write.
csv  jdbc   json  orc   parquet textFile… …
````

df.write.format("...")[.option("...")].save("...")

* format("…")："csv"、"jdbc"、"json"、"orc"、"parquet"和"textFile"

* option("…")：JDBC参数，url、user、password和dbtable

* load("...")：文件路径

```scala
ds.write.format("jdbc")
        .option("url", "jdbc:mysql://hadoop202:3306/test")
        .option("user", "root")
        .option("password", "123456")
        .option("dbtable", "user")
        .mode(SaveMode.Append)
        .save()
```

------------------------------
POM中添加：
```xml
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>5.1.27</version>
</dependency>
```

连接JDBC的其他两种方式：

* 用options代替option，把连接参数放入Map集合中
* 直接调用 jdbc()方法，把连接参数放入Properties中

```scala
//从MySQL中读取数据 方式2:
spark.read.format("jdbc").options(
        Map(
          "url"->"jdbc:mysql://hadoop202:3306/test?user=root&password=123456",
          "dbtable"->"user",
          "driver"->"com.mysql.jdbc.Driver"
        )
      ).load.show
//从MySQL中读取数据 方式3
    val props: Properties = new Properties()
    props.setProperty("driver","com.mysql.jdbc.Driver")
    props.setProperty("user","root")
    props.setProperty("password","123456")
    spark.read.jdbc("jdbc:mysql://hadoop202:3306/test","user",props).show()
    */
```

SaveMode:

| scala/java                      | other       | description                |
| :------------------------------ | ----------- | :------------------------- |
| SaveMode.ErrorIfExists(default) | "error"     | 如果文件已经存在则抛出异常 |
| SaveMode.Append                 | "append"    | 如果文件已经存在则追加     |
| SaveMode.Overwrite              | "overwrite" | 如果文件已经存在则覆盖     |
| SaveMode.Ignore                 | "ignore"    | 如果文件已经存在则忽略     |

----------------------------

Spark SQL的默认数据源为Parquet格式,不需要使用format

修改配置项spark.sql.sources.default，可修改默认数据源格式

### Hive

自带hive,在目录下会生成元数据目录

启用外部Hive:

* 拷贝hive-site.xml到conf/
* MySQL驱动拷贝至jars/
* 不可用TEZ（从hive-site.xml删除）

POM中添加：

```xml
 <dependency>
 	<groupId>org.apache.spark</groupId>
    <artifactId>spark-hive_2.11</artifactId>
    <version>2.1.1</version>
</dependency>
<dependency>
	<groupId>org.apache.hive</groupId>
	<artifactId>hive-exec</artifactId>
	<version>1.2.1</version>
</dependency>

<!-- 记得拷贝hive-site.xml到resources目录 -->
```

```scala
//创建SparkSession
val spark: SparkSession = SparkSession
      .builder()
      .enableHiveSupport()//添加Hive支持
      .master("local[*]")
      .appName("SQLTest")
      .getOrCreate()

```

# Spark Streaming

## 概念

**流式数据的特点：**

* 数据快速连续到达，潜在数据量大小巨大
* 数据来源多，格式复杂
* 数据量大，但不重视存储，处理后被丢弃或者归档存储
* 注重数据整体价值，不关注个别数据（丢失一点数据不影响）

**Spark Streaming：**Spark用于处理流式数据的模块

**DStreams：**高级抽象的离散化流，底层是RDD的一种用于处理流式数据的分布式数据集

**Spark Streaming 的特点：**

* 易用
* 高容错
* 与Spark生态的整合
* 缺点：**微量批处理**架构，相对于Strom这种*”一次处理一条数据“*的架构，延迟会高一点

**背压机制：**

* spark.streaming.receiver.maxRate ，可以限制接受速率
* spark.streaming.backpressure.enabled ，控制是否启用背压机制，默认为false

## DStream


使用supervisor监控进程，使其挂了可以自动重启，保证项目运行

POM添加：

```xml
<dependency>
	<groupId>org.apache.spark</groupId>
    <artifactId>spark-streaming_2.11</artifactId>
    <version>2.1.1</version>
</dependency>
```

```scala
//创建配置文件对象 注意：Streaming程序至少不能设置为local，至少需要2个线程
    val conf: SparkConf = new SparkConf().setAppName("Spark01_W").setMaster("local[*]")
//创建Spark Streaming上下文环境对象
    val ssc = new StreamingContext(conf,Seconds(3))
//Duration：毫秒值；通常用Seconds()或Minutes()


//启动采集器
    ssc.start()
//默认情况下，上下文对象不能关闭
	//ssc.stop()
//等待采集结束，终止上下文环境对象
    ssc.awaitTermination()
```

## 创建DStream

### 从端口中获取

**netcat指令：**

* sudo yun install -y nc

* nc -lk 9999

```scala
    val conf: SparkConf = new SparkConf().setAppName("Spark01_W").setMaster("local[*]")
    val ssc = new StreamingContext(conf,Seconds(3))
    //操作数据源-从端口中获取一行数据


    val socketDS: ReceiverInputDStream[String] = ssc.socketTextStream("hadoop202",9999)
	//操作ds：


	//输出结果   注意：调用的是DS的print函数
    reduceDS.print()
    //启动采集器
    ssc.start()
    //默认情况下，上下文对象不能关闭
    //ssc.stop()
    //等待采集结束，终止上下文环境对象
    ssc.awaitTermination()

```

### Kafka数据源

POM添加：

```xml
<dependency>
    <groupId>org.apache.spark</groupId>
    <artifactId>spark-streaming-kafka-0-8_2.11</artifactId>
    <version>2.1.1</version>
</dependency>
```

```shell
# 查看topic是否存在
bin/kafka-topics.sh --zookeeper hadoop102:2181 --list
# 创建topic
bin/kafka-topics.sh --zookeeper hadoop102:2181 --create --topic my-bak --partitions 3 --replication-factor 2
# 生产消息
bin/kafka-console-producer.sh --broker-list hadoop102:9092 --topic my-bak
```

#### 高级API方式一

```scala
//kafka参数声明
    val brokers = "hadoop202:9092,hadoop203:9092,hadoop204:9092"
    val topic = "my-bak"
    val group = "bigdata"
    val deserialization = "org.apache.kafka.common.serialization.StringDeserializer"
    val kafkaParams = Map(
      ConsumerConfig.GROUP_ID_CONFIG -> group,
      ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG -> brokers,
      ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG -> deserialization,
      ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG -> deserialization
    )
    //创建DS
    val kafkaDS: InputDStream[(String, String)] = KafkaUtils.createDirectStream[String, String, StringDecoder, StringDecoder](ssc, kafkaParams, Set(topic))
```

#### 高级API方式二

* 实质就是嵌套一层，将逻辑放在函数中，main只提供入口
* 调用StreamingContext.getActiveOrCreate("检查点路径",函数)
* 函数中实现主要逻辑

```scala
// main方法只提供入口
def main(args: Array[String]): Unit = {
    val ssc: StreamingContext = StreamingContext.getActiveOrCreate("check-point",createSSC())
    ssc.start()
    ssc.awaitTermination()
  }

// 主要逻辑，new StreamingContext，checkpoint
  def createSSC(): StreamingContext = {
    val conf: SparkConf = new SparkConf().setMaster("local[*]").setAppName("HighKafka")
    val ssc = new StreamingContext(conf, Seconds(3))
    // 偏移量保存在 checkpoint 中, 可以从上次的位置接着消费
    ssc.checkpoint("D:\\dev\\workspace\\my-bak\\spark-bak\\cp")
    //kafka参数声明
    val brokers = "hadoop202:9092,hadoop203:9092,hadoop204:9092"
    val topic = "my-bak"
    val group = "bigdata"
    val deserialization = "org.apache.kafka.common.serialization.StringDeserializer"
    val kafkaParams = Map(
      "zookeeper.connect" -> "hadoop202:2181,hadoop203:2181,hadoop204:2181",
      ConsumerConfig.GROUP_ID_CONFIG -> group,
      ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG -> brokers,
      ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG -> deserialization,
      ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG -> deserialization
    )
    //创建DS
    val kafkaDS = KafkaUtils.createDirectStream[String, String, StringDecoder, StringDecoder]( ssc, kafkaParams, Set(topic))
    //wordCount
    val resDS: DStream[(String, Int)] = kafkaDS.map(_._2).flatMap(_.split(" ")).map((_,1)).reduceByKey(_+_)
    //打印输出
    resDS.print()
    ssc
  }
```

####   低级API方式

##### 主要思路

在高级API方式一的基础上增加：

* 创建DS前：
  * 创建   **KafkaCluster**   的对象
  * 从zk获取offset： **Map[TopicAndPartition, Long]**
* 数据消费完：
  * 提交维护offset（实际上是调用DS的输出方法：**foreachRDD**）

```scala
//创建KafkaCluster（维护offset）
    val kafkaCluster = new KafkaCluster(kafkaPara)

//获取ZK中保存的offset
    val fromOffset: Map[TopicAndPartition, Long] = getOffsetFromZookeeper(kafkaCluster, group, Set(topic))

//读取kafka数据创建DStream
    val kafkaDStream: InputDStream[String] = KafkaUtils.createDirectStream[String, String, StringDecoder, StringDecoder, String](ssc,
      kafkaPara,
      fromOffset,
      (x: MessageAndMetadata[String, String]) => x.message())

//数据处理
    kafkaDStream.print

//提交offset
    offsetToZookeeper(kafkaDStream, kafkaCluster, group)
```

##### 从ZK获取offset：

* 创建  **本次**  获取offset的容器：new mutable.**Map\[TopicAndPartition, Long\]()**
* 从KafkaCluster获取offset：
  * **partitions**=kafkaCluster.getPartitions(kafkaTopicSet)
  * kafkaCluster.getConsumerOffsets(kafkaGroup, **partitions**)
* 把获得的offset放入Map并返回

```scala
//从ZK获取offset
def getOffsetFromZookeeper(kafkaCluster: KafkaCluster, kafkaGroup: String, kafkaTopicSet: Set[String]): Map[TopicAndPartition, Long] = {

	// 创建Map存储Topic和分区对应的offset
	val topicPartitionOffsetMap = new mutable.HashMap[TopicAndPartition, Long]()
	
	// 获取传入的Topic的所有分区
	// Either[Err, Set[TopicAndPartition]]  : Left(Err)   Right[Set[TopicAndPartition]]
	val topicAndPartitions: Either[Err, Set[TopicAndPartition]] = kafkaCluster.getPartitions(kafkaTopicSet)
	
	// 如果成功获取到Topic所有分区
	// topicAndPartitions: Set[TopicAndPartition]
	if (topicAndPartitions.isRight) {
		// 获取分区数据
		// partitions: Set[TopicAndPartition]
		val partitions: Set[TopicAndPartition] = topicAndPartitions.right.get
	
		// 获取指定分区的offset
		// offsetInfo: Either[Err, Map[TopicAndPartition, Long]]
		// Left[Err]  Right[Map[TopicAndPartition, Long]]
		val offsetInfo: Either[Err, Map[TopicAndPartition, Long]] = kafkaCluster.getConsumerOffsets(kafkaGroup, partitions)
	
		if (offsetInfo.isLeft) {
	
			// 如果没有offset信息则存储0
			// partitions: Set[TopicAndPartition]
			for (top <- partitions)
			topicPartitionOffsetMap += (top -> 0L)
		} else {
	
			// 如果有offset信息则存储offset
			// offsets: Map[TopicAndPartition, Long]
			val offsets: Map[TopicAndPartition, Long] = offsetInfo.right.get
			for ((top, offset) <- offsets)
			topicPartitionOffsetMap += (top -> offset)
		}
	}
	topicPartitionOffsetMap.toMap
}
```

##### 向ZK提交offset：

* 使用输出方法：Dstream.foreachRDD
* 获取DStream中的offset信息：
  * rdd.asInstanceOf[HasOffsetRanges].offsetRanges
  * KafkaRDD是HasOffsetRanges的私有实现类
* 遍历offset并更新Zookeeper中的元数据：
  * kafkaCluster.setConsumerOffsets
  * offset的标准格式： **Map[TopicAndPartition, Long]**
  * 偏移量Long：offsets.untilOffset
  * PS：示例是每次遍历提交一个，理论上可以遍历之后统一提交

```scala
//提交offset
  def offsetToZookeeper(kafkaDstream: InputDStream[String], kafkaCluster: KafkaCluster, kafka_group: String): Unit = {
    kafkaDstream.foreachRDD {
      rdd =>
        // 获取DStream中的offset信息
        // offsetsList: Array[OffsetRange]
        // OffsetRange: topic partition fromoffset untiloffset
        val offsetsList: Array[OffsetRange] = rdd.asInstanceOf[HasOffsetRanges].offsetRanges

        // 遍历每一个offset信息，并更新Zookeeper中的元数据
        // OffsetRange: topic partition fromoffset untiloffset
        for (offsets <- offsetsList) {
          val topicAndPartition = TopicAndPartition(offsets.topic, offsets.partition)
          // ack: Either[Err, Map[TopicAndPartition, Short]]
          // Left[Err]
          // Right[Map[TopicAndPartition, Short]]
          val ack: Either[Err, Map[TopicAndPartition, Short]] = kafkaCluster.setConsumerOffsets(kafka_group, Map((topicAndPartition, offsets.untilOffset)))
          if (ack.isLeft) {
            println(s"Error updating the offset to Kafka cluster: ${ack.left.get}")
          } else {
            println(s"update the offset to Kafka cluster: ${offsets.untilOffset} successfully")
          }
        }
    }
  }
}
```

### 自定义数据源

**步骤：**

1. 继承Receiver

2. 实现OnStart、onStop方法

3. 在StreamingContext中调用 ssc.receiverStream(new MyReceiver()) ，生成DStream

**具体内容：**

> onStart：创建新线程，实现run方法并启动
>
> > 数据入口： store(input)
> >
> > 把需要交给Dstream 的数据放入store()即可

> onStop：关闭资源

```scala
class MyReceiver(host: String, port: Int) extends Receiver[String](StorageLevel.MEMORY_ONLY) {
	//创建一个Socket
  private var socket: Socket = _

  //最初启动的时候，调用该方法，作用为：读数据并将数据发送给Spark
//核心：创建新线程并启动
  override def onStart(): Unit = {
    new Thread("Socket Receiver") {
      setDaemon(true)
      override def run() { receive() }
    }.start()
  }
//核心：关闭资源
  override def onStop(): Unit = {
    if(socket != null ){
      socket.close()
      socket = null
    }
  }
    
  //读数据并将数据发送给Spark
  def receive(): Unit = {
    try {
      socket = new Socket(host, port)
      //创建一个BufferedReader用于读取端口传来的数据
      val reader = new BufferedReader(
        new InputStreamReader(
          socket.getInputStream, StandardCharsets.UTF_8))
      //定义一个变量，用来接收端口传过来的数据
      var input: String = null

      //读取数据 循环发送数据给Spark 一般要想结束发送特定的数据 如："==END=="
      while ((input = reader.readLine())!=null) {
        store(input)
      }
    } catch {
      case e: ConnectException =>
        restart(s"Error connecting to $host:$port", e)
        return
    }
  }
    
}
```

### RDD队列

**步骤：**

1. 定义一个Queue[RDD]队列
2. 调用ssc.queueStreaam( 队列，是否每次消费一个RDD )
3. 向队列中放入RDD

```scala
object Spark02_DStreamCreate_RDDQueue {
  def main(args: Array[String]): Unit = {
    // 创建Spark配置信息对象
    val conf = new SparkConf().setMaster("local[*]").setAppName("RDDStream")
    // 创建SparkStreamingContext
    val ssc = new StreamingContext(conf, Seconds(3))
    // 创建RDD队列
    val rddQueue = new mutable.Queue[RDD[Int]]()
    // 创建QueueInputDStream
    val inputStream = ssc.queueStream(rddQueue,oneAtATime = false)
    // 处理队列中的RDD数据
    val mappedStream = inputStream.map((_,1))
    val reducedStream = mappedStream.reduceByKey(_ + _)
    // 打印结果
    reducedStream.print()
    // 启动任务
    ssc.start()
    // 循环创建并向RDD队列中放入RDD
    for (i <- 1 to 5) {
      rddQueue += ssc.sparkContext.makeRDD(1 to 5, 10)
      Thread.sleep(2000)
    }
    ssc.awaitTermination()
  }
}
```

## DStream转换

### 无状态转换

**Transform：**

DStream的一个方法，将所有操作转换为RDD的操作

```scala
val vDStream=DStream.transform(rdd => rdd）
```

- map(func)
- flatMap(func)
- filter(func)
- count()
- union(otherStream)
- reduce(func)
- reduceByKey(func, [numTasks])
- repartition(numPartitions)： 改变Dstream分区数

### 有状态转换

#### UpdateStateByKey

```scala
//设置检查点  用于保存状态
ssc.checkpoint("path")
```

* DStream.updateStateByKey()
* 将之前的结果和当前批次数据做聚合，只能用于K-V结构
* 当前数据按照K聚合，依次放入Seq中
* 历史结果放入Option[T]中
* 处理之后把结果包装为Option[T]返回

  * PS：历史结果和当前结果类型一致

```scala
val stateDS: DStream[(String, Int)] = mapDS.updateStateByKey(
      (seq: Seq[Int], state: Option[Int]) => {
        Option(seq.sum + state.getOrElse(0))
      }
    )
```

### Window Operations

**1)**     **window(windowLength, slideInterval)**

基于对源DStream窗化的批次进行计算返回一个新的Dstream

* 对一定时间段内的数据操作，类似于滑窗
* 窗口时长：计算内容的时间范围
* 滑动步长：隔多久触发一次计算
  * PS：这两者都必须为采集周期的整数倍

```scala
 //设置窗口大小，滑动的步长
 val windowDS: DStream[String] = flatMapDS.window(Seconds(6),Seconds(3))
```

**2)**     **countByWindow(windowLength, slideInterval)**

 返回滑动窗口中的**元素个数**

**3)**     **countByValueAndWindow()**

返回DStream，结构为**(Value，num)**

**4)**     **reduceByWindow(func, windowLength, slideInterval)**

返回DStream，里面只有一个元素

**5)**     **reduceByKeyAndWindow(func, windowLength, slideInterval, [numTasks])**

当在一个(K,V)对的DStream上调用此函数，会返回一个新(K,V)对的DStream，此处通过对滑动窗口中批次数据使用reduce函数来整合每个key的value值

**6)**     **reduceByKeyAndWindow(func, invFunc, windowLength, slideInterval, [numTasks])**

通过reduce进入到滑动窗口数据并”反向reduce”离开窗口的旧数据来实现这个操作。

## DStream输出

**1)**     **print()**

打印DStream中每一批次数据的最开始10个元素。这用于开发和调试。

**2)**     **saveAsTextFiles(prefix, [suffix])**

以text文件形式存储这个DStream的内容。

**prefix：前**缀，即文件路径+前缀

**suffix：**后缀，默认为空

**3)**     **saveAsObjectFiles(prefix, [suffix])**

以Java对象序列化的方式将Stream中的数据保存为 SequenceFiles .

**prefix：前**缀，即文件路径+前缀

**suffix：**后缀，默认为空

**4)**     **saveAsHadoopFiles(prefix, [suffix])**

将Stream中的数据保存为 Hadoop files. 

**prefix：前**缀，即文件路径+前缀

**suffix：**后缀，默认为空

**5)**     **foreachRDD(func：Unit)**

对DStream中每个RDD执行操作（类似于RDD里的foreach）

PS：在使用JDBC时，用rdd.foreachPartition

## DStream进阶

###  累加器和广播变量

和Spark core一样，不过要先获得sparkContext

```scala
// 定义累加器
val sum1=ssc.sparkContext.doubleAccumulator
// 使用累加器
	sum1.add(count)
// 获取累加器
	sum1.value


val list: List[(String, Int)] = List(("a", 4), ("b", 5), ("c", 6))
// 声明广播变量
val broadcastList: Broadcast[List[(String, Int)]] = ssc.sparkContext.broadcast(list)
// 使用广播变量
fbroadcastList.value
```

### Caching / Persist

和Spark core一样

```scala
resDStream.cache()
resDStream.persist()
resDStream.persist(StorageLevel.MEMORY_ONLY)
```

### SQL Operation

* 获得SparkSession
* 引入implicits
* 把DStream转换为RDD，通常用foreachRDD
* RDD转换为DataFrame，rdd.toDF("...")
* df.createOrReplaceTempView("words")
* spark.sql("sql")

```scala
val spark = SparkSession.builder.config(conf).getOrCreate()
import spark.implicits._
mapDS.foreachRDD(rdd =>{
  val df: DataFrame = rdd.toDF("word", "count")
  df.createOrReplaceTempView("words")
  spark.sql("select * from words").show
})
```

## 优雅的关闭

* SparkConf设置优雅的关闭为true
* 在ssc.start() 之后启动新的线程
  * 线程做死循环，需要sleep，否则一直占用资源
  * 线程中监控一个标志位，通常为Redis里的值或者HDFS的文件和文件夹
  * 标志位条件为true，调用context.stop(true, true)
  * 最后System.exit(0)

```scala
// SparkConf设置优雅的关闭为true
sparkConf.set("spark.streaming.stopGracefullyOnShutdown", "true")

// 启动新的线程，希望在特殊的场合关闭SparkStreaming
new Thread(new Runnable {
    override def run(): Unit = {

        while ( true ) {
            try {
                Thread.sleep(5000)
            } catch {
                case ex : Exception => println(ex)
            }

            // 监控HDFS文件的变化
            val fs: FileSystem = FileSystem.get(new URI("hdfs://linux1:9000"), new Configuration(), "root")

            val state: StreamingContextState = context.getState()
            // 如果环境对象处于活动状态，可以进行关闭操作
            if ( state == StreamingContextState.ACTIVE ) {

                // 判断路径是否存在
                val flg: Boolean = fs.exists(new Path("hdfs://linux1:9000/stopSpark2"))
                if ( flg ) {
                    context.stop(true, true)
                    System.exit(0)
                }

            }
        }

    }
}).start()
```



# 附录

## 向HBase读写数据

由于org.apache.hadoop.hbase.mapreduce.TableInputFormat类的实现，Spark可以通过Hadoop输入格式访问HBase。这个输入格式会返回键值对数据，其中键的类型为org.apache.hadoop.hbase.io.ImmutableBytesWritable，而值的类型为org.apache.hadoop.hbase.client.Result。

### 添加依赖

```xml
<dependency>
   <groupId>org.apache.hbase</groupId>
    <artifactId>hbase-server</artifactId>
    <version>1.3.1</version>
</dependency>
 
<dependency>
    <groupId>org.apache.hbase</groupId>
    <artifactId>hbase-client</artifactId>
    <version>1.3.1</version>
</dependency>
```
### 从HBase读取数据

```scala
package com.atguigu

import org.apache.hadoop.conf.Configuration
import org.apache.hadoop.hbase.HBaseConfiguration
import org.apache.hadoop.hbase.client.Result
import org.apache.hadoop.hbase.io.ImmutableBytesWritable
import org.apache.hadoop.hbase.mapreduce.TableInputFormat
import org.apache.spark.rdd.RDD
import org.apache.spark.{SparkConf, SparkContext}
import org.apache.hadoop.hbase.util.Bytes

object HBaseSpark {

  def main(args: Array[String]): Unit = {

    //创建spark配置信息
    val sparkConf: SparkConf = new SparkConf().setMaster("local[*]").setAppName("JdbcRDD")

    //创建SparkContext
    val sc = new SparkContext(sparkConf)

    //构建HBase配置信息
    val conf: Configuration = HBaseConfiguration.create()
    conf.set("hbase.zookeeper.quorum", "hadoop102,hadoop103,hadoop104")
    conf.set(TableInputFormat.INPUT_TABLE, "rddtable")

    //从HBase读取数据形成RDD
    val hbaseRDD: RDD[(ImmutableBytesWritable, Result)] = sc.newAPIHadoopRDD(
      conf,
      classOf[TableInputFormat],
      classOf[ImmutableBytesWritable],
      classOf[Result])

    val count: Long = hbaseRDD.count()
    println(count)

    //对hbaseRDD进行处理
    hbaseRDD.foreach {
      case (_, result) =>
        val key: String = Bytes.toString(result.getRow)
        val name: String = Bytes.toString(result.getValue(Bytes.toBytes("info"), Bytes.toBytes("name")))
        val color: String = Bytes.toString(result.getValue(Bytes.toBytes("info"), Bytes.toBytes("color")))
        println("RowKey:" + key + ",Name:" + name + ",Color:" + color)
    }

    //关闭连接
    sc.stop()
  }
}
```

### 往HBase写入

```scala
def main(args: Array[String]) {
//获取Spark配置信息并创建与spark的连接
  val sparkConf = new SparkConf().setMaster("local[*]").setAppName("HBaseApp")
val sc = new SparkContext(sparkConf)

//创建HbaseConf
val conf = HBaseConfiguration.create()
  val jobConf = new JobConf(conf)
  jobConf.setOutputFormat(classOf[TableOutputFormat])
  jobConf.set(TableOutputFormat.OUTPUT_TABLE, "fruit_spark")

//定义往Hbase插入数据的方法
  def convert(triple: (Int, String, Int)) = {
    val put = new Put(Bytes.toBytes(triple._1))
    put.addImmutable(Bytes.toBytes("info"), Bytes.toBytes("name"), Bytes.toBytes(triple._2))
    put.addImmutable(Bytes.toBytes("info"), Bytes.toBytes("price"), Bytes.toBytes(triple._3))
    (new ImmutableBytesWritable, put)
  }

//创建一个RDD
val initialRDD = sc.makeRDD(List((1,"apple",11), (2,"banana",12), (3,"pear",13)))

//将RDD内容写到Hbase
val localData = initialRDD.map(convert)

localData.saveAsHadoopDataset(jobConf)
}
```

