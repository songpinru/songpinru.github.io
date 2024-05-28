# 概念

## 架构

* Client（客户端）
  * 解析器
  * 编译器
  * 优化器
  * 执行器
* Metastore（元数据）
* HDFS（数据存储）
* MapReduce（计算引擎）

## 数据类型

| Hive数据类型 | Java数据类型 | 长度                                                 | 例子                                 |
| ------------ | ------------ | ---------------------------------------------------- | ------------------------------------ |
| TINYINT      | byte         | 1byte有符号整数                                      | 20                                   |
| SMALINT      | short        | 2byte有符号整数                                      | 20                                   |
| INT          | int          | 4byte有符号整数                                      | 20                                   |
| BIGINT       | long         | 8byte有符号整数                                      | 20                                   |
| BOOLEAN      | boolean      | 布尔类型，true或者false                              | TRUE  FALSE                          |
| FLOAT        | float        | 单精度浮点数                                         | 3.14159                              |
| DOUBLE       | double       | 双精度浮点数                                         | 3.14159                              |
| STRING       | string       | 字符系列。可以指定字符集。可以使用单引号或者双引号。 | ‘now is the time’ “for all good men” |
| TIMESTAMP    |              | 时间类型                                             |                                      |
| BINARY       |              | 字节数组                                             |                                      |

## 集合类型

| 数据类型 | 描述                                  | 语法示例                                            |
| -------- | ------------------------------------- | --------------------------------------------------- |
| STRUCT   | 结构化数据，使用`.`  来获得元素       | struct()   例如struct<street:string,   city:string> |
| MAP      | K-V键值对，那么可以通过字段名获取元素 | map()   例如map<string, int>                        |
| ARRAY    | 通过数组下标获取元素                  | Array()   例如array<string>                         |

# 安装

```
1.安装hive
	（1）解压tar包
	（2）修改hive-env.sh
		修改HADOOP_HOME和HIVE_CONF_DIR
2.替换数据库为mysql
	（1）先查询卸载原有的mysql数据库
		rpm -qa|grep mysql
		sudo rpm -e --nodeps mysql-libs-5.1.73-7.el6.x86_64
	（2）安装server（记得用sudo）
		解压unzip mysql-libs.zip
		安装mysql服务端：sudo rpm -ivh MySQL-server-5.6.24-1.el6.x86_64.rpm
		查询mysql的运行状态：sudo service mysql status
		启动mysql的服务端：sudo service mysql start
	（3）安装clinet（修改密码）：
		sudo rpm -ivh MySQL-client-5.6.24-1.el6.x86_64.rpm
		登陆mysql修改密码：mysql -uroot -p密码
		修改mysql的密码：set password = password（‘000000’）;
	（4）修改mysql的权限
		修改mysql数据库下的user表
		update user set host='%' where host='localhost';
		delete from user where Host='hadoop102';
		delete from user where Host='127.0.0.1';
		delete from user where Host='::1';
		（记得flush privileges）
	（5）拷贝mysql的jdbc的驱动到hive的lib
		cp mysql-connector-java-5.1.27-bin.jar /opt/module/hive/lib/（注意这块拷贝的是jar包）
	（6）修改hive的配置文件
		创建hive-site.xml
		添加关于mysql的四个字符串的配置
		检测是否可以多窗口访问hive
3.在mysql查看metastore数据库
	重点关注DBS，TBLS，COLUMN_V2表
```



#  DDL

## 创建数据库

```sql
CREATE DATABASE [IF NOT EXISTS] database_name
[COMMENT database_comment]
[LOCATION hdfs_path]
[WITH DBPROPERTIES (property_name=property_value, ...)];
```

数据库命令

```sql
--显示数据库
show databases;
show databases like 'db_hive*';

--显示数据库信息
desc database db_hive;
--显示数据库详细信息
desc database extended db_hive;
--切换数据库
use db_hive;
--修改数据库
alter database db_hive set dbproperties('createtime'='20170830');
--删除数据库
drop database db_hive2;
--强制删除
drop database db_hive cascade;
```

## 创建表

```sql
CREATE [EXTERNAL] TABLE [IF NOT EXISTS] table_name 
[(col_name data_type [COMMENT col_comment], ...)] 
[COMMENT table_comment] 
[PARTITIONED BY (col_name data_type [COMMENT col_comment], ...)] 
[CLUSTERED BY (col_name, col_name, ...) 
[SORTED BY (col_name [ASC|DESC], ...)] INTO num_buckets BUCKETS ] 
[ROW FORMAT row_format] 
[STORED AS file_format] 
[LOCATION hdfs_path]
[TBLPROPERTIES (property_name=property_value, ...)]
[AS select_statement]
```

（1）CREATE TABLE 创建一个指定名字的表。如果相同名字的表已经存在，则抛出异常；用户可以用 IF NOT EXISTS 选项来忽略这个异常。

（2）EXTERNAL关键字可以让用户创建一个外部表，在建表的同时可以指定一个指向实际数据的路径（LOCATION），在删除表的时候，内部表的元数据和数据会被一起删除，而外部表只删除元数据，不删除数据。

