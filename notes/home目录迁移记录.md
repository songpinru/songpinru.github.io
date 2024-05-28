home目录迁移记录

在迁移之前，让大家不要操作home目录

`who`命令查看有谁在线

```shell
root@pbj-gw-20-81:/# who
ubuntu   pts/0        2021-07-05 14:14 (10.11.18.252)
gezhaoqiang pts/1        2021-06-24 15:54 (10.10.68.38)
ubuntu   pts/2        2021-07-05 14:25 (10.11.18.252)
ubuntu   pts/4        2021-07-05 14:19 (10.11.18.252)
fuzhihao pts/5        2021-07-05 14:29 (192.168.97.175)
ubuntu   pts/6        2021-07-05 10:27 (192.168.97.140)
ubuntu   pts/8        2021-07-05 10:41 (192.168.97.140)
ubuntu   pts/10       2021-07-05 11:38 (192.168.97.140)
```

首先需要把home里的数据拷贝到新的磁盘目录下

```shell
cp -r -p /home /opt/home
```

拷贝结束后，对home目录改名，并新建一个空的home目录

```shell
mv /home /home_bck
mkdir /home
```

把拷贝后的目录挂载到 /home下

```shell
mount --bind /opt/home /home
```

持久化，bind挂载重启后就无效，需要持久化这个操作

```shell
vim /etc/fstab

/opt/home	/home	none	bind	0	0
```

运行一段时间没问题的话，就删除`/home_bck`目录，释放系统盘的空间