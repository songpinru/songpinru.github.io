#  安装

1. 解压
2. 配置环境变量
3. 拷贝JDBC驱动到lib目录下

# Sqoop参数

## 导入到HDFS

查询导入：

```bash
/opt/module/sqoop/bin/sqoop import \
--connect \
--username \
--password \
--target-dir \
--delete-target-dir \
--num-mappers \
--fields-terminated-by   \
--query   "$2 and $CONDITIONS;"
```

 全部导入：

```bash
$ bin/sqoop import \
--connect jdbc:mysql://hadoop102:3306/company \
--username root \
--password 000000 \
--table staff \
--target-dir /user/company \
--delete-target-dir \
--num-mappers 1 \
--fields-terminated-by "\t"
```

指定导入：

```bash
$ bin/sqoop import \
--connect jdbc:mysql://hadoop102:3306/company \
--username root \
--password 000000 \
--target-dir /user/company \
--delete-target-dir \
--num-mappers 1 \
--fields-terminated-by "\t" \
--table staff
--columns id,sex \
--where "id=1"
```

## 导入到Hive

```bash
$ bin/sqoop import \
--connect jdbc:mysql://hadoop102:3306/company \
--username root \
--password 000000 \
--table staff \
--num-mappers 1 \
--hive-import \
--fields-terminated-by "\t" \
--hive-overwrite \
--hive-table staff_hive
```

## 导入到Hbase

```bash
$ bin/sqoop import \
--connect jdbc:mysql://hadoop102:3306/company \
--username root \
--password 000000 \
--table company \
--columns "id,name,sex" \
--column-family "info" \
--hbase-create-table \
--hbase-row-key "id" \
--hbase-table "hbase_company" \
--num-mappers 1 \
--split-by id
```

## 导出到MySQL

```bash
$ bin/sqoop export \
--connect jdbc:mysql://hadoop102:3306/company \
--username root \
--password 000000 \
--table staff \
--num-mappers 1 \
--export-dir /user/hive/warehouse/staff_hive \
--input-fields-terminated-by "\t"
```

## import参数

|      | **参数**                        | **说明**                                                     |
| ---- | ------------------------------- | ------------------------------------------------------------ |
| 1    | --enclosed-by <char>            | 给字段值前加上指定的字符                                     |
| 2    | --escaped-by <char>             | 对字段中的双引号加转义符                                     |
| 3    | --fields-terminated-by <char>   | 设定每个字段是以什么符号作为结束，默认为逗号                 |
| 4    | --lines-terminated-by <char>    | 设定每行记录之间的分隔符，默认是\n                           |
| 5    | --mysql-delimiters              | Mysql默认的分隔符设置，字段之间以逗号分隔，行之间以\n分隔，默认转义符是\，字段值以单引号包裹。 |
| 6    | --optionally-enclosed-by <char> | 给带有双引号或单引号的字段值前后加上指定字符。               |



|      | **参数**                          | **说明**                                                     |
| ---- | --------------------------------- | ------------------------------------------------------------ |
| 1    | --append                          | 将数据追加到HDFS中已经存在的DataSet中，如果使用该参数，sqoop会把数据先导入到临时文件目录，再合并。 |
| 2    | --as-avrodatafile                 | 将数据导入到一个Avro数据文件中                               |
| 3    | --as-sequencefile                 | 将数据导入到一个sequence文件中                               |
| 4    | --as-textfile                     | 将数据导入到一个普通文本文件中                               |
| 5    | --boundary-query <statement>      | 边界查询，导入的数据为该参数的值（一条sql语句）所执行的结果区间内的数据。 |
| 6    | --columns   <col1, col2, col3>    | 指定要导入的字段                                             |
| 7    | --direct                          | 直接导入模式，使用的是关系数据库自带的导入导出工具，以便加快导入导出过程。 |
| 8    | --direct-split-size               | 在使用上面direct直接导入的基础上，对导入的流按字节分块，即达到该阈值就产生一个新的文件 |
| 9    | --inline-lob-limit                | 设定大对象数据类型的最大值                                   |
| 10   | --m或–num-mappers                 | 启动N个map来并行导入数据，默认4个。                          |
| 11   | --query或--e <statement>          | 将查询结果的数据导入，使用时必须伴随参--target-dir，--hive-table，如果查询中有where条件，则条件后必须加上$CONDITIONS关键字 |
| 12   | --split-by   <column-name>        | 按照某一列来切分表的工作单元，不能与--autoreset-to-one-mapper连用（请参考官方文档） |
| 13   | --table   <table-name>            | 关系数据库的表名                                             |
| 14   | --target-dir   <dir>              | 指定HDFS路径                                                 |
| 15   | --warehouse-dir   <dir>           | 与14参数不能同时使用，导入数据到HDFS时指定的目录             |
| 16   | --where                           | 从关系数据库导入数据时的查询条件                             |
| 17   | --z或--compress                   | 允许压缩                                                     |
| 18   | --compression-codec               | 指定hadoop压缩编码类，默认为gzip(Use Hadoop codec default gzip) |
| 19   | --null-string   <null-string>     | string类型的列如果null，替换为指定字符串                     |
| 20   | --null-non-string   <null-string> | 非string类型的列如果null，替换为指定字符串                   |
| 21   | --check-column   <col>            | 作为增量导入判断的列名                                       |
| 22   | --incremental   <mode>            | mode：append或lastmodified                                   |
| 23   | --last-value   <value>            | 指定某一个值，用于标记增量导入的位置                         |

