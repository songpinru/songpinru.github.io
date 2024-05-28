# 结构

* hdfs
* yarn
* mapreducer
* common

# 安装

## 虚拟机准备

内存2G，硬盘50G

1） 安装好linux
/boot 200M 
/swap 2g 
/ 剩余

2） *安装VMTools

3） 关闭防火墙

```bash
sudo service iptables stop
sudo chkconfig iptables off
```

4） 设置静态IP，改主机名

```bash
#改ip
vim /etc/sysconfig/network-scripts/ifcfg-eth0
#改主机名，改HOSTNAME=那一行
vim /etc/sysconfig/network
```

```bash
DEVICE=eth0
TYPE=Ethernet
ONBOOT=yes
BOOTPROTO=static
NAME="eth0"
IPADDR=192.168.5.101
PREFIX=24
GATEWAY=192.168.5.2
DNS1=192.168.5.2

```

5） 配置/etc/hosts

```bash
vim /etc/hosts

192.168.5.100   hadoop100
192.168.5.101   hadoop101
192.168.5.102   hadoop102
192.168.5.103   hadoop103
192.168.5.104   hadoop104
192.168.5.105   hadoop105
192.168.5.106   hadoop106
192.168.5.107   hadoop107
192.168.5.108   hadoop108
192.168.5.109   hadoop109
```

6） 创建一个一般用户user，给他配置密码
useradd user
passwd user

7）配置这个用户为sudoers

```bash
vi /etc/sudoers
在root	ALL=(ALL）       ALL
添加user	ALL=(ALL）       NOPASSWD:ALL
```

8） *在/opt目录下创建两个文件夹module和software，并把所有权赋给atguigu

```bash
mkdir /opt/module /opt/software
chown atguigu:atguigu /opt/module /opt/software
```

9） 关机，快照，克隆

从这里开始要以一般用户登陆

10） 克隆的虚拟机改IP

11） 搞一个分发脚本

```bash
cd ~
vim xsync
```

```bash
#!/bin/bash
#1. 判断参数个数
if [ $# -lt 1 ]
then
    echo Not Enough Arguement!
    exit;
fi
#2. 遍历集群所有机器
for host in hadoop102 hadoop103 hadoop104
do
    echo ====================    $host    ====================
    #3. 遍历所有目录，挨个发送
    for file in $@
    do
        #4 判断文件是否存在
        if [ -e $file ]
        then
            #5. 获取父目录
            pdir=$(cd -P $(dirname $file); pwd)
            
            #6. 获取当前文件的名称
            fname=$(basename $file)

            
            ssh $host "mkdir -p $pdir"
            rsync -av $pdir/$fname $host:$pdir
            
        else
            echo $file does not exists!
        fi
    done
done
    
```

```bash
# 加权限
chmod +x xsync
# 放入/bin
sudo cp xsync /bin
# 同步至其他虚拟机
sudo xsync /bin/xsync
```

12） 配置免密登陆

```bash
# 1. 生成密钥对
ssh-keygen -t rsa 三次回车

# 2. 发送公钥
ssh-copy-id hadoop102 输入一次密码
ssh-copy-id hadoop103
ssh-copy-id hadoop104

# 3. 分别ssh登陆一下所有虚拟机
ssh hadoop103
exit
ssh hadoop104
exit

# 4. 把/home/user/.ssh 文件夹发送到集群所有服务器
xsync /home/user/.ssh
```
## hadoop安装

1. 在一台机器上安装Java和Hadoop，并配置环境变量，并分发到集群其他机器
          1. 拷贝文件到/opt/software，两个tar包
       2. 解压
       3. 配置环境变量

*在/etc/profile.d文件夹下新建一个environment.sh文件，内容如下

```bash
#JAVA_HOME
export JAVA_HOME=/opt/module/jdk1.8.0_144
export PATH=$PATH:$JAVA_HOME/bin

#HADOOP_HOME
export HADOOP_HOME=/opt/module/hadoop-2.7.2
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
```

并将这个文件同步到集群所有机器

```bash
sudo xsync /etc/profile.d
```

