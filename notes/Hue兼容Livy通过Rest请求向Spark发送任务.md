## Hue兼容Livy通过Rest请求向Spark发送任务

***参考hue官方文档***

```nginx
https://gethue.com/how-to-use-the-livy-spark-rest-job-server-for-interactive-spark-2-2/
```

### 环境准备

必须安装JDK

必须安装HADOOP

必须安装Spark

### 安装过程

#### 下载

```shell
wget http://archive.cloudera.com/beta/livy/livy-server-0.3.0.zip
```

#### 解压

```shell
unzip ./livy-server-0.3.0.zip
```

#### 修改conf/livy.conf

```shell
#默认local模式
#增加如下配置
livy.server.session.factory = yarn
```

#### 修改conf/livy-env.sh

```shell
#增加如下配置
export SPARK_HOME=/opt/spark
export HADOOP_CONF_DIR=/etc/hadoop/conf
export SPARK_CONF_DIR=/opt/spark/conf
```

#### 启动livy-server服务

````shell
#写上start是后台运行
bin/livy-server start
````

#### 查看服务进程

```shell
jps
20229 LivyServer
```

### Hue兼容

#### 修改hue.ini

```shell
#添加如下内容

 [spark]
 
 # livy 服务器域名
 livy_server_host=ddc001.lqad

 # livy 服务器端口
 livy_server_port=8998

 # Configure Livy to start in local 'process' mode, or 'yarn' workers.
 livy_server_session_kind=yarn
```

### 使用样例

#### 引入第三方依赖

```scala
import util.Random
val r = new Random
println(r.nextInt(10))
```

#### 运行结果

![image-20210317172903479](Hue%E5%85%BC%E5%AE%B9Livy%E9%80%9A%E8%BF%87Rest%E8%AF%B7%E6%B1%82%E5%90%91Spark%E5%8F%91%E9%80%81%E4%BB%BB%E5%8A%A1.assets/image-20210317172903479.png)

#### 创建spark任务

```scala
var counter = 0
val data = Array(1, 2, 3, 4, 5)
var rdd = sc.parallelize(data)
rdd.map(x=>x+1).collect()
```

#### 运行结果

![image-20210317172916534](Hue%E5%85%BC%E5%AE%B9Livy%E9%80%9A%E8%BF%87Rest%E8%AF%B7%E6%B1%82%E5%90%91Spark%E5%8F%91%E9%80%81%E4%BB%BB%E5%8A%A1.assets/image-20210317172916534.png)