## export参数

| **序号** | **参数**                              | **说明**                                   |
| -------- | ------------------------------------- | ------------------------------------------ |
| 1        | --input-enclosed-by <char>            | 对字段值前后加上指定字符                   |
| 2        | --input-escaped-by <char>             | 对含有转移符的字段做转义处理               |
| 3        | --input-fields-terminated-by <char>   | 字段之间的分隔符                           |
| 4        | --input-lines-terminated-by <char>    | 行之间的分隔符                             |
| 5        | --input-optionally-enclosed-by <char> | 给带有双引号或单引号的字段前后加上指定字符 |

|      | **参数**                                | **说明**                                                     |
| ---- | --------------------------------------- | ------------------------------------------------------------ |
| 1    | --direct                                | 利用数据库自带的导入导出工具，以便于提高效率                 |
| 2    | --export-dir <dir>                      | 存放数据的HDFS的源目录                                       |
| 3    | -m或--num-mappers <n>                   | 启动N个map来并行导入数据，默认4个                            |
| 4    | --table <table-name>                    | 指定导出到哪个RDBMS中的表                                    |
| 5    | --update-key <col-name>                 | 对某一列的字段进行更新操作                                   |
| 6    | --update-mode <mode>                    | updateonly   allowinsert(默认)                               |
| 7    | --input-null-string <null-string>       | 请参考import该类似参数说明                                   |
| 8    | --input-null-non-string   <null-string> | 请参考import该类似参数说明                                   |
| 9    | --staging-table   <staging-table-name>  | 创建一张临时表，用于存放所有事务的结果，然后将所有事务结果一次性导入到目标表中，防止错误。 |
| 10   | --clear-staging-table                   | 如果第9个参数非空，则可以在导出操作执行前，清空临时事务结果表 |

## Hive参数

| **序号** | **参数**                        | **说明**                                                  |
| -------- | ------------------------------- | --------------------------------------------------------- |
| 1        | --hive-delims-replacement <arg> | 用自定义的字符串替换掉数据中的\r\n和\013 \010等字符       |
| 2        | --hive-drop-import-delims       | 在导入数据到hive时，去掉数据中的\r\n\013\010这样的字符    |
| 3        | --map-column-hive <arg>         | 生成hive表时，可以更改生成字段的数据类型                  |
| 4        | --hive-partition-key            | 创建分区，后面直接跟分区名，分区字段的默认类型为string    |
| 5        | --hive-partition-value <v>      | 导入数据时，指定某个分区的值                              |
| 6        | --hive-home <dir>               | hive的安装目录，可以通过该参数覆盖之前默认配置的目录      |
| 7        | --hive-import                   | 将数据从关系数据库中导入到hive表中                        |
| 8        | --hive-overwrite                | 覆盖掉在hive表中已经存在的数据                            |
| 9        | --create-hive-table             | 默认是false，即，如果目标表已经存在了，那么创建任务失败。 |
| 10       | --hive-table                    | 后面接要创建的hive表,默认使用MySQL的表名                  |
| 11       | --table                         | 指定关系数据库的表名                                      |

# 经验

###  Sqoop导入导出Null存储一致性问题

Hive中的Null在底层是以“\N”来存储，而MySQL中的Null在底层就是Null，为了保证数据两端的一致性。在导出数据时采用`--input-null-string`和`--input-null-non-string`两个参数。导入数据时采用`--null-string`和`--null-non-string`。

PS:`--null-non-string`为非string为空时的null值。

### Sqoop数据导出一致性问题

1）场景1：如Sqoop在导出到Mysql时，使用4个Map任务，过程中有2个任务失败，那此时MySQL中存储了另外两个Map任务导入的数据，此时老板正好看到了这个报表数据。而开发工程师发现任务失败后，会调试问题并最终将全部数据正确的导入MySQL，那后面老板再次看报表数据，发现本次看到的数据与之前的不一致，这在生产环境是不允许的。

官网：http://sqoop.apache.org/docs/1.4.6/SqoopUserGuide.html

–staging-table方式

先将数据写入到中间表，写入中间表成功，在一个transaction中将中间表的数据写入目标表