###所有配置文件都在$HADOOP_HOME/etc/hadoop

### 配置Core-site.xml

```xml
 <!-- 指定HDFS中NameNode的地址 -->
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://hadoop102:9000</value>
    </property>

    <!-- 指定Hadoop运行时产生文件的存储目录 -->
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/opt/module/hadoop-2.7.2/data/tmp</value>
    </property>
```



### 配置hdfs-site.xml
```xml
<!-- 数据的副本数量 -->
<property>
    <name>dfs.replication</name>
    <value>3</value>
</property>
<!-- 指定Hadoop辅助名称节点主机配置 -->
<property>
      <name>dfs.namenode.secondary.http-address</name>
      <value>hadoop104:50090</value>
</property>
```
### 配置yarn-site.xml
```xml
<!-- Site specific YARN configuration properties -->
<!-- Reducer获取数据的方式 -->
<property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
</property>

<!-- 指定YARN的ResourceManager的地址 -->
<property>
    <name>yarn.resourcemanager.hostname</name>
    <value>hadoop103</value>
</property>

<!-- 日志聚集功能使能 -->
<property>
    <name>yarn.log-aggregation-enable</name>
    <value>true</value>
</property>

<!-- 日志保留时间设置7天 -->
<property>
    <name>yarn.log-aggregation.retain-seconds</name>
    <value>604800</value>
</property>
```
### 配置mapred-site.xml
```xml
<property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
</property>
<!-- 历史服务器端地址 -->
<property>
    <name>mapreduce.jobhistory.address</name>
    <value>hadoop104:10020</value>
</property>
<!-- 历史服务器web端地址 -->
<property>
    <name>mapreduce.jobhistory.webapp.address</name>
    <value>hadoop104:19888</value>
</property>
```
### 配置Slaves
```
hadoop102
hadoop103
hadoop104
```
### 分发、格式化

```bash
# 分发配置文件
xsync /opt/module/hadoop-2.7.2/etc

# 格式化Namenode 在hadoop102
hdfs namenode -format
```

# 命令

```bash
# 群起群开
start-dfs.sh
stop-dfs.sh
start-yarn.sh
stop-yarn.sh
# 单点启动
hadoop-daemon.sh start namenode
hadoop-daemon.sh start datanode
yarn-daemon.sh start resourcemanager
yarn-daemon.sh start nodemanager
# 启动历史服务器：
mr-jobhistory-daemon.sh start historyserver
```
```bash
#如果集群出了问题
stop-dfs.sh
stop-yarn.sh
#三台机器都要执行
cd $HADOOP_HOME
rm -rf data logs
#然后重新格式化
```

# 其他配置

## 时间同步（必须root用户）

（1）检查ntp是否安装

```bash
rpm -qa|grep ntp
ntp-4.2.6p5-10.el6.centos.x86_64
fontpackages-filesystem-1.41-1.1.el6.noarch
ntpdate-4.2.6p5-10.el6.centos.x86_64
```

（2）修改ntp配置文件

```bash
vim /etc/ntp.conf

# 修改内容如下
# 修改1（授权192.168.1.0-192.168.1.255网段上的所有机器可以从这台机器上查询和同步时间）
#取消下列注释
# restrict 192.168.1.0 mask 255.255.255.0 nomodify notrap为
restrict 192.168.1.0 mask 255.255.255.0 nomodify notrap

# 修改2（集群在局域网中，不使用其他互联网上的时间）
# 注释以下
# server 0.centos.pool.ntp.org iburst
# server 1.centos.pool.ntp.org iburst
# server 2.centos.pool.ntp.org iburst
# server 3.centos.pool.ntp.org iburst

# 添加3（当该节点丢失网络连接，依然可以采用本地时间作为时间服务器为集群中的其他节点提供时间同步）
server 127.127.1.0
fudge 127.127.1.0 stratum 10
```

```bash
vim /etc/sysconfig/ntpd

# 增加内容如下（让硬件时间与系统时间一起同步）
SYNC_HWCLOCK=yes
```

（3）重新启动ntpd服务