```sql
--内部表外部表转换
alter table student2 set tblproperties('EXTERNAL'='FALSE');
```

（3）COMMENT：为表和列添加注释。

（4）PARTITIONED BY创建分区表

（5）CLUSTERED BY创建分桶表

（6）SORTED BY不常用，对桶中的一个或多个列另外排序

（7）ROW FORMAT 

```
DELIMITED [FIELDS TERMINATED BY char] 
		  [COLLECTION ITEMS TERMINATED BY char]
		  [MAP KEYS TERMINATED BY char] 
		  [LINES TERMINATED BY char] 
| SERDE serde_name [WITH SERDEPROPERTIES (property_name=property_value, property_name=property_value, ...)]
```

> 用户在建表的时候可以自定义SerDe或者使用自带的SerDe。如果没有指定ROW FORMAT 或者ROW FORMAT DELIMITED，将会使用自带的SerDe。在建表的时候，用户还需要为表指定列，用户在指定表的列的同时也会指定自定义的SerDe，Hive通过SerDe确定表的具体的列的数据。
>
> SerDe是Serialize/Deserilize的简称， hive使用Serde进行行对象的序列与反序列化。

[SerDE详细][1]

（8）STORED AS指定存储文件类型

> 常用的存储文件类型：SEQUENCEFILE（二进制序列文件）、TEXTFILE（文本）、RCFILE（列式存储格式文件）
>
> 如果文件数据是纯文本，可以使用STORED AS TEXTFILE。如果数据需要压缩，使用 STORED AS SEQUENCEFILE。

（9）LOCATION ：指定表在HDFS上的存储位置。

（10）AS：后跟查询语句，根据查询结果创建表。

（11）LIKE允许用户复制现有的表结构，但是不复制数据。

## 修改表

```sql
--查询表结构
desc dept_partition;

--修改表名
ALTER TABLE table_name RENAME TO new_table_name

--更新列
ALTER TABLE table_name CHANGE [COLUMN] col_old_name col_new_name column_type [COMMENT col_comment] [FIRST|AFTER column_name]

--增加、替换列
ALTER TABLE table_name ADD|REPLACE COLUMNS (col_name data_type [COMMENT col_comment], ...) 

--删除表
drop table dept_partition;
```

## 数据装载

```sql
load data [local] inpath '/opt/module/datas/student.txt' [overwrite] into table student [partition (partcol1=val1,…)];
```

（1）load data:表示加载数据

（2）local:表示从本地加载数据到hive表；否则从HDFS加载数据到hive表

（3）inpath:表示加载数据的路径

（4）overwrite:表示覆盖表中已有数据，否则表示追加

（5）into table:表示加载到哪张表

（6）student:表示具体的表

（7）partition:表示上传到指定分区

## 数据导出

```sql
insert overwrite directory '/user/atguigu/student2'
	ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t' 
    select * from student;
    
 export table default.student to
 '/user/hive/warehouse/export/student';
```

# 查询

```sql
SELECT [ALL | DISTINCT] select_expr, select_expr, ...
FROM table_reference
  [WHERE where_condition]
  [GROUP BY col_list]
  [HAVING having_condition]
  [ORDER BY col_list]
  [CLUSTER BY col_list | [DISTRIBUTE BY col_list] [SORT BY col_list] ]
  [LIMIT number]
```

# 常用函数

## NVL

NVL：给值为NULL的数据赋值，它的格式是NVL( value，default_value)

## CASE WHEN

```sql
CASE WHEN condition THEN result
	 WHEN condition THEN result
	 ELSE result
END
```

## 行转列

ONCAT(string A/col, string B/col…)：返回输入字符串连接后的结果，支持任意个输入字符串;

CONCAT_WS(separator, str1, str2,...)：它是一个特殊形式的 CONCAT()。第一个参数剩余参数间的分隔符。

COLLECT_SET(col)：函数只接受基本数据类型，它的主要作用是将某字段的值进行==去重汇总==，产生array类型字段。

COLLECT_LIST(col)：不去重

```sql
--用法
concat_ws('|', collect_set(t1.name)) name
```

## 列转行

EXPLODE(col)：将hive一列中复杂的array或者map结构拆分成多行。

LATERAL VIEW

用法：LATERAL VIEW udtf(expression) tableAlias AS columnAlias

解释：用于和split, explode等UDTF一起使用，它能够将一列数据拆成多行数据，在此基础上可以对拆分后的数据进行聚合。

```sql
select
    movie,
    category_name
from 
    movie_info 
    lateral view explode(category) table_tmp as category_name;
```

## 开窗函数

OVER()：指定分析函数工作的数据窗口大小，这个数据窗口大小可能会随着行的变而变化。

CURRENT ROW：当前行

n PRECEDING：往前n行数据

n FOLLOWING：往后n行数据

UNBOUNDED：起点，UNBOUNDED PRECEDING 表示从前面的起点， UNBOUNDED FOLLOWING表示到后面的终点

LAG(col,n,default_val)：往前第n行数据

LEAD(col,n, default_val)：往后第n行数据

NTILE(n)：把有序窗口的行分发到指定数据的组中，各个组有编号，编号从1开始，对于每一行，NTILE返回此行所属的组的编号。注意：n必须为int类型。

