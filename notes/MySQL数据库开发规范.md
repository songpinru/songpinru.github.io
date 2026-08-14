# MySQL数据库开发规范

## 前言

*   1、本规范定义了研发过程中数据库相关技术标准，研发过程中的数据库相关操作务必遵守。
    
*   2、本规范适配MySQL数据库，如在使用其他数据库不兼容相应规则时，参考对应数据库相同思路解决方案。
    

## 设计规范

*   1、数据库设计工具应使用PDManer。
    
*   2、数据库字符集应使用utf8mb4，排序字符集应使用utf8mb4\_0900\_ai\_ci。
    

> 统一字符集以避免乱码、乱序问题；MySQL中utf8编码只支持1-3个字节，utf8mb4支持1-4个字节，一些较复杂的汉字占4个字节，emoji符号占4个字节，所以统一使用utf8mb4；utf8mb4\_0900\_ai\_ci，0900对应Unicode 9.0的规范，ai表示不区分音调，ci表示不区分大小写；新建数据库实例时会指定以上默认字符集，创建库表时无需显示声明。

*   3、数据库对象名应使用小写字母，使用简拼或英文命名，禁用保留字，长度小于等于30个字符。
    

> 统一使用小写字母，避免出现需要大小写识别、转换问题，sql关键字同样使用小写字母；业务字段建议使用简拼，如：车辆编号--clbh、车辆别名--clbm；通用字段建议使用英文，如：姓名--name，状态--state；禁用如select、from、as、asc等保留字；长度小于等于30个字符，以兼容oracle 11g。

*   4、库名建议与应用名一致。
    
*   5、表设计。
    

> 原则：字段选择可以简单、正确存储数据的最小数据类型。

*   1、表命名建议为：业务名称\_表的作用。
    

> 如用户信息表命名为user\_info。

*   2、所有表必须设计主键。
    

> 为保障数据库性能、集群部署等需求，要求所有表必须有主键；主键命名为业务id，主子表关联时，子表逻辑外键与主表主键命名一致；MySQL库推荐使用有序主键。

*   3、业务表必备4个字段，create\_time、update\_time、create\_user、update\_user。
    

> 字段作为审计、增量设计用。

*   4、非负数整形应标识unsigned。
    
*   5、字符串长度几乎相等时建议使用char类型。
    
*   6、varchar类型字段长度建议不超过1000。
    
*   7、日期类型应使用date。
    
*   8、时间类型应使用datetime。
    
*   9、小数类型应使用decimal或bigint。
    

> float和double存在精度问题，decimal可以实现精确计算，但计算代价高；若数据量较大且频繁运算时使用bigint，如数据精确到百分之一元，则把数据乘以100存到bigint中。

*   10、表达是否概念的字段应使用unsigned tinyint，1代表是，0代表否，建议命名为is\_xx。
    
*   11、避免使用text、blob类型，如必须使用，建议独立到一张新表与主表关联。
    

> 大字段会降低表上dml操作的性能，增加io和带宽的消耗。

*   12、字段建议设置not null属性，按需定义默认值。
    

> 对于经常需要查询、计算的字段，建议设置not null属性，使用null值会占用额外存储空间，引起聚合偏差、降低查询性能等问题。

*   13、表、字段应设置注释。
    
*   14、禁止使用物理外键。
    

> 物理外键增加表间逻辑复杂度，降低数据库性能。

*   15、字段可适当冗余，以提高查询性能。
    

> 如出现经常需要大表join的查询，可适当冗余字段。

*   16、oltp系统单表字段数建议不超过80个。
    
*   17、表数据量过大时考虑分区、分表、或归档。
    

> 需综合硬件资源、参数配置、sql情况考虑，一般表大小超过内存1/2，索引大小超过内存1/8，可酌情操作。

*   18、应设计version表记录脚本版本信息。
    

*   6、索引设计。
    

> 原则：当索引可以将相关记录放到一起、降低扫描成本时创建索引，并尽可能利用索引的有序性和覆盖性。

*   1、索引命名建议为：索引前缀\_表名\_字段名。
    

> 主键前缀pk\_、唯一键前缀uk\_、普通索引前缀i\_；如在user\_info表username字段上创建索引，索引命名为i\_user\_info\_username；组合索引时使用”\_“追加字段名称，索引名称超过30个字符时，整合字段名称部分以组合含义命名。

*   2、业务上有唯一性的字段应创建唯一索引。
    
*   3、join关联字段应创建索引。
    
*   4、创建组合索引时，建议区分度最高的字段放在最左边。
    
*   5、禁止创建冗余索引。
    

> 如表中已存在索引key(a,b)，则key(a)为冗余索引。

*   6、单表索引数建议不超过7个。
    
*   7、利用索引有序性场景。
    

> 正列：where a=? and b=? order by c; 索引：a\_b\_c可以利用索引排序；反列：where a>? order by b; 索引：a\_b无法利用索引排序 。