```bash
# 查看ntp状态
service ntpd status
# 启动ntp
service ntpd start
# 开机自启
chkconfig ntpd on
```

（4） 其他机器配置（必须root用户）

在其他机器配置10分钟与时间服务器同步一次

```bash
crontab -e
编写定时任务如下：
*/10 **** /usr/sbin/ntpdate hadoop102
```

## 黑白名单

黑名单退役：

dfs.hosts.exclude

白名单防火墙：

dfs.hosts

## datanode多目录

hdfs-site.xml 添加：

```xml
<property>
<name>dfs.datanode.data.dir</name>
<value>file:///${hadoop.tmp.dir}/dfs/data1,file:///${hadoop.tmp.dir}/dfs/data2</value>
</property>
```

hadoop.tmp.dir在core中配置

## NameNode镜像文件目录

hdfs-site.xml 添加：

```xml
<property>
  <name>dfs.namenode.name.dir</name>
  <value>/opt/module/hadoop-2.7.2/data/tmp/dfs/name</value>
</property>
```

## CheckPoint时间

hdfs-site.xml 添加：

```xml
<property>
  <name>dfs.namenode.checkpoint.period</name>
  <value>3600</value>
    <description>每隔一小时执行一次</description>
</property>

<property>
  <name>dfs.namenode.checkpoint.txns</name>
  <value>1000000</value>
<description>操作动作次数</description>
</property>

<property>
  <name>dfs.namenode.checkpoint.check.period</name>
  <value>60</value>
<description> 1分钟检查一次操作次数</description>
</property >
```

## DataNode掉线时间

hdfs-site.xml 添加：

```xml
<property>
    <name>dfs.namenode.heartbeat.recheck-interval</name>
    <value>300000</value>
</property>
<property>
    <name>dfs.heartbeat.interval</name>
    <value>3</value>
</property>
```

TimeOut = 2 * dfs.namenode.heartbeat.recheck-interval + 10 * dfs.heartbeat.interval。

而默认的dfs.namenode.heartbeat.recheck-interval 大小为5分钟，dfs.heartbeat.interval默认为3秒。

## NameNode高可用

安装配置zookeeper

core-site.xml:

```xml
<configuration>
<!-- 把两个NameNode）的地址组装成一个集群mycluster -->
		<property>
			<name>fs.defaultFS</name>
        	<value>hdfs://mycluster</value>
		</property>

		<!-- 指定hadoop运行时产生文件的存储目录 -->
		<property>
			<name>hadoop.tmp.dir</name>
			<value>/opt/module/hadoop-2.7.2/data/tmp</value>
		</property>
</configuration>

```

hdfs-site.xml:

```xml
<configuration>
<!-- 完全分布式集群名称 -->
<property>
    <name>dfs.nameservices</name>
    <value>mycluster</value>
</property>
<!-- 集群中NameNode节点都有哪些 -->
<property>
    <name>dfs.ha.namenodes.mycluster</name>
    <value>nn1,nn2</value>
</property>
<!-- nn1的RPC通信地址 -->
<property>
    <name>dfs.namenode.rpc-address.mycluster.nn1</name>
    <value>hadoop102:9000</value>
</property>
<!-- nn2的RPC通信地址 -->
<property>
    <name>dfs.namenode.rpc-address.mycluster.nn2</name>
    <value>hadoop103:9000</value>
</property>
<!-- nn1的http通信地址 -->
<property>
    <name>dfs.namenode.http-address.mycluster.nn1</name>
    <value>hadoop102:50070</value>
</property>
<!-- nn2的http通信地址 -->
<property>
    <name>dfs.namenode.http-address.mycluster.nn2</name>
    <value>hadoop103:50070</value>
</property>
<!-- 指定NameNode元数据在JournalNode上的存放位置 -->
<property>
    <name>dfs.namenode.shared.edits.dir</name>
    <value>qjournal://hadoop102:8485;hadoop103:8485;hadoop104:8485/mycluster</value>
</property>
<!-- 配置隔离机制，即同一时刻只能有一台服务器对外响应 -->
<property>
    <name>dfs.ha.fencing.methods</name>
    <value>sshfence</value>
</property>
<!-- 使用隔离机制时需要ssh无秘钥登录-->
<property>
    <name>dfs.ha.fencing.ssh.private-key-files</name>
    <value>/home/user/.ssh/id_rsa</value>
</property>
<!-- 声明journalnode服务器存储目录-->
<property>
    <name>dfs.journalnode.edits.dir</name>
    <value>/opt/module/hadoop-2.7.2/data/jn</value>
</property>
<!-- 关闭权限检查-->
<property>
    <name>dfs.permissions.enable</name>
    <value>false</value>
</property>
<!-- 访问代理类：client，mycluster，active配置失败自动切换实现方式-->
<property>
    <name>dfs.client.failover.proxy.provider.mycluster</name>			 	<value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
</property>
</configuration>
```