```sql
select 
sum(cost) over() as sample1,--所有行相加 
sum(cost) over(partition by name) as sample2,--按name分组，组内数据相加 
sum(cost) over(partition by name order by orderdate) as sample3,--按name分组，组内数据累加 
sum(cost) over(partition by name order by orderdate rows between UNBOUNDED PRECEDING and current row ) as sample4 ,--和sample3一样,由起点到当前行的聚合 
sum(cost) over(partition by name order by orderdate rows between 1 PRECEDING and current row) as sample5, --当前行和前面一行做聚合 
sum(cost) over(partition by name order by orderdate rows between 1 PRECEDING AND 1 FOLLOWING ) as sample6,--当前行和前边一行及后面一行 
sum(cost) over(partition by name order by orderdate rows between current row and UNBOUNDED FOLLOWING ) as sample7 --当前行及后面所有行 
from business;

select 
name,
orderdate,
cost, 
lag(orderdate,1,'1900-01-01') over(partition by name order by orderdate ) as time1, --前往第1行的数据
lag(orderdate,2) over (partition by name order by orderdate) as time2 --往前第2行的数据
from business;

select 
name, 
ntile(5) over(order by orderdate) sorted --把第5行分发到所有行
from business;
```

## RANK

RANK() 排序相同时会重复，总数不会变

DENSE_RANK() 排序相同时会重复，总数会减少

ROW_NUMBER() 会根据顺序计算

```sql
select 
name,
subject,
score,
rank() over(partition by subject order by score desc) rp,
dense_rank() over(partition by subject order by score desc) drp,
row_number() over(partition by subject order by score desc) rmp
from score;
```

# 自定义函数

```sql
show functions;--查看所有函数
desc function upper;--查看函数用法
desc function extended upper;--查看函数详细
```

* UDF（User-Defined-Function）

  一进一出

* UDAF（User-Defined Aggregation Function）
  聚集函数，多进一出
  类似于：count/max/min

* UDTF（User-Defined Table-Generating Functions）
  一进多出，如lateral view explore()

==编程步骤：==

1. 继承org.apache.hadoop.hive.ql.exec.UDF
2. 需要实现evaluate函数；evaluate函数支持重载；
3. 在hive的命令行窗口创建函数
   1. 添加jar
      add jar linux_jar_path
   2. 创建function
      create [temporary] function [dbname.]function_name AS class_name;
   3. 在hive的命令行窗口删除函数
      Drop [temporary] function [if exists] [dbname.]function_name;

==注意事项==

UDF必须要有返回类型，可以返回null，但是返回类型不能为void；

# 动态分区

> 不指定分区字段的值，但是必须放在查询的最后一个字段

（1）开启动态分区功能（默认true，开启）

==hive.exec.dynamic.partition=true==

（2）设置为非严格模式（动态分区的模式，默认strict，表示必须指定至少一个分区为静态分区，nonstrict模式表示允许所有的分区字段都可以使用动态分区。）

==hive.exec.dynamic.partition.mode=nonstrict==

（3）在所有执行MR的节点上，最大一共可以创建多少个动态分区。默认1000

hive.exec.max.dynamic.partitions=1000

（4）在每个执行MR的节点上，最大可以创建多少个动态分区。该参数需要根据实际的数据来设定。比如：源数据中包含了一年的数据，即day字段有365个值，那么该参数就需要设置成大于365，如果使用默认值100，则会报错。

hive.exec.max.dynamic.partitions.pernode=100

（5）整个MR Job中，最大可以创建多少个HDFS文件。默认100000

hive.exec.max.created.files=100000

（6）当有空分区生成时，是否抛出异常。一般不需要设置。默认false

hive.error.on.empty.partition=false

# hive支持中文

1.	修改hive-site.xml中的参数

```xml
<property>
	<name>javax.jdo.option.ConnectionURL</name>     
	<value>jdbc:mysql://hadoop102:3306/metastore?createDatabaseIfNotExist=true&amp;useUnicode=true&amp;characterEncoding=UTF-8</value>
	<description>JDBC connect string for a JDBC metastore</description>
</property>
```

2.	进入数据库 Metastore 中执行以下 SQL 语句 

```sql
（1）修改表字段注解和表注解：
alter table COLUMNS_V2 modify column COMMENT varchar(256) character set utf8；
alter table TABLE_PARAMS modify column PARAM_VALUE varchar(4000) character set utf8；

（2）修改分区字段注解：
alter table PARTITION_PARAMS modify column PARAM_VALUE varchar(4000) character set
utf8;
alter table PARTITION_KEYS modify column PKEY_COMMENT varchar(4000) character set utf8;
```

3.	测试：

```sql
hive (default)> create table student(id int COMMENT '学生ID', name string COMMENT '
学生姓名')
              > COMMENT '学生表'
              > PARTITIONED BY ( `date` string COMMENT '日期')
              > row format delimited
              > fields terminated by '\t';
desc formatted student;
查看是否能够正常显示中文
```











[1]:https://www.jianshu.com/p/9c43f03b97e7	"SerDe详解"