```bash
sqoop export \
--connect jdbc:mysql://192.168.137.10:3306/user_behavior \
--username root \
--password 123456 \
--table app_cource_study_report \
--columns watch_video_cnt,complete_video_cnt,dt \
--fields-terminated-by "\t" \
--export-dir "/user/hive/warehouse/tmp.db/app_cource_study_analysis_${day}" \
--staging-table app_cource_study_report_tmp \
--clear-staging-table \
--input-null-string '\N'
```



2）场景2：设置map数量为1个（不推荐，面试官想要的答案不只这个）

多个Map任务时，采用-`–staging-table`方式，仍然可以解决数据一致性问题。

### Sqoop底层运行的任务是什么

只有Map阶段，没有Reduce阶段的任务。

### Sqoop数据导出的时候一次执行多长时间

Sqoop任务5分钟-2个小时的都有。取决于数据量。

## 脚本

```bash
#! /bin/bash
sqoop=/opt/module/sqoop-1.4.6/bin/sqoop
if [[ -n "$2" ]] ;then
db_date=$2
else
db_date=`date -d '-1 day' +%F`
fi
import_data(){
$sqoop import \
--connect jdbc:mysql://hadoop102:3306/gmall \
--username root \
--password 123456 \
--target-dir /origin_data/gmall/db/$1/$db_date \
--delete-target-dir \
--num-mappers 1 \
--fields-terminated-by '\t' \
--query "$2 and \$CONDITIONS;"
}
import_sku_info(){
import_data 'sku_info' "select  
id, spu_id, price, sku_name, sku_desc, weight, tm_id,category3_id, create_time 
from sku_info where 1=1
"
}
import_user_info(){
  import_data "user_info" "select 
id, name, birthday, gender, email, user_level, 
create_time,operate_time
from user_info 
where (DATE_FORMAT(create_time,'%Y-%m-%d')='$db_date' or DATE_FORMAT(operate_time,'%Y-%m-%d')='$db_date')"
}
import_base_category1(){
  import_data "base_category1" "select 
id, name from base_category1 where 1=1"
}

import_base_category2(){
  import_data "base_category2" "select 
id, name, category1_id from base_category2 where 1=1"
}

import_base_category3(){
  import_data "base_category3" "select id, name, category2_id from base_category3 where 1=1"
}

import_order_detail(){
  import_data   "order_detail"   "select 
    od.id, 
    order_id, 
    user_id, 
    sku_id, 
    sku_name, 
    order_price, 
    sku_num, 
    od.create_time  
  from order_info o, order_detail od
  where o.id=od.order_id
  and DATE_FORMAT(od.create_time,'%Y-%m-%d')='$db_date'"
}

import_payment_info(){
  import_data "payment_info"   "select 
    id,  
    out_trade_no, 
    order_id, 
    user_id, 
    alipay_trade_no, 
    total_amount,  
    subject, 
    payment_type, 
    payment_time 
  from payment_info 
  where DATE_FORMAT(payment_time,'%Y-%m-%d')='$db_date'"
}

import_order_info(){
  import_data   "order_info"   "select 
    id, 
    total_amount, 
    order_status, 
    user_id, 
    payment_way, 
    out_trade_no, 
    create_time, 
    operate_time,
    province_id
  from order_info 
  where (DATE_FORMAT(create_time,'%Y-%m-%d')='$db_date' or DATE_FORMAT(operate_time,'%Y-%m-%d')='$db_date')"
}

import_base_province(){
  import_data base_province "select
    id,
    name,
    region_id,
    area_code
  from base_province
  where 1=1"
}

import_base_region(){
  import_data base_region "select
    id,
    region_name
  from base_region
  where 1=1"
}

import_base_trademark(){
  import_data base_trademark "select
    tm_id,
    tm_name
  from base_trademark
  where 1=1"
}

case $1 in
  "base_category1")
     import_base_category1
;;
  "base_category2")
     import_base_category2
;;
  "base_category3")
     import_base_category3
;;
  "order_info")
     import_order_info
;;
  "order_detail")
     import_order_detail
;;
  "sku_info")
     import_sku_info
;;
  "user_info")
     import_user_info
;;
  "payment_info")
     import_payment_info
;;
  "base_province")
     import_payment_info
;;

  "base_region")
     import_base_region
;;

  "base_trademark")
     import_base_trademark
;;

   "first")
   import_base_category1
   import_base_category2
   import_base_category3
   import_order_info
   import_order_detail
   import_sku_info
   import_user_info
   import_payment_info
   import_base_province
   import_base_region
   import_base_trademark
;;
   "all")
   import_base_category1
   import_base_category2
   import_base_category3
   import_order_info
   import_order_detail
   import_sku_info
   import_user_info
   import_payment_info
   import_base_trademark
;;
esac
```

