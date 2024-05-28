HDFS 临时维护操作方法

# 背景

HDFS会自动维持副本数量，默认检查间隔是每3s一次，当发现集群中有块的副本数不足时，会触发副本自动复制，来保证数据的安全性。

但是当我们需要临时停机某个DataNode进行维护时（比如需要重启节点时），这个副本自动复制的机制就有点太过于灵活且耗费资源了，这个机制会消耗大量的网络和磁盘io来进行副本复制，此时我们知道我们的数据是没有丢失的（或者并没有全部丢失，不需要全量复制），我们希望能够暂时不要进行副本复制，尽量不要影响线上业务运行，此时我们可以利用HDFS的退役机制来完成这个功能。

# 实现原理

HDFS使用黑名单退役时（只加入黑名单即可），DateNode节点的状态变化：

* `NORMAL` 
* `DECOMMISSION_INPROGRESS`
* `DECOMMISSIONED`
* `ENTERING_MAINTENANCE` 
* `IN_MAINTENANCE` 

加入黑名单后，该节点的状态会变为`IN_MAINTENANCE`,即维护状态，此时该节点上的数据也会处于维护状态，不会触发副本的自动复制机制，DataNode也不会把维护状态的Block返回给应用程序

当节点维护完成，只需把节点从黑名单中排除，然后refreshNodes，让其成为正常节点，即可退出维护模式

# 操作方法

## 进入维护模式

### 方法一：手动

登录到NameNode节点,找到NameNode的配置文件夹

```shell
root@bjuc51:/home/ops# cd /var/run/cloudera-scm-agent/process/ 
root@bjuc51:/var/run/cloudera-scm-agent/process# ll|grep NAMENODE 
drwxr-x--x  3 hdfs   hdfs   540 Sep 26 10:44 5438-hdfs-NAMENODE/
drwxr-x--x  3 hdfs   hdfs   560 Sep 27 10:46 5442-hdfs-NAMENODE-nnRpcWait/ 
drwxr-x--x  3 hdfs   hdfs   560 Oct 23 15:41 5646-hdfs-NAMENODE-refresh/
drwxr-x--x  3 hdfs   hdfs   560 Oct 28 11:52 5701-hdfs-NAMENODE-refresh/
```

前缀id最大的文件夹即当前运行的NameNode配置文件夹

```shell
cd 5701-hdfs-NAMENODE-refresh/
vim hdfs-site.xml
```
首先查看有没有配置以下两项：

以下两个配置可以启用新的节点管理方式（可以使用json设置节点状态）

```xml
<property>
    <name>dfs.hosts</name>
    <value>/var/run/cloudera-scm-agent/process/5701-hdfs-NAMENODE-refresh/dfs_all_hosts.txt</value>
</property>
<property>
    <name>dfs.namenode.hosts.provider.classname</name>
    <value>org.apache.hadoop.hdfs.server.blockmanagement.CombinedHostFileManager</value>
</property>
```



编辑`dfs_all_hosts.txt`文件，把需要维护的节点的`adminState`添加改为`IN_MAINTENANCE`

```json
vim dfs_all_hosts.txt

{"hostName":"10.11.20.49","port":9866,"adminState":"NORMAL"}
{"hostName":"10.11.20.50","port":9866,"adminState":"NORMAL"}
{"hostName":"10.11.20.53","port":9866,"adminState":"NORMAL"}
{"hostName":"10.11.20.54","port":9866,"adminState":"NORMAL"}
{"hostName":"10.11.20.55","port":9866,"adminState":"NORMAL"}
{"hostName":"10.11.20.101","port":9866,"adminState":"NORMAL"}
{"hostName":"10.11.20.102","port":9866,"adminState":"NORMAL"}
{"hostName":"10.11.20.114","port":9866,"adminState":"NORMAL"}
{"hostName":"10.11.20.115","port":9866,"adminState":"NORMAL"}
{"hostName":"10.11.20.116","port":9866,"adminState":"NORMAL"}
{"hostName":"10.11.20.69","port":9866,"adminState":"NORMAL"}
{"hostName":"10.11.20.70","port":9866,"adminState":"NORMAL"}
{"hostName":"10.11.20.71","port":9866,"adminState":"NORMAL"}
{"hostName":"10.11.20.72","port":9866,"adminState":"NORMAL"}
--{"hostName":"10.11.20.73","port":9866,"adminState":"NORMAL"}
++{"hostName":"10.11.20.73","port":9866,"adminState":"IN_MAINTENANCE"}
```

在当前目录下执行

```shell
hdfs --config /var/run/cloudera-scm-agent/process/5701-hdfs-NAMENODE-refresh dfsadmin -refreshNodes
```

> 注意此处config的参数为找到的NameNode配置文件夹
>
> 

之后等待节点进入`ENTERING_MAINTENANCE` 状态，可以使用以下命令查看：

```shell
hdfs dfsadmin -report -enteringmaintenance
hdfs dfsadmin -report -inmaintenance
```

进入维护状态的节点可以下线停机进行维护，此时不会触发副本的自动复制

当维护结束时，确认节点已连接回NameNode后，只需推出维护模式即可

### 方法二：CDH刷新NameNode

借助CDH管理界面，进入

HDFS->配置->NameNode->dfs_all_hosts.txt

![image-20220117153035750](HDFS%E7%BB%B4%E6%8A%A4%E8%A7%84%E8%8C%83.assets/image-20220117153035750.png)

增加`{"hostName":"10.11.20.73","port":9866,"adminState":"IN_MAINTENANCE"}`

然后执行：

HDFS->实例->NameNode(活动)->操作->刷新节点列表

![image-20211028144500187](HDFS%E7%BB%B4%E6%8A%A4%E8%A7%84%E8%8C%83.assets/image-20211028144500187.png)

等待刷新完成即可

## 退出维护模式

### 方法一：手动

和进入维护模式一样，找到NameNode配置文件夹

修改`dfs_all_hosts.txt`文件内容，改回退出维护模式的节点为`NORMAL`

然后执行：

```shell
hdfs --config /var/run/cloudera-scm-agent/process/5701-hdfs-NAMENODE-refresh dfsadmin -refreshNodes
```

> 注意此处config的参数为找到的NameNode配置文件夹

### 方法二：CDH刷新NameNode

借助CDH管理界面，进入

HDFS->实例->NameNode(活动)->操作->刷新节点列表

![image-20211028144500187](HDFS%E7%BB%B4%E6%8A%A4%E8%A7%84%E8%8C%83.assets/image-20211028144500187.png)

等待刷新完成即可
