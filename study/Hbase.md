# **架构角色：**

1. Region Server

Region Server为 Region的管理者，其实现类为HRegionServer，主要作用如下:

对于数据的操作：get, put, delete；

对于Region的操作：splitRegion、compactRegion。

2. Master

Master是所有Region Server的管理者，其实现类为HMaster，主要作用如下：

  对于表的操作：create, delete, alter

对于RegionServer的操作：分配regions到每个RegionServer，监控每个RegionServer的状态，负载均衡和故障转移。

3. Zookeeper

HBase通过Zookeeper来做Master的高可用、RegionServer的监控、元数据的入口以及集群配置的维护等工作。

4. HDFS

HDFS为HBase提供最终的底层数据存储服务，同时为HBase提供高可用的支持。



# 架构原理

1. StoreFile

保存实际数据的物理文件，StoreFile以HFile的形式存储在HDFS上。每个Store会有一个或多个StoreFile（HFile），数据在每个StoreFile中都是有序的。

Hfile是和列族绑定的,一个Hfile只有一个列族,因此需要修改的数据尽量自己一个列族(几个数据需要一起修改的话,可以放在一个列族里)

2. MemStore

其实就是Hflie的内存缓存部分,和HFile完全一致,也是和列族绑定

写缓存，由于HFile中的数据要求是有序的，所以数据是先存储在MemStore中，排好序后，等到达刷写时机才会刷写到HFile，每次刷写都会形成一个新的HFile。

3. WAL

由于数据要经MemStore排序后才能刷写到HFile，但把数据保存在内存中会有很高的概率导致数据丢失，为了解决这个问题，数据会先写在一个叫做Write-Ahead logfile的文件中，然后再写入MemStore中。所以在系统出现故障的时候，数据可以通过这个日志文件重建。

## 数据模型



1. Name Space和Table

命名空间，类似于关系型数据库的DatabBase概念，每个命名空间下有多个表。HBase有两个自带的命名空间，分别是hbase和default，hbase中存放的是HBase内置的表，default表是用户默认使用的命名空间

table是命名空间的细分,命名空间可以有多个表,每个表可以有多个region

元数据也是个表,zookeeper里记录了元数据表的region列表

2. Region

table的进一步拆分,每个region保存table的一部分数据

==Region 是HBase集群分布数据的最小单位==,即一个region只在一个节点上

3. Row

HBase表中的每行数据都由一个**RowKey**和多个**Column**（列）组成，数据是按照RowKey的字典顺序存储的，并且查询数据时只能根据RowKey进行检索，所以RowKey的设计十分重要。

4. Column Family

HBase中的每个列都由Column Family(列族)和Column Qualifier（列限定符）进行限定，例如info：name，info：age。建表时，只需指明列族，而列限定符无需预先定义。

5. Time Stamp

用于标识数据的不同版本（version），每条数据写入时，如果不指定时间戳，系统会自动为其加上该字段，其值为写入HBase的时间。

6. Cell

由{rowkey, column Family：column Qualifier, time Stamp} 唯一确定的单元。cell中的数据是没有类型的，全部是字节码形式存贮。

##  写流程

写流程：

1）Client先访问zookeeper，获取hbase:meta表位于哪个Region Server。

2）访问对应的Region Server，获取hbase:meta表，根据读请求的namespace:table/rowkey，查询出目标数据位于哪个Region Server中的哪个Region中。并将该table的region信息以及meta表的位置信息缓存在客户端的meta cache，方便下次访问。

3）与目标Region Server进行通讯；

4）将数据顺序写入（追加）到WAL；

5）将数据写入对应的MemStore，数据会在MemStore进行排序；

6）向客户端发送ack；

7）等达到MemStore的刷写时机后，将数据刷写到HFile。

## MemStore Flush

**MemStore刷写时机**

1.当某个memstroe的大小达到了***hbase.hregion.memstore.flush.size***（默认值128M），其所在region的所有memstore都会刷写。

当memstore的大小达到了

**hbase.hregion.memstore.flush.size**（默认值128M） **hbase.hregion.memstore.block.multiplier**（默认值4）

时，会阻止继续往该memstore写数据。

2.当region server中memstore的总大小达到

**java_heapsize**

***hbase.regionserver.global.memstore.size***（默认值0.4）

***hbase.regionserver.global.memstore.size.lower.limit***（默认值0.95），

region会按照其所有memstore的大小顺序（由大到小）依次进行刷写。直到region server中所有memstore的总大小减小到上述值以下。

当region server中memstore的总大小达到

***java_heapsize\*hbase.regionserver.global.memstore.size***（默认值0.4）

时，会阻止继续往所有的memstore写数据。

\3. 到达自动刷写的时间，也会触发memstore flush。自动刷新的时间间隔由该属性进行配置**hbase.regionserver.optionalcacheflushinterval****（默认1****小时）**。

