# 安装部署

1） Flume官网地址

<http://flume.apache.org/>

2）文档查看地址

<http://flume.apache.org/FlumeUserGuide.html>

3）下载地址

http://archive.apache.org/dist/flume/

**安装：解压即可**

命令：

```bash
# conf/为conf目录，name为文件内的应用名，file为应用的配置
bin/flume-ng agent -c conf/ -f job/mysource.conf -n a1 -Dflume.root.logger=INFO,console
bin/flume-ng agent --conf conf/ --name a3 --conf-file job/flume-taildir-hdfs.conf
```

**修改/opt/module/flume/conf目录下的flume-env.sh配置：**

```bash
JAVA_OPTS="
-Xms100m
-Xmx200m"
```

默认

# 概念

* agent：进程
  * source
  * channel
  * sink
* event：数据单元head+body
  * head：k=v
  * body：byte[]

## 事务

* put事务，source端
* take事务，sink端

## ChannelSelector

在source配置，配合拦截器使用

* replicating(复制)
* multiplexing(多路复用)：根据条件发送至channel

```properties
#header为event中head的k，mapping后为对应的值
tier1.sources.source1.channels=channel1 channel2
tier1.sources.source1.selector.type=multiplexing
tier1.sources.source1.selector.header=flume.source
tier1.sources.source1.selector.mapping.app1=channel1
tier1.sources.source1.selector.mapping.app2=channel2
```

## SinkProcessor

在sink配置

* DefaultSinkProcessor：默认，对应单个sink
* LoadBalancingSinkProcessor：负载均衡，对应sinkgroups
* FailoverSinkProcessor：容灾，对应sinkgroups

# 自定义Iterceptor

POM：

```xml
<dependency>
    <groupId>org.apache.flume</groupId>
    <artifactId>flume-ng-core</artifactId>
    <version>1.7.0</version>
</dependency>
```

* 继承Iterceptor接口
* 实现其方法
* 定义内部类实现Builder
  * Builder implements Interceptor.Builder
  * build方法返回自定义拦截器类
* 打包上传至flume的lib目录下，使用时要用全类名

# 自定义Source

继承**AbstractSource**类并实现**Configurable**和**PollableSource ** 接口

* getBackOffSleepIncrement()//休眠递增值
* getMaxBackOffSleepInterval()//休眠间歇
* configure(Context context)//初始化context（读取配置文件内容）
* process()//获取数据封装成event并写入channel，这个方法将被循环调用。

打包上传至flume的lib目录下，使用时要用全类名

# 自定义Sink

继承**AbstractSink**类并实现**Configurable**接口

实现相应方法：

* configure(Context context)//初始化context（读取配置文件内容）
* process()//从Channel读取获取数据（event），这个方法将被循环调用。

打包上传至flume的lib目录下，使用时要用全类名

# 监控

```bash
# 安装http和php
sudo yum -y install httpd php
sudo yum -y install rrdtool perl-rrdtool rrdtool-devel
sudo yum -y install apr-devel
# 安装ganglia
sudo rpm -Uvh http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
sudo yum -y install ganglia-gmetad
sudo yum -y install ganglia-web
sudo yum install -y ganglia-gmond
```
```bash
sudo vim /etc/httpd/conf.d/ganglia.conf

# Ganglia monitoring system php web frontend
Alias /ganglia /usr/share/ganglia
<Location /ganglia>
  Order deny,allow
  #Deny from all
  Allow from all
  # Allow from 127.0.0.1
  # Allow from ::1
  # Allow from .example.com
</Location>
```
```bash
sudo vim /etc/ganglia/gmetad.conf

data_source "hadoop102" 192.168.1.102
```
```bash
sudo vim /etc/ganglia/gmond.conf 
#修改为：
cluster {
  name = "hadoop102"
  owner = "unspecified"
  latlong = "unspecified"
  url = "unspecified"
}
udp_send_channel {
  #bind_hostname = yes # Highly recommended, soon to be default.
                       # This option tells gmond to use a source address
                       # that resolves to the machine's hostname.  Without
                       # this, the metrics may appear to come from any
                       # interface and the DNS names associated with
                       # those IPs will be used to create the RRDs.
  # mcast_join = 239.2.11.71
  host = 192.168.1.102
  port = 8649
  ttl = 1
}
udp_recv_channel {
  # mcast_join = 239.2.11.71
  port = 8649
  bind = 192.168.1.102
  retry_bind = true
  # Size of the UDP buffer. If you are handling lots of metrics you really
  # should bump it up to e.g. 10MB or even higher.
  # buffer = 10485760
}
```
```bash
sudo vim /etc/selinux/config
#修改为：
# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=disabled
# SELINUXTYPE= can take one of these two values:
#     targeted - Targeted processes are protected,
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted
```
selinux本次生效关闭必须重启，如果此时不想重启，可以临时生效之：
```bash
sudo setenforce 0
```
启动命令
```bash
sudo service httpd start
sudo service gmetad start
sudo service gmond start
```

**打开网页浏览ganglia页面**

http://192.168.1.102/ganglia

提示：如果完成以上操作依然出现权限不足错误，请修改/var/lib/ganglia目录的权限：

```bash
$ sudo chmod -R 777 /var/lib/ganglia
```
启动flume任务
```bash
bin/flume-ng agent \
--conf conf/ \
--name a1 \
--conf-file job/flume-netcat-logger.conf \
-Dflume.root.logger==INFO,console \
-Dflume.monitoring.type=ganglia \
-Dflume.monitoring.hosts=192.168.1.102:8649
```
永久生效
```bash
JAVA_OPTS=
"-Dflume.monitoring.type=ganglia
-Dflume.monitoring.hosts=192.168.1.102:8649
-Xms100m
-Xmx200m"
```

