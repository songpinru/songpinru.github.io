# 原理

1）Master主库将改变记录写到二进制日志(binary log)中；

2）Slave从库向mysql master发送dump协议，将master主库的binary log events拷贝到它的中继日志(relay log)；

3）Slave从库读取并重做中继日志中的事件，将改变的数据同步到自己的数据库。

 

**Canal的工作原理很简单，就是把自己伪装成Slave，假装从Master复制数据。**

#  安装

## Binlog的分类设置

修改MySQL的/usr/my.cnf配置文件。

```bash
[mysqld]

#开启binlog
log_bin = mysql-bin
#binlog日志类型
binlog_format = row
#MySQL服务器唯一id
server_id = 1
```

```mysql
mysql> GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'%' IDENTIFIED BY 'canal';
```

MySQL Binlog的格式，那就是有三种，分别是STATEMENT,MIXED,ROW。

区别：

1）statement

​    语句级，binlog会记录每次一执行写操作的语句。

​    相对row模式节省空间，但是可能产生不一致性，比如1

update  tt set create_date=now()

​    如果用binlog日志进行恢复，由于执行时间不同可能产生的数据就不同。

​    优点：节省空间

​    缺点：有可能造成数据不一致。

2）row

​    行级，binlog会记录每次操作后每行记录的变化。

​    优点：保持数据的绝对一致性。因为不管sql是什么，引用了什么函数，他只记录执行后的效果。

​    缺点：占用较大空间。

3）mixed

​    statement的升级版，一定程度上解决了，因为一些情况而造成的statement模式不一致问题

​    在某些情况下譬如：

​    当函数中包含 UUID() 时；

​    包含 AUTO_INCREMENT 字段的表被更新时；

​    执行 INSERT DELAYED 语句时；

​    用 UDF 时；

​    会按照 ROW的方式进行处理

​    优点：节省空间，同时兼顾了一定的一致性。

缺点：还有些极个别情况依旧会造成不一致，

另外statement和mixed对于需要对binlog的监控的情况都不方便。

## Canal部署

解压

修改配置

```bash
#canal的配置
vim conf/canal.properties

canal.id = 1
canal.ip =
canal.port = 11111
canal.metrics.pull.port = 11112
canal.zkServers =
# 当前server上部署的instance列表
canal.destinations=
```

```bash
# instance配置，即每个应用的具体配置
vim conf/example/instance.properties

# 不能与mysql的server-id相同
canal.instance.mysql.slaveId=0

# enable gtid use true/false
canal.instance.gtidon=false

# position info，数据库信息
canal.instance.master.address=127.0.0.1:3306
canal.instance.master.journal.name=
canal.instance.master.position=
canal.instance.master.timestamp=
canal.instance.master.gtid=# username/password
canal.instance.dbUsername=canal
canal.instance.dbPassword=canal
canal.instance.connectionCharset = UTF-8
canal.instance.defaultDatabaseName =test
# enable druid Decrypt database password
canal.instance.enableDruid=false
```

[参数配置](https://blog.csdn.net/u012758088/article/details/78789616)

启动

```bash
./bin/startup.sh
```

# API

| **对象名称**  | **介绍**                                                     | **包含内容**                                                 |
| ------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **message**   | 一次canal从日志中抓取的信息，一个message包含多个sql(event)   | 包含 一个Entry集合                                           |
| **entry**     | 对应一个sql命令，一个sql可能会对多行记录造成影响。           | 序列化的数据内容storeValue                                   |
| **rowchange** | 是把entry中的storeValue反序列化的对象。                      | 1 rowdatalist    行集    <br>2 eventType(数据的变化类型:  insert update delete create  alter drop) |
| **RowData**   | 出现变化的数据行信息                                         | 1 afterColumnList (修改后)   <br>  2 beforeColumnList（修改前） |
| **column**    | 一个RowData里包含了多个column，每个column包含了   name和 value | 1 columnName   <br>2 columnValue                             |

POM:

```xml
<dependency>
    <groupId>com.alibaba.otter</groupId>
    <artifactId>canal.client</artifactId>
    <version>1.1.2</version>
</dependency>
```

```java
import com.alibaba.fastjson.JSON;
import com.alibaba.otter.canal.client.CanalConnector;
import com.alibaba.otter.canal.client.CanalConnectors;
import com.alibaba.otter.canal.protocol.CanalEntry;
import com.alibaba.otter.canal.protocol.Message;
import com.atguigu.utils.MyKafkaSender;
import com.google.protobuf.ByteString;
import com.google.protobuf.InvalidProtocolBufferException;

import java.net.InetSocketAddress;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public class CanalClient {
    public static void main(String[] args){
        //1、获得单连接
        CanalConnector canalConnector = CanalConnectors.newSingleConnector(new InetSocketAddress("hadoop102", 11111), "example", "", "");
        //2、长轮询
        while (true){
            canalConnector.connect();
            canalConnector.subscribe("gmall.*");
            Message message = canalConnector.get(100);
            List<CanalEntry.Entry> entries = message.getEntries();
            if(entries.size()<=0){
                try {
                    Thread.sleep(5000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }else {
                for (CanalEntry.Entry entry : entries) {
                    if (CanalEntry.EntryType.ROWDATA.equals(entry.getEntryType())  ) {
                            try {
                                ByteString storeValue = entry.getStoreValue();
                                CanalEntry.RowChange rowChange = CanalEntry.RowChange.parseFrom(storeValue);
                                List<CanalEntry.RowData> rowDatasList = rowChange.getRowDatasList();
                                String tableName = entry.getHeader().getTableName();
                                CanalEntry.EventType eventType = rowChange.getEventType();
                                if ("order_info"==tableName && CanalEntry.EventType.INSERT.equals(eventType)) {
                                    handler(GmallConstants.GMALL_ORDER_INFO,rowDatasList);
                                }else if("order_detail"==tableName && CanalEntry.EventType.INSERT.equals(eventType)){
                                    handler(GmallConstants.GMALL_ORDER_DETAIL,rowDatasList);
                                }else if("user_info"==tableName && (CanalEntry.EventType.INSERT.equals(eventType) ||CanalEntry.EventType.UPDATE.equals(eventType)) ){
                                    handler(GmallConstants.GMALL_USER_INFO,rowDatasList);
                                }
                            } catch (InvalidProtocolBufferException e) {
                                e.printStackTrace();
                            }
                    }
                }
            }
        }
    }

    private static void handler(String topic, List<CanalEntry.RowData> rowDatasList) {
            for (CanalEntry.RowData rowData : rowDatasList) {
                List<CanalEntry.Column> afterColumnsList = rowData.getAfterColumnsList();
                Map<String, String> map = new HashMap<>();
//                JSONObject map = new JSONObject();
                for (CanalEntry.Column column : afterColumnsList) {
                    map.put(column.getName(),column.getValue());
                }
                MyKafkaSender.send(topic, JSON.toJSONString(map));
            }
    }
}

```