命令：

```bash
# 各个journalnode启动
sbin/hadoop-daemon.sh start journalnode
# nn1格式化，启动
bin/hdfs namenode -format
sbin/hadoop-daemon.sh start namenode
# nn2同步，启动
bin/hdfs namenode -bootstrapStandby
sbin/hadoop-daemon.sh start namenode
```

## ResourceManager高可用

```xml
<configuration>

<property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
</property>

<!--启用resourcemanager ha-->
<property>
    <name>yarn.resourcemanager.ha.enabled</name>
    <value>true</value>
</property>

<!--声明两台resourcemanager的地址-->
<property>
    <name>yarn.resourcemanager.cluster-id</name>
    <value>cluster-yarn1</value>
</property>

<property>
    <name>yarn.resourcemanager.ha.rm-ids</name>
    <value>rm1,rm2</value>
</property>

<property>
    <name>yarn.resourcemanager.hostname.rm1</name>
    <value>hadoop102</value>
</property>

<property>
    <name>yarn.resourcemanager.hostname.rm2</name>
    <value>hadoop103</value>
</property>

<!--指定zookeeper集群的地址--> 
<property>
    <name>yarn.resourcemanager.zk-address</name>
    <value>hadoop102:2181,hadoop103:2181,hadoop104:2181</value>
</property>

<!--启用自动恢复--> 
<property>
    <name>yarn.resourcemanager.recovery.enabled</name>
    <value>true</value>
</property>

<!--指定resourcemanager的状态信息存储在zookeeper集群--> 
<property>
    <name>yarn.resourcemanager.store.class</name>     				 <value>org.apache.hadoop.yarn.server.resourcemanager.recovery.ZKRMStateStore</value>
</property>
    
    <!-- 日志聚集功能使能 -->
<property>
    <name>yarn.log-aggregation-enable</name>
    <value>true</value>
</property>

<!-- 日志保留时间设置7天 -->
<property>
    <name>yarn.log-aggregation.retain-seconds</name>
    <value>604800</value>
</property>

</configuration>
```



```bash
# 各个journalnode启动
sbin/hadoop-daemon.sh start journalnode
# nn1格式化，启动
bin/hdfs namenode -format
sbin/hadoop-daemon.sh start namenode
# nn2同步，启动
bin/hdfs namenode -bootstrapStandby
sbin/hadoop-daemon.sh start namenode

# rm2单点启动
sbin/yarn-daemon.sh start resourcemanager
```

## HDFS-HA 自动故障转移

```xml
<!--（1）在hdfs-site.xml中增加-->
<property>
	<name>dfs.ha.automatic-failover.enabled</name>
	<value>true</value>
</property>

<!--（2）在core-site.xml文件中增加-->
<property>
	<name>ha.zookeeper.quorum</name>
	<value>hadoop102:2181,hadoop103:2181,hadoop104:2181</value>
</property>
```

## capacity多队列设置

capacity-scheduler.xml