*   8、利用索引覆盖性场景。
    

> 正例：select a, b, c from t where a=?; 索引：a\_b\_c，可利用覆盖性，不需要回表查询；反列：select a, b, c from t where a=?; 索引：a，无法利用覆盖性，需要回表查询。

**规范建表示例**

```plaintext
create table user_info (
    id int unsigned not null auto_increment comment '主键id',
    user_id mediumint unsigned not null default '0' comment '用户id',
    user_name varchar (30) not null default '' comment '姓名',
    birthday date not null default '0000-01-01' comment '生日',
    sex char (1) not null default '' comment '性别',
    user_review_status tinyint not null default '-1' comment '用户资料审核状态，1为通过，2为审核中，3为未通过，4为还未提交审核',
    user_register_ip int unsigned not null default '0' comment '用户注册时的源ip',
    short_introduce varchar (150) not null default '' comment '一句话介绍自己，最多150个汉字',
    is_valid tinyint unsigned not null default '1' comment '用户是否有效，1为有效，2为无效',
    create_time datetime not null default current_timestamp comment '创建时间',
    update_time datetime not null default current_timestamp on update current_timestamp comment '更新时间',
        primary key (id),
        unique key uk_user_id (user_id),
        key i_user_info_username (user_name),
        key i_user_info_create_time_user_review_status (create_time,user_review_status)
) comment = '用户基本信息';
```

## sql编写规范

*   1、禁止跨库查询。
    

> 为数据库迁移、分库、分表留出余地，降低业务耦合度，控制权限风险。

*   2、禁止select/insert \*，枚举所需字段。
    

> 降低cpu、io、网络资源消耗，避免无法使用覆盖索引，减少表结构变化带来的影响。insert \*指insert into t values(‘a’,’b’,’c’);。

*   3、使用count(\*)标准语法统计行数，禁止使用count(col)或count(1)等。
    
*   4、使用is null、is not null标准语法进行判空，禁止使用= null或!= null等。
    
*   5、如有全模糊或左模糊查询需求，建议通过搜索引擎实现。
    

> 当全模糊、左模糊查询需求性能要求较高，数据库无法满足时，建议通过搜索引擎实现。

*   6、禁止使用无条件或变相无条件的sql语句。
    

> 如where 1 = 1;属于变相无条件。

*   7、禁止程序中出现ddl语句。
    

> 为保障数据安全及数据库稳定，禁止程序中出现truncate table、drop table、create index等语句；对于需实现自动脚本控制的功能除外，但需严格检查脚本安全性。

*   8、禁止使用存储过程、自定义函数、触发器。
    

> 存储过程、自定义函数难以维护、扩展；触发器严重影响数据库性能；自定义函数使用在涉及不同数据库兼容时例外，原则上应避免使用非sql标准的数据库特有函数。

*   9、禁止使用hint语法。
    

> 随版本迭代及数据量变化，历史的hint语法可能不适合最新情况，出现性能问题。

*   10、禁止在where条件等号左边使用函数或表达式。
    

> 如where a-1 = ?，会导致索引key(a)失效，应实现为where a = ?+1。

*   11、禁止出现隐式转换。
    

> 字段类型不一致时会发生隐式转换，导致索引失效；常见隐式转换现象：不同表间相同字段，字段类型不一致，进行关联查询；字段类型为整形，查询条件传入字符串。

*   12、union使用，无去重需求时使用union all。
    

> union操作无论数据是否重复，都会进行排序去重操作，union all则不会。

*   13、join使用，关联字段要有索引，不建议在多个大表查询时进行join操作。
    
*   14、in操作尽量避免，如需要，建议控制in集元素在500个以内。
    

> in集过大是易导致索引失效，全表扫描。

*   15、insert into t values (),()...;建议控制插入值在500个以内。
    

> 提高sql效率，避免长事务引起复制延迟等问题。

*   16、分页时，若count为0，直接返回结果，避免执行后面的分页查询。
    
*   17、分页优化，分页超多场景时，建议利用延迟关联优化。
    

> limit m offset n实现时会先取m+n行记录，再舍弃掉m，保留n，当m很大时，效率很低；可以先快速定位需要获取的id，再进行关联，如：select a1 ,a2 from t order by a3 limit 1000000 offset 20;可以优化为：select t.a1 ,t.a2 from t inner join (select id from t order by a3 limit 1000000 offset 20) as tmp using(id);。

*   18、排序优化，order by、group by时考虑利用索引的有序性，避免文件排序。
    

> 利用索引有序性场景参考设计规范中6.7章节；当无法使用索引时，排序会使用临时表或文件排序，应始终避免使用文件排序。

*   19、避免使用子查询、not in、not exists，建议改写为join。
    

> 多数情况下，子查询、反连接性能低于join。
<!--stackedit_data:
eyJoaXN0b3J5IjpbNTUzMzU3OTIzXX0=
-->