 

# 技术概述

对于多数应用来说，MySQL都是作为最关键的数据存储中心的，所以，如何让MySQL提供HA服务，是我们不得不面对的一个问题。MySQL的HA方案不止一种，本文介绍较为常用的一种——基于keepalived的MySQl HA。

## 1.1 MySQL主从复制

MySQL的HA离不开其主从复制的技术。主从复制是指一台服务器充当主数据库服务器（master），另一台或多台服务器充当从数据库服务器（slave），从服务器（slave）自动向主服务器（master）同步数据。实现MySQL的HA，需使两台服务器互为主从关系。

## 1.2 Keepalived

Keepalived是基于VRRP（Virtual Router Redundancy Protocol，虚拟路由器冗余协议）协议的一款高可用软件。Keepailived有一台主服务器（master）和多台备份服务器（backup），在主服务器和备份服务器上面部署相同的服务配置，使用一个虚拟IP地址对外提供服务，当主服务器出现故障时，虚拟IP地址会自动漂移到备份服务器。

# MySQL主从复制

## 2.1 一主一从

**（1）集群规划**

| hadoop102 | hadoop103                   | hadoop104                  |
| --------- | --------------------------- | -------------------------- |
|           | MySQL（master）| MySQL（slave） |

**（2）配置master**

​	1）修改**hadoop103**中MySQL的/usr/my.cnf配置文件。

```bash
[mysqld]

#开启binlog
log_bin = mysql-bin
#binlog日志类型
binlog_format = row
#MySQL服务器唯一id
server_id = 1
```

​	2）重启**hadoop103**的MySQL服务

```bash
sudo service mysql restart
```

​	3）进入mysql客户端，执行以下命令，查看master状态
```mysql
mysql> show master status;
```
**（3）配置slave**

​	1）修改**hadoop104**中MySQL的/usr/my.cnf配置文件

```bash
[mysqld]

#MySQL服务器唯一id
server_id = 2
#开启slave中继日志
relay_log=mysql-relay
```

​	2）重启**hadoop104**的MySQL服务

```bash
sudo service mysql restart
```

​	3）进入**hadoop104**的mysql客户端

执行以下命令:

```mysql
mysql>
CHANGE MASTER TO 
MASTER_HOST='bjuc49.optaim.com',
MASTER_USER='root',
MASTER_PASSWORD='123456',
MASTER_LOG_FILE='mariadb-bin.000004',
MASTER_LOG_POS=329;
```

> `MASTER_LOG_POS=`值为master的position值

​	4）启动slave
```mysql
mysql> start slave;
```
​	5）查看slave状态

```mysql
show slave status\G
```
>  \G 为横向显示

## 2.2 双主（互为主从即可）

**（1）集群规划**

| hadoop102 | hadoop103                              | hadoop104                              |
| --------- | -------------------------------------- | -------------------------------------- |
|           | **MySQL（master，slave）** | **MySQL（slave，master）** |

### 配置104
​	1）修改**hadoop104**中MySQL的/usr/my.cnf配置文件。

```bash
[mysqld]

#MySQL服务器唯一id
server_id = 2
#开启binlog
log_bin = mysql-bin
#binlog日志类型
binlog_format = row
#开启slave中继日志
relay_log=mysql-relay
```

​	2）重启**hadoop104**的MySQL服务

```bash
sudo service mysql restart
```
​	3）进入**hadoop104**的MySQL客户端，执行以下命令，查看master状态
```mysql
mysql> show master status;
```
### 配置103

​	1）修改**hadoop103**中MySQL的/usr/my.cnf配置文件
```bash
[mysqld]

#MySQL服务器唯一id
server_id = 1
#开启binlog
log_bin = mysql-bin
#binlog日志类型
binlog_format = row
#开启slave中继日志
relay_log=mysql-relay
```


​	2）重启**hadoop103**的MySQL服务
```bash
sudo service mysql restart
```
​	3）进入**hadoop103**的mysql客户端

执行以下命令
```mysql
mysql>
CHANGE MASTER TO 
MASTER_HOST='hadoop104',
MASTER_USER='root',
MASTER_PASSWORD='123456',
MASTER_LOG_FILE='mysql-bin.000001',
MASTER_LOG_POS=120;
```
> `MASTER_LOG_POS=`值为master的position值

​	4）启动slave

```mysql
mysql> start slave;
```
​	5）查看slave状态
```mysql
show slave status\G
```
>  \G 为横向显示
>  
# Keepalived

须在hadoop103，hadoop104两台节点上部署Keepalived。

## hadoop103