4.当WAL文件的数量超过**hbase.regionserver.max.logs**，region会按照时间顺序依次进行刷写，直到WAL文件数量减小到**hbase.regionserver.max.log**以下（该属性名已经废弃，现无需手动设置，最大值为32）。

## 读流程

 

**读流程**

1）Client先访问zookeeper，获取hbase:meta表位于哪个Region Server。

2）访问对应的Region Server，获取hbase:meta表，根据读请求的namespace:table/rowkey，查询出目标数据位于哪个Region Server中的哪个Region中。并将该table的region信息以及meta表的位置信息缓存在客户端的meta cache，方便下次访问。

3）与目标Region Server进行通讯；

4）分别在Block Cache（读缓存），MemStore和Store File（HFile）中查询目标数据，并将查到的所有数据进行合并。此处所有数据是指同一条数据的不同版本（time stamp）或者不同的类型（Put/Delete）。

5） 将从文件中查询到的数据块（Block，HFile数据存储单元，默认大小为64KB）缓存到Block Cache。

6）将合并后的最终结果返回给客户端。

## StoreFile Compaction

由于memstore每次刷写都会生成一个新的HFile，且同一个字段的不同版本（timestamp）和不同类型（Put/Delete）有可能会分布在不同的HFile中，因此查询时需要遍历所有的HFile。为了减少HFile的个数，以及清理掉过期和删除的数据，会进行StoreFile Compaction。

Compaction分为两种，分别是Minor Compaction和Major Compaction。Minor Compaction会将临近的若干个较小的HFile合并成一个较大的HFile，但**不会**清理过期和删除的数据。Major Compaction会将一个Store下的所有的HFile合并成一个大HFile，并且**会**清理掉过期和删除的数据。

 

##  Region Split

默认情况下，每个Table起初只有一个Region，随着数据的不断写入，Region会自动进行拆分。刚拆分时，两个子Region都位于当前的Region Server，但处于负载均衡的考虑，HMaster有可能会将某个Region转移给其他的Region Server。

Region Split时机：

1.当1个region中的某个Store下所有StoreFile的总大小超过hbase.hregion.max.filesize，该Region就会进行拆分（0.94版本之前）。