```xml
<property>
    <name>yarn.scheduler.capacity.root.queues</name>
    <value>default,hive</value>
    <description>The queues at the this level (root is the root queue). </description>
  </property>
<property>
    <name>yarn.scheduler.capacity.root.default.capacity</name>
    <value>40</value>
</property>

<!--队列目标资源百分比，所有队列相加必须等于100,hive为子队列-->
<property>
    <name>yarn.scheduler.capacity.root.hive.capacity</name>
    <value>60</value>
  </property>
<!--队列最大资源百分比-->
  <property>
    <name>yarn.scheduler.capacity.root.hive.maximum-capacity</name>
    <value>100</value>
  </property>
<!--单用户可用队列资源占比100%-->
  <property>
    <name>yarn.scheduler.capacity.root.hive.user-limit-factor</name>
    <value>1</value>
  </property>
<!--队列状态（RUNNING或STOPPING）-->
  <property>
    <name>yarn.scheduler.capacity.root.hive.state</name>
    <value>RUNNING</value>
  </property>
<!-- 队列允许哪些用户提交-->
  <property>
    <name>yarn.scheduler.capacity.root.hive.acl_submit_applications</name>
    <value>*</value>
  </property>
<!-- 队列允许哪些用户管理-->
<property>
    <name>yarn.scheduler.capacity.root.hive.acl_administer_queue</name>
    <value>*</value>
  </property>

```