1）通过yum方式安装Keepalived
```bash
sudo yum install -y keepalived
```
2）修改Keepalived配置文件/etc/keepalived/keepalived.conf
```bash
! Configuration File for keepalived
global_defs {
    router_id MySQL-ha
}
vrrp_instance VI_1 {
    state master #初始状态
    interface eth0 #网卡
    virtual_router_id 51 #虚拟路由id
    priority 100 #优先级
    advert_int 1 #Keepalived心跳间隔
    nopreempt #只在高优先级配置，原master恢复之后不重新上位
    authentication {
        auth_type PASS #认证相关
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.1.100 #虚拟ip
    }
} 

#声明虚拟服务器
virtual_server 192.168.1.100 3306 {
    delay_loop 6
    persistence_timeout 30
    protocol TCP
    #声明真实服务器
    real_server 192.168.1.103 3306 {
        notify_down /var/lib/mysql/killkeepalived.sh #真实服务故障后调用脚本
        TCP_CHECK {
            connect_timeout 3 #超时时间
            nb_get_retry 1 #重试次数
            delay_before_retry 1 #重试时间间隔
        }
    }
}

```
3）编辑脚本文件/var/lib/mysql/killkeepalived.sh
```bash
#! /bin/bash
sudo service keepalived stop
```
4）加执行权限
```bash
sudo chmod +x /var/lib/mysql/killkeepalived.sh
```
5）启动Keepalived服务
```bash
sudo service keepalived start
```
6）设置Keepalived服务开机自启
```bash
sudo chkconfig keepalived on
```
7）确保开机时MySQL先于Keepalived启动

第一步：查看MySQL启动次序
```bash
sudo vim /etc/init.d/mysql
```
第二步：查看Keepalived启动次序
```bash
sudo vim /etc/init.d/keepalived
```
第三步：若Keepalived先于MySQL启动，则需要按照以下步骤设置二者启动顺序

1.修改/etc/init.d/mysql
```bash
sudo vim /etc/init.d/mysql
```


2.重新设置mysql开机自启
```bash
sudo chkconfig --del mysql
sudo chkconfig --add mysql
sudo chkconfig mysql on
```
3.修改/etc/init.d/keepalived
```bash
sudo vim /etc/init.d/keepalived
```

4.重新设置keepalived开机自启
```bash
sudo chkconfig --del keepalived
sudo chkconfig --add keepalived
sudo chkconfig keepalivedon
```
## hadoop104

## 更改优先级和真实ip即可



1）通过yum方式安装Keepalived
```bash
sudo yum install -y keepalived
```
2）修改Keepalived配置文件/etc/keepalived/keepalived.conf
```bash
! Configuration File for keepalived
global_defs {
    router_id MySQL-ha
}
vrrp_instance VI_1 {
    state backup #初始状态
    interface eth0 #网卡
    virtual_router_id 51 #虚拟路由id
    priority 50 #优先级
    advert_int 1 #Keepalived心跳间隔
    authentication {
        auth_type PASS #认证相关
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.1.100 #虚拟ip
    }
} 

#声明虚拟服务器
virtual_server 192.168.1.100 3306 {
    delay_loop 6
    persistence_timeout 30
    protocol TCP
    #声明真实服务器
    real_server 192.168.1.104 3306 {
        notify_down /var/lib/mysql/killkeepalived.sh #真实服务故障后调用脚本
        TCP_CHECK {
            connect_timeout 3 #超时时间
            nb_get_retry 1 #重试次数
            delay_before_retry 1 #重试时间间隔
        }
    }
}

```
3）编辑脚本文件/var/lib/mysql/killkeepalived.sh
```bash
#! /bin/bash
sudo service keepalived stop
```
4）加执行权限
```bash
sudo chmod +x /var/lib/mysql/killkeepalived.sh
```
5）启动Keepalived服务
```bash
sudo service keepalived start
```
6）设置Keepalived服务开机自启
```bash
sudo chkconfig keepalived on
```
7）确保开机时MySQL先于Keepalived启动

第一步：查看MySQL启动次序
```bash
sudo vim /etc/init.d/mysql
```
第二步：查看Keepalived启动次序
```bash
sudo vim /etc/init.d/keepalived
```

第三步：若Keepalived先于MySQL启动，则需要按照以下步骤设置二者启动顺序

1.修改/etc/init.d/mysql
```bash
sudo vim /etc/init.d/mysql
```

2.重新设置mysql开机自启
```bash
sudo chkconfig --del mysql
sudo chkconfig --add mysql
sudo chkconfig mysql on
```
3.修改/etc/init.d/keepalived
```bash
sudo vim /etc/init.d/keepalived
```

4.重新设置keepalived开机自启
```bash
sudo chkconfig --del keepalived
sudo chkconfig --add keepalived
sudo chkconfig keepalivedon
```