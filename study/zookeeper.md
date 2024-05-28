# 安装

1. 解压
2. 把conf文件夹下配置文件改个名字
```bash
cp zoo_sample.cfg zoo.cfg
```
3. 编辑zoo.cfg,配置datadir
```bash
dataDir=/opt/module/zookeeper-3.4.10/zkData
```
4. 配置集群机器，每台机器分配一个不同的Serverid
```bash
tickTime=2000
dataDir=/opt/module/zookeeper-3.4.10/zkData
clientPort=2181
initLimit=5
syncLimit=2
server.2=hadoop102:2888:3888
server.3=hadoop103:2888:3888
server.4=hadoop104:2888:3888
 # 以上配置2，3，4就是Serverid
```
5. 在zkData文件夹里新建一个myid文件，内容是本机的Serverid
6. 配置Zookeeper的LogDIR：配置bin/zkEnv.sh文件
```bash
ZOO_LOG_DIR="."改为/opt/module/zookeeper-3.4.10/logs
```
7. 分发，其他机器修改myid
8. 启动zookeeper
```bash
bin/zkServer.sh start
```
群起脚本
```bash
#! /bin/bash

case $1 in
"start"){
	for i in hadoop102 hadoop103 hadoop104
	do
		ssh $i "/opt/module/zookeeper-3.4.10/bin/zkServer.sh start"
	done
};;
"stop"){
	for i in hadoop102 hadoop103 hadoop104
	do
		ssh $i "/opt/module/zookeeper-3.4.10/bin/zkServer.sh stop"
	done
};;
"status"){
	for i in hadoop102 hadoop103 hadoop104
	do
		ssh $i "/opt/module/zookeeper-3.4.10/bin/zkServer.sh status"
	done
};;
esac
```

# 选举机制

半数机制：集群中半数以上机器存活，集群可用。所以Zookeeper适合安装奇数台服务器。

[用zookeeper实现HA](https://blog.csdn.net/zhongqi2513/article/details/92840447)

# 监听原理



zookeeper使用Watcher做监听

* ZooKeeper 构造方法:
  * 全局监控,并且永久存在,每次zookeeper的改动都会通知
*  getData、exists、getChildren :
  * 对path做单次监控，想要一直监控需要在process中再次注册

