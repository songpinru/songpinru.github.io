## 节点退役方案:



#### 1.退役前检查HDFS块信息是否有错误,如果有错误的块,先修复

(PS:使用hdfs用户执行)

```
检查命令
hdfs fsck -list-corruptfileblocks -openforwrite -files -blocks -locations 
```

```
修复命令
hdfs debug  recoverLease  -path /path
```

#### 2.退役节点

CDH中搜索**dfs_hosts_exclude.txt**

增加要下线的节点ip

在黑名单中配置要下线的节点,然后保存并刷新hdfs(不会重启集群)

```
或者登录active namenode ，找到配置文件目录，修改dfs_hosts_exclude.txt
然后执行：
hadoop dfsadmin -refreshNodes 
```

此时下线节点状态变为decommissioning(可在web界面查看)

#### 3.等待节点下线完毕

可以在webUi查看,或使用以下命令

```
此命令可用于查看副本是否复制完毕:
hdfs dfsadmin -report -decommissioning
```

```bash
root@bjuc49:/home/ops# hdfs dfsadmin -report -decommissioning
Configured Capacity: 67358447177728 (61.26 TB)
Present Capacity: 66731792277188 (60.69 TB)
DFS Remaining: 19512827293507 (17.75 TB)
DFS Used: 47218964983681 (42.95 TB)
DFS Used%: 70.76%
Replicated Blocks:
	Under replicated blocks: 0
	Blocks with corrupt replicas: 0
	Missing blocks: 0
	Missing blocks (with replication factor 1): 0
	Low redundancy blocks with highest priority to recover: 0
	Pending deletion blocks: 6
Erasure Coded Block Groups: 
	Low redundancy block groups: 0
	Block groups with corrupt internal blocks: 0
	Missing block groups: 0
	Low redundancy blocks with highest priority to recover: 0
	Pending deletion blocks: 0

-------------------------------------------------
Decommissioning datanodes (0):
```

* Under replicated blocks: 副本数不完整的块(**用来查看下线进度**)
* Blocks with corrupt replicas: 副本错误的块
* Missing blocks: 丢失的块

#### 4.取消datanode授权即可