2.当1个region中的某个Store下所有StoreFile的总大小超过Min(R^2 * "hbase.hregion.memstore.flush.size",hbase.hregion.max.filesize")，该Region就会进行拆分，其中R为当前Region Server中属于该Table的个数（0.94版本之后）。



#  基本操作

1. 进入HBase客户端命令行

bin/hbase shell

2. 查看帮助命令

hbase(main):001:0> help

3. 查看当前数据库中有哪些表

hbase(main):002:0> list

## 表的操作

1. **创建表**

hbase(main):002:0> create 'student','info'

2. **插入数据到表**

hbase(main):003:0> put 'student','1001','info:sex','male'

hbase(main):004:0> put 'student','1001','info:age','18'

hbase(main):005:0> put 'student','1002','info:name','Janna'

hbase(main):006:0> put 'student','1002','info:sex','female'

hbase(main):007:0> put 'student','1002','info:age','20'

3. **扫描查看表数据**

hbase(main):008:0> scan 'student'

hbase(main):009:0> scan 'student',{STARTROW => '1001', STOPROW => '1001'}

hbase(main):010:0> scan 'student',{STARTROW => '1001'}

4. **查看表结构**

hbase(main):011:0> describe ‘student’

5. **更新指定字段的数据**

hbase(main):012:0> put 'student','1001','info:name','Nick'

hbase(main):013:0> put 'student','1001','info:age','100'

6. **查看“指定行”或“指定列族:列”的数据**

hbase(main):014:0> get 'student','1001'

hbase(main):015:0> get 'student','1001','info:name'

7. **统计表数据行数**

hbase(main):021:0> count 'student'

8. **删除数据**

删除某rowkey的全部数据：

hbase(main):016:0> deleteall 'student','1001'

删除某rowkey的某一列数据：

hbase(main):017:0> delete 'student','1002','info:sex'

9. **清空表数据**

hbase(main):018:0> truncate 'student'

提示：清空表的操作顺序为先disable，然后再truncate。

10. **删除表**

首先需要先让该表为disable状态：

hbase(main):019:0> disable 'student'

然后才能drop这个表：

hbase(main):020:0> drop 'student'

提示：如果直接drop表，会报错：ERROR: Table student is enabled. Disable it first.

11. **变更表信息**

将info列族中的数据存放3个版本：

hbase(main):022:0> alter 'student',{NAME=>'info',VERSIONS=>3}

hbase(main):022:0> get 'student','1001',{COLUMN=>'info:name',VERSIONS=>3}



#  HBaseAPI

```xml
<dependency>
    <groupId>org.apache.hbase</groupId>
    <artifactId>hbase-client</artifactId>
    <version>2.1.10</version>
</dependency>
```

1. 创建config,配置zookeeper
2. 使用config获得连接
3. 获取Admin或HTable
4. 操作

### 获取Configuration对象


```java
public static  Configuration conf; 
static{    
    //使用HBaseConfiguration的单例方法实例化    
    conf = HBaseConfiguration.create();  
    conf.set("hbase.zookeeper.quorum",  "192.166.9.102"); 
    conf.set("hbase.zookeeper.property.clientPort",  "2181");
}  
```
### 判断表是否存在

```java
public static boolean  isTableExist(String tableName) throws MasterNotRunningException,   ZooKeeperConnectionException, IOException{    
    //在HBase中管理、访问表需要先创建HBaseAdmin对象  
    Connection connection  = ConnectionFactory.createConnection(conf);  
    HBaseAdmin admin = (HBaseAdmin)  connection.getAdmin();    
    //HBaseAdmin admin = new HBaseAdmin(connection);    
    return admin.tableExists(tableName);  
} 
```

### 创建表

```java
public static void  createTable(String tableName, String... columnFamily) throws MasterNotRunningException,  ZooKeeperConnectionException, IOException{    
    HBaseAdmin admin = new HBaseAdmin(conf);    
    //判断表是否存在    
    if(isTableExist(tableName)){     
        System.out.println("表" +  tableName + "已存在");     
        //System.exit(0);    
    }else{     
        //创建表属性对象,表名需要转字节     
        HTableDescriptor descriptor = new  HTableDescriptor(TableName.valueOf(tableName));     
        //创建多个列族     
        for(String cf : columnFamily){       
            descriptor.addFamily(new HColumnDescriptor(cf));     
        }     
        //根据对表的配置，创建表     
        admin.createTable(descriptor);     
        System.out.println("表" +  tableName + "创建成功！");    
    }  
}  
```

### 删除表

```java
 public static void  dropTable(String tableName) throws MasterNotRunningException,   ZooKeeperConnectionException, IOException{    
     HBaseAdmin admin = new HBaseAdmin(conf);    
     if(isTableExist(tableName)){     
         admin.disableTable(tableName);     
         admin.deleteTable(tableName);     
         System.out.println("表" +  tableName + "删除成功！");    
     }else{     
         System.out.println("表" +  tableName + "不存在！");    
     }  
 }  
```

###  向表中插入数据

```java
public static void  addRowData(String tableName, String rowKey, String columnFamily, String   column, String value) throws IOException{    
    //创建HTable对象    
    TableName test = TableName.valueOf("test");
    Htable hTable=connection.getTable(test);
    //向表中插入数据    
    Put put = new Put(Bytes.toBytes(rowKey));    
    //向Put对象中组装数据    
    put.add(Bytes.toBytes(columnFamily), Bytes.toBytes(column),  Bytes.toBytes(value));    
    hTable.put(put);    
    hTable.close();    
    System.out.println("插入数据成功");  
}  
```

### 删除多行数据

```java
 public static void  deleteMultiRow(String tableName, String... rows) throws IOException{   
    TableName test = TableName.valueOf("test");
    Htable hTable=connection.getTable(test);  
     List<Delete> deleteList = new ArrayList<Delete>();    
     for(String row : rows){     
         Delete delete = new Delete(Bytes.toBytes(row));     
         deleteList.add(delete);    
     }    
     hTable.delete(deleteList);    
     hTable.close();  
 }  
```

###  获取所有数据

```java
public static void  getAllRows(String tableName) throws IOException{    
    TableName test = TableName.valueOf("test");
    Htable hTable=connection.getTable(test);    
    //得到用于扫描region的对象    
    Scan scan = new Scan();    
    //使用HTable得到resultcanner实现类的对象    
    ResultScanner resultScanner = hTable.getScanner(scan);    
    for(Result result : resultScanner){     
        Cell[] cells = result.rawCells();     
        for(Cell cell : cells){       
            //得到rowkey       
            System.out.println("行键:" +  Bytes.toString(CellUtil.cloneRow(cell)));       
            //得到列族       
            System.out.println("列族" + Bytes.toString(CellUtil.cloneFamily(cell)));       
            System.out.println("列:" + Bytes.toString(CellUtil.cloneQualifier(cell)));       
            System.out.println("值:" + Bytes.toString(CellUtil.cloneValue(cell)));     
        }    
    }  
}                                                                   
```

###  获取某一行数据

```java
public static void  getRow(String tableName, String rowKey) throws IOException{    
    HTable table = new HTable(conf, tableName);    
    Get get = new Get(Bytes.toBytes(rowKey));    
    //get.setMaxVersions();显示所有版本    
    //get.setTimeStamp();显示指定时间戳的版本    
    Result result = table.get(get);    
    for(Cell cell : result.rawCells()){     
        System.out.println("行键:" +  Bytes.toString(result.getRow()));     
        System.out.println("列族" +  Bytes.toString(CellUtil.cloneFamily(cell)));
        System.out.println("列:" +  Bytes.toString(CellUtil.cloneQualifier(cell)));
        System.out.println("值:" +  Bytes.toString(CellUtil.cloneValue(cell))); 
        System.out.println("时间戳:" +  cell.getTimestamp());  
    } 
}  
```

###  获取某一行指定“列族:列”的数据

```java
public static void  getRowQualifier(String tableName, String rowKey, String family, String   qualifier) throws IOException{    
    TableName test = TableName.valueOf("test");
    Htable hTable=connection.getTable(test);
    Get get = new Get(Bytes.toBytes(rowKey));  
    get.addColumn(Bytes.toBytes(family), Bytes.toBytes(qualifier)); 
    Result result = table.get(get);  
    for(Cell cell : result.rawCells()){      
        System.out.println("行键:" + Bytes.toString(result.getRow()));  
        System.out.println("列族" +  Bytes.toString(CellUtil.cloneFamily(cell)));  
        System.out.println("列:" +  Bytes.toString(CellUtil.cloneQualifier(cell)));  
        System.out.println("值:" +  Bytes.toString(CellUtil.cloneValue(cell)));  
    }
}  
```

# 基础优化

1．允许在HDFS的文件中追加内容

hdfs-site.xml、hbase-site.xml

  属性：dfs.support.append  解释：开启HDFS追加同步，可以优秀的配合HBase的数据同步和持久化。默认值为true。  

2．优化DataNode允许的最大文件打开数

hdfs-site.xml

  属性：dfs.datanode.max.transfer.threads  解释：HBase一般都会同一时间操作大量的文件，根据集群的数量和规模以及数据动作，设置为4096或者更高。默认值：4096  

3．优化延迟高的数据操作的等待时间

hdfs-site.xml

  属性：dfs.image.transfer.timeout  解释：如果对于某一次数据操作来讲，延迟非常高，socket需要等待更长的时间，建议把该值设置为更大的值（默认60000毫秒），以确保socket不会被timeout掉。  

4．优化数据的写入效率

mapred-site.xml

  属性：  mapreduce.map.output.compress  mapreduce.map.output.compress.codec  解释：开启这两个数据可以大大提高文件的写入效率，减少写入时间。第一个属性值修改为true，第二个属性值修改为：org.apache.hadoop.io.compress.GzipCodec或者其他压缩方式。  

5．设置RPC监听数量

hbase-site.xml

  属性：Hbase.regionserver.handler.count  解释：默认值为30，用于指定RPC监听的数量，可以根据客户端的请求数进行调整，读写请求较多时，增加此值。  

6．优化HStore文件大小

hbase-site.xml

  属性：hbase.hregion.max.filesize  解释：默认值10737418240（10GB），如果需要运行HBase的MR任务，可以减小此值，因为一个region对应一个map任务，如果单个region过大，会导致map任务执行时间过长。该值的意思就是，如果HFile的大小达到这个数值，则这个region会被切分为两个Hfile。  

7．优化HBase客户端缓存

hbase-site.xml

  属性：hbase.client.write.buffer  解释：用于指定Hbase客户端缓存，增大该值可以减少RPC调用次数，但是会消耗更多内存，反之则反之。一般我们需要设定一定的缓存大小，以达到减少RPC次数的目的。  

8．指定scan.next扫描HBase所获取的行数

hbase-site.xml

  属性：hbase.client.scanner.caching  解释：用于指定scan.next方法获取的默认行数，值越大，消耗内存越大。  

9．flush、compact、split机制

当MemStore达到阈值，将Memstore中的数据Flush进Storefile；compact机制则是把flush出来的小文件合并成大的Storefile文件。split则是当Region达到阈值，会把过大的Region一分为二。

**涉及属性：**

即：128M就是Memstore的默认阈值

  hbase.hregion.memstore.flush.size：134217728  

即：这个参数的作用是当单个HRegion内所有的Memstore大小总和超过指定值时，flush该HRegion的所有memstore。RegionServer的flush是通过将请求添加一个队列，模拟生产消费模型来异步处理的。那这里就有一个问题，当队列来不及消费，产生大量积压请求时，可能会导致内存陡增，最坏的情况是触发OOM。

  hbase.regionserver.global.memstore.upperLimit：0.4  hbase.regionserver.global.memstore.lowerLimit：0.38  

即：当MemStore使用内存总量达到hbase.regionserver.global.memstore.upperLimit指定值时，将会有多个MemStores flush到文件中，MemStore flush 顺序是按照大小降序执行的，直到刷新到MemStore使用内存略小于lowerLimit