[ 队列配置详解 ](https://www.cnblogs.com/yanghaolie/p/6274098.html)

<https://www.cnblogs.com/yanghaolie/p/6274098.html>

## 压缩设置

```bash
# 查看支持的压缩格式
hadoop checknative
# 如果出现openssl为false，那么就在线安装一下依赖包
yum install openssl-devel
```

| key（默认在mapred-site.xml中配置）               | value                                                        |      |
| ------------------------------------------------ | ------------------------------------------------------------ | ---- |
| io.compression.codecs（在core-site.xml中配置）   | org.apache.hadoop.io.compress.DefaultCodec, org.apache.hadoop.io.compress.GzipCodec,   org.apache.hadoop.io.compress.BZip2Codec |      |
| mapreduce.map.output.compress                    | false                                                        |      |
| mapreduce.map.output.compress.codec              | org.apache.hadoop.io.compress.DefaultCodec                   |      |
| mapreduce.output.fileoutputformat.compress       | false                                                        |      |
| mapreduce.output.fileoutputformat.compress.codec | org.apache.hadoop.io.compress.   DefaultCodec                |      |
| mapreduce.output.fileoutputformat.compress.type  | RECORD                                                       |      |

| 压缩格式 | 工具  | 算法    | 文件扩展名 | 是否可切分 |
| -------- | ----- | ------- | ---------- | ---------- |
| DEFLATE  | 无    | DEFLATE | .deflate   | 否         |
| Gzip     | gzip  | DEFLATE | .gz        | 否         |
| bzip2    | bzip2 | bzip2   | .bz2       | 是         |
| LZO      | lzop  | LZO     | .lzo       | 是         |
| Snappy   | 无    | Snappy  | .snappy    | 否         |

| 压缩格式 | 对应的编码/解码器                          |
| -------- | ------------------------------------------ |
| DEFLATE  | org.apache.hadoop.io.compress.DefaultCodec |
| gzip     | org.apache.hadoop.io.compress.GzipCodec    |
| bzip2    | org.apache.hadoop.io.compress.BZip2Codec   |
| LZO      | com.hadoop.compression.lzo.LzopCodec       |
| Snappy   | org.apache.hadoop.io.compress.SnappyCodec  |

| 压缩格式     | 压缩比 | 压缩速率 | 解压速率 |
| ------------ | ------ | -------- | -------- |
| gzip/deflate | 13.4%  | 21 MB/s  | 118 MB/s |
| bzip2        | 13.2%  | 2.4MB/s  | 9.5MB/s  |
| lzo          | 20.5%  | 135 MB/s | 410 MB/s |
| snappy       | 22.2%  | 172 MB/s | 409 MB/s |

# HDFS

## 命令：

```bash
bin/hadoop fs

本地--》HDFS
    -put
    -copyFromLocal
    -moveFromLocal
    -appendToFile
    
HDFS--》HDFS
    -cp
    -mv
    -chown
    -chgrp
    -chmod
    -mkdir
    -du
    -df
    -cat
    -rm
    
HFDS--》本地
    -get
    -getmerge
    -copyToLocal
    
# 归档    
bin/hadoop archive -archiveName input.har –p  /user/atguigu/input   /user/atguigu/output
# 查看归档
hadoop fs -lsr /user/atguigu/output/input.har
# 解压归档
hadoop fs -cp har:/// user/atguigu/output/input.har/*    /user/atguigu
```

## POM：

```xml
<dependencies>
		<dependency>
			<groupId>junit</groupId>
			<artifactId>junit</artifactId>
			<version>RELEASE</version>
		</dependency>
		<dependency>
			<groupId>org.apache.logging.log4j</groupId>
			<artifactId>log4j-core</artifactId>
			<version>2.8.2</version>
		</dependency>
		<dependency>
			<groupId>org.apache.hadoop</groupId>
			<artifactId>hadoop-common</artifactId>
			<version>2.7.2</version>
		</dependency>
		<dependency>
			<groupId>org.apache.hadoop</groupId>
			<artifactId>hadoop-client</artifactId>
			<version>2.7.2</version>
		</dependency>
		<dependency>
			<groupId>org.apache.hadoop</groupId>
			<artifactId>hadoop-hdfs</artifactId>
			<version>2.7.2</version>
		</dependency>
		<dependency>
			<groupId>jdk.tools</groupId>
			<artifactId>jdk.tools</artifactId>
			<version>1.8</version>
			<scope>system</scope>
			<systemPath>${JAVA_HOME}/lib/tools.jar</systemPath>
		</dependency>
</dependencies>
```

## client API

```java
// 1 获取文件系统
Configuration configuration = new Configuration();
FileSystem fs = FileSystem.get(new URI("hdfs://hadoop102:9000"),configuration,"atguigu");

// 2 执行操作
fs.copyFromLocalFile(new Path("e:/banzhang.txt"), new Path("/banzhang.txt"));
fs.copyToLocalFile(false, new Path("/banzhang.txt"), new Path("e:/banhua.txt"), true);
...
// 3 关闭资源
fs.close();
```

## HDFS 读写流程

### 读数据

- client向namenode请求下载数据
- namenode返回文件元数据
- client向datanode发送读取数据块请求
- datanode传输数据块
  - 如果没有响应或失败就找其他节点

```sequence
client->namenode: 请求下载文件
namenode-->client: 文件元数据
client->datanode1: 读取数据块
datanode1-->>client: 传输数据块
client->datanode2: 读取数据块
datanode2-->>client: 传输数据块
client->datanode3: 读取数据块
datanode3-->>client: 传输数据块or失败
```

### 写数据

- clien->namenode:请求写数据
- namenode-> client :返回响应
- client->namenode: 请求上传第一个块
- namenode-> client :返回可用datanode
- client->datanode : 请求连接
- datanode-> client: 返回响应
- client->datanode : 上传数据

```sequence
client->namenode:请求写数据
namenode-> client :返回响应
client->namenode: 请求上传第一个块
namenode-> client :返回可用datan
client->datanode : 请求连接
datanode--> client: 返回响应
client->datanode : 上传数据
```

NN和2NN：

```sequence
Title:nn加载镜像和日志
2nn->nn :询问是否checkpoint
nn--> 2nn :yes/no
2nn->nn:请求checkpoint
nn->2nn:滚动日志（改名），把日志和镜像传给2nn
2nn->nn:2nn合并后传回给nn，nn替换并改名
```



# Yarn

工作机制

```sequence
client->rm:申请application
rm->client:返回响应和资源提交地址
Note left of client:提交xml，jar，split
client->rm:申请执行app
Note right of rm :app放入任务调度器
rm->nm1:分配app给nm1
Note right of nm1 :生成container，\n下载资源，\n启动appMaster
nm1->rm:申请资源执行maptask
Note left of rm :放入任务调度器
rm->nm2:分配任务给nm2，生成container
nm1->nm2:发送任务启动脚本
Note right of nm2:执行maptask
nm2-->nm1:maptask结束
nm1->rm:申请资源执行reducetask
Note left of rm :放入任务调度器
rm->nm3:分配任务给nm3，生成container
nm1->nm3:发送任务启动脚本
Note right of nm3:执行reducetask
nm3-->nm1:maptask结束
nm1->rm:任务结束，注销app
```
## InputFormat
| inputformat      | 切片规则       | RecordReader             |
| ---------------- | -------------- | ------------------------ |
| InputFormat      | 1. 切片        | 2.把切片打散成KV         |
| FileInputFormat  | 按文件->块大小 | 没有实现                 |
| CombineTextIF    | 重写了切片规则 | CombineFileRecordReader  |
| TextInputFormat  | FIF的          | LineRecordReader         |
| KeyValueIF       | FIF的          | KeyValueLineRecordReader |
| NLineInputFormat | 重写：按行切   | LineRecordReader         |
| 自定义           | FIF的          | 自定义RR                 |
## yarn调度器

FIFO

Capacity（容量调度器）：多队列，每个队列可以并行执行（如果有资源的话）

fair（公平调度器）：多队列，所有任务并行执行，但是队列变成池（pool），抢占式，新任务等一会没资源就会kill池内其他task（kill的task有优先级，老task会优先被kill）

# 源码编译

## jar包准备

**hadoop源码、JDK8、maven、ant 、protobuf**

（1）hadoop-2.7.2-src.tar.gz

（2）jdk-8u144-linux-x64.tar.gz

（3）apache-ant-1.9.9-bin.tar.gz（build工具，打包用的）

（4）apache-maven-3.0.5-bin.tar.gz

（5）protobuf-2.5.0.tar.gz（序列化的框架）

## jar包安装

* hadoop-src
  * 解压
* JDK
  * 解压，配置环境变量
  * 验证命令：java -version
* maven
  * 解压，配置环境变量
  * 改仓库为阿里云（setting.xml）
* ant
  * 解压，配置环境变量
* protobuf
  * 解压，c编译，配置环境变量

```bash
# 安装  glibc-headers 和  g++
yum install glibc-headers
yum install gcc-c++

# 安装 make 和 cmake
yum install make
yum install cmake

# 编译protobuf
cd /opt/module/protobuf-2.5.0/
./configure
make 
make check 
make install 
ldconfig  # 重加载动态链接库

# 安装 openssl 和 ncurses 库
yum install openssl-devel
yum install ncurses-devel
```

## 编译源码

```bash
cd /opt/module/hadoop-2.7.2-src
# maven 编译
mvn package -Pdist,native -DskipTests -Dtar
# 编译成功后的hadoop在： /opt/module/hadoop-2.7.2-src/hadoop-dist/target

# 编译snappy版本（需编译安装snappy）
# mvn clean package -DskipTests -Pdist,native -Dtar -Dsnappy.lib=/usr/local/lib -Dbundle.snappy
```



## Hadoop支持LZO

```
0. 环境准备
maven（下载安装，配置环境变量，修改sitting.xml加阿里云镜像）
gcc-c++
zlib-devel
autoconf
automake
libtool
通过yum安装即可，yum -y install gcc-c++ lzo-devel zlib-devel autoconf automake libtool

1. 下载、安装并编译LZO

wget http://www.oberhumer.com/opensource/lzo/download/lzo-2.10.tar.gz

tar -zxvf lzo-2.10.tar.gz

cd lzo-2.10

./configure -prefix=/usr/local/hadoop/lzo/

make

make install

2. 编译hadoop-lzo源码

2.1 下载hadoop-lzo的源码，下载地址：https://github.com/twitter/hadoop-lzo/archive/master.zip
2.2 解压之后，修改pom.xml
    <hadoop.current.version>2.7.2</hadoop.current.version>
2.3 声明两个临时环境变量
     export C_INCLUDE_PATH=/usr/local/hadoop/lzo/include
     export LIBRARY_PATH=/usr/local/hadoop/lzo/lib 
2.4 编译
    进入hadoop-lzo-master，执行maven编译命令
    mvn package -Dmaven.test.skip=true
2.5 进入target，hadoop-lzo-0.4.21-SNAPSHOT.jar 即编译成功的hadoop-lzo组件
```

