# 第一章 MySQL

## 1.1 数据库概述

（1）为什么要学习数据库

​	java是将数据存储在运行内存中，需要将数据进行持久化操作(运行内存中的数据存储到磁盘上)
​		a. IO流 -> 字节流、字符流、对象流 
​		b. 数据库：数据库就是存储数据的"仓库" (永久性的)

​	数据库的优点：

​		a. 统一管理易于查询

​		b. 共享性

（2）数据库的相关概念

​	**DB**:(Database) 数据库：存储数据的“仓库”，它保存了一系列有组织的数据

​	**DBMS**:(Database Managerment System)数据库管理系统:  数据库是通过DBMS创建和操作的容器

​	**SQL**:(Structure Query Language) 结构化查询语言: 专门用来和数据库通信的语言

（3）数据库的分类

​	关系型数据库: 采用表格的形式存储数据，表格和表格之间存在关联关系

​	非关系型数据库

## 1.2 初识MySQL

（1） MySQL产品介绍

- MySQL是关系型数据库的一种

- MySQL是一个小型关系型数据库管理系统，开发者为瑞典MySQL AB公司

- 在2008年1月16号被Sun公司收购。而2009年,SUN又被Oracle收购

- MySQL被广泛地应用在Internet上的中小型网站中。由于其体积小、速度快、总体拥有成本低

（2） MySQL的卸载

​	1.找服务：在开始菜单下输入services.msc 停止mysql服务 

​	2.在控制面板添加删除程序中卸载MySQL 

​	3.到安装目录删除MySQL 

​	4.删除数据文件 

​	5.查看注册表：(regedit命令) 
​		HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services 
​		HKEY_LOCAL_MACHINE\SYSTEM\ControlSet001\Services
​		HKEY_LOCAL_MACHINE\SYSTEM\ControlSet002\Services
​		搜索mysql，找到一律干掉！

（3） MySQL产品的安装(服务器端)	★

​	略

（4）MySQL服务的启动和停止	★ 

​		**通过工具**：搜索服务-->services.msc-->找到mysql服务-->右键停止和启动
​		**通过命令**:
​			打开cmd命令窗口
​				net stop 服务的名称
​				net start 服务的名称

（5） MySQL服务的登录和退出	★

​		登录需要采用用户名和密码
​			mysql -h ip地址 -P端口号 -uroot -p密码
​		退出：  exit

（6）MySQL的常见命令

​		show databases; 查看当前数据库管理系统中所有的数据库(会有四个默认的数据库)
​		create database 数据库名;    create database java0805;
​		use 数据库名;切换数据库
​		drop database 数据库名; 删除数据库
​		show tables; 查看当前数据库下所有的表格

（7）语法规范：
		a. MySQL中不区分大小写，关键字都大写，数据库名、表名、列名一般都小写 
		b. 每行sql语句都必须用分号结尾   并且如果sql语句太长  可以换行的
		c.注释
			单行注释
				-- 
				# 
			多行注释
				/* 

​				*/

## 1.3 SQL语言的分类

​	**DDL:  Data Definition Language** 数据定义语言   (和数据无关) 
​		库和表的管理
​		常见的约束
​	**DML: Data Manipulation Language** 数据操纵语言   ★
​		新增数据 insert  
​		修改数据 update
​		删除数据 delete
​		查询数据 select  **DQL: data query language** 数据查询语言 
​	**DCL：Data Control Language** 数据控制语言
​		权限 (MySQL没有) 
​		事务

## 1.4 DDL 数据定义语言

### 1.4.1 库的管理

```mysql
	#显示所有数据库
		show databases;
	#切换数据库
		use 数据库名;
	#新建数据库
		create database 【if not exists】 数据库名(自定义);
	#删除数据库
		drop database 【if exists】 数据库名;
```
### 1.4.2 表的管理

​	（1）mysql数据类型：
​			**整型：**
​				tinyint、smallint、mediumint、int、bigint
​				int【(3)】   默认是11位  
​			**浮点型：**
​			float、double
​			float/double【(n,m)】    
​				m:小数的位数     3
​				n:整数+小数的位数   5     	
​			**字符串：**
​				char【(n)】 (定长字符串)  n代表的是字符串的最大长度，如果省略默认是1
​				varchar(n)  (变长字符串)  n代表的是字符串的最大长度,设置长度不能省略
​					char【(5)】
​					varchar(10)  
​			**日期类型：**
​				year    只能存储年份
​				date    只能存储年月日
​				datetime   日期+时间		1900-1-1~9999-12-31    	不受时区和服务器版本的影响
​				timestamp  日期+时间  时间戳   1970-1-1~2038-12-31		受时区和服务器版本的影响
​			**大字节：** blob	
​			**大文本：** text

​	（2）约束条件  六大约束

​		a. **非空**    not null    create table test1(id int,name varchar(10) not null);

​		b. **唯一 **   unique       create table test2(id int unique,name varchar(10) unique);

​		c. **主键**(非空+唯一)   primary key  一般设置在编号这一列  create table test4(id int primary key,age int);

​		d. **默认值 **  default 值    create table test5(id int,name varchar(10) default 'jack');

​		e. **检查约束**   check  (MySQL不支持)

​		f. **外键约束**   foreign key    涉及到多表 

​	（3）机制：自增长机制

​		auto_increment    必须使用在主键列、使用方式和约束条件一致

​		create table test4(id int primary key auto_increment,age int);

```mysql
	#显示所有表格
		show tables;
	#新建表格(表名、有哪些列：列的个数和名称、列的数据类型、【约束条件】) 
		#语法：create table 表名(列名1 1列的数据类型 【1列的约束条件】,列名2 2列的数据类型 【2列的约束条件】...);
	#查看表结构(表的列的信息)
		desc 表名;
	#删除表格
		drop table 表名;
	#表格结构的修改 (改名、新增一列、删除一列、修改列名、修改列的数据类型)
		#语法： alter table 表名 后续操作;
					
		#a. 修改表名
			alter table emp rename employee;
		#b. 新增一列(必须有数据类型，可以有约束条件)
			alter table employee add name varchar(10) not null;
		#c. 修改列名和数据类型
			alter table employee change gender sex char;
		#d. 修改列的数据类型    (新数据类型要符合原来数据的类型)
			alter table employee modify sex varchar(10);
		#e. 删除一列
			alter table employee drop name;
```

## 1.5 DML 数据操纵语言

### 1.5.1 新增数据  insert into

**语法1**：insert into 表名(指定列名...) values(值...);   指定部分列的数据

​	INSERT INTO stu(stu_name,stu_age) values('周杰伦',20);
​	INSERT INTO stu(stu_age,stu_address,stu_name) values(25,'北京','张学友');

**语法2**：insert into 表名 values(值...);    添加全部列数据(值的顺序按照表结构的字段顺序)

​	INSERT INTO stu(stu_id,stu_name,stu_age,stu_address) VALUES(NULL,'aaa',20,'上海');
​	INSERT INTO stu VALUES(NULL,'bbb',20,'深圳');

**语法3**： insert into 表名(部分列...) values(值...),(值...),(值...)...

​	INSERT INTO stu(stu_name,stu_age) values('华仔',30),('郭富城',25),('蔡徐坤',28);

**语法4**： insert into 表名 values(值...),(值...),(值...)...

​	INSERT INTO stu VALUES(NULL,'ccc',20,'深圳'),(NULL,'ddd',20,'深圳');

```mysql
CREATE TABLE books(
	b_id INT PRIMARY KEY,
	b_name VARCHAR(50) NOT NULL,
	authers VARCHAR(100) NOT NULL,
	price FLOAT NOT NULL,
	pubdate YEAR,
	note VARCHAR(100),
	num int
);
INSERT INTO books(b_id,b_name,authers,price,pubdate,note,num) 
	VALUES(1,'Tal of AAA','Dickes',23,1995,'novel',11);
INSERT INTO books 
	VALUES(2,'EmmaT','Jane lura',35,1993,'Joke',22);
INSERT INTO books 
	VALUES(2,'EmmaT','Jane lura',35,1993,'Joke',22)
	,(2,'EmmaT','Jane lura',35,1993,'Joke',22)
	,(2,'EmmaT','Jane lura',35,1993,'Joke',22);
```

### 1.5.2 修改数据 update

**语法**：update 表名 set 需要修改的列名=新值,需要修改的列名=新值,... 【where 修改条件】

（1）修改表格中所有数据

```mysql
# 将stu表格中的所有学生的地址修改为中国
	UPDATE stu SET stu_address='中国'
# 将stu表格中的所有学生的地址修改为中国北京，并且将所有人的年龄增加1岁
	UPDATE stu SET stu_address='中国北京',stu_age=stu_age+1;
```

（2）添加修改条件

**关系运算符**    > >= < <= = != <>  in()

```mysql
# 将周杰伦的地址修改为中国台湾
	UPDATE stu SET stu_address='中国台湾' WHERE stu_name='周杰伦'
# 将年龄大于25岁的学生，薪资设置为20000并且手机号码设置为666
	UPDATE stu SET stu_salary=20000,stu_phone='666' WHERE stu_age>25;
# 将不是台湾人的薪资设置为26000
	UPDATE stu SET stu_salary=26000 WHERE stu_address<>'中国台湾'
```

**逻辑运算符**   并且 and   或者 or   非  not

```mysql
# 将年龄是小于25岁并且地址是中国北京的学生手机号码设置为888
	UPDATE stu SET stu_phone='888' WHERE stu_age<25 AND stu_address='中国北京'
# 将不是中国北京的学生薪资设置为2000
	UPDATE stu SET stu_salary=2000 WHERE not stu_address='中国北京'
# 将薪资为空的学生薪资设置为25000
	UPDATE stu SET stu_salary=25000 WHERE stu_salary IS NULL
# 将手机号码不为空的学生薪资设置为35000
	UPDATE stu SET stu_salary=35000 WHERE stu_phone IS NOT NULL
	UPDATE stu SET stu_salary=45000 WHERE NOT stu_phone IS  NULL
```

### 1.5.3 删除数据 delete

**语法**： delete from 表名 【where 删除条件】    

（1）删除表格中全部数据

```mysql
# 将class表中数据全部删除
	DELETE FROM class;-- 一条一条的删
# 清空表  TRUNCATE TABLE 表名;
	TRUNCATE TABLE class;
```

| 方式     | 效率 | 自增长机制     | 是否支持数据回滚 | 删除条件 |
| -------- | ---- | -------------- | ---------------- | -------- |
| DELETE   | 较低 | 不会破坏原来的 | 支持             | 允许     |
| TRUNCATE | 较高 | 会破坏原来的   | 不支持           | 不允许   |

（2）添加删除条件

​	**删除条件的语法和修改条件的语法一致**

```mysql
# 将class表中id大于3的删除
	DELETE FROM class WHERE id>3;
```

## 1.6 DQL 数据查询语言

### 1.6.1 简单查询

**语法**： select 常量、字段名、函数、表达式以及上述组合形式 from 表名 where 查询条件; 

（1）查询所有列信息

```mysql
# 查询所有学生的所有信息
	SELECT stu_id,stu_name,stu_age,stu_address,stu_phone,stu_salary FROM stu;
# 简写  *代表所有列，顺序是按照表结构的顺序
	SELECT * FROM stu;
```

（2）查询部分列数据

```mysql
# 查询所有学生的姓名和地址
	SELECT stu_name,stu_address,stu_age FROM stu;
# 查询年龄大于25岁的学生姓名和工资
	SELECT stu_name,stu_salary,stu_age FROM stu WHERE stu_age>25
```

（3）添加查询条件

​	**查询条件和修改删除条件写法都一样**

```mysql
# 查询年龄大于25 并且小于30的学生信息
SELECT * FROM stu WHERE stu_age>25 and stu_age<30; 
SELECT * FROM stu WHERE stu_age BETWEEN 25 AND 30;
```

（4）模糊查询

**语法**：字段 like 值

​	%   0-n个字符

​	_    1个字符

```mysql
# 查询出名字中包含张字的学生信息    模糊查询   like
	SELECT * FROM stu WHERE stu_name like '%张%'
# 查询出姓张的学生信息
	SELECT * FROM stu WHERE stu_name like '张%'
# 查询出姓张并且全名为2个字的学生信息
	SELECT * FROM stu WHERE stu_name like '张__'
# 查询名字中包含j的
	SELECT * FROM stu WHERE BINARY stu_name like '%J%';  -- 英文没有区分大小写  添加BINARY区分
```

（5）排序

**语法**： ORDER BY 需要排序的字段名 排序规则;       Ps: 不需要where关键字

```mysql
# 查询所有学生的数据，按照工资进行排序   默认是升序(ASC)   降序(desc)
	SELECT * FROM stu ORDER BY stu_salary DESC;
# 将工资不为空的所有学生信息查出，并按照工资进行排序
	SELECT * FROM stu WHERE stu_salary IS NOT NULL ORDER BY stu_salary DESC
# 将工资不为空的所有学生信息查出，并按照工资进行排序,如果工资相同按照年龄排序
	SELECT * FROM stu WHERE stu_salary IS NOT NULL ORDER BY stu_salary DESC,stu_age DESC
```

（6）表达式

```mysql
# 查询出所有学生的姓名和学生的年薪
	SELECT stu_name,stu_salary,stu_salary*12 FROM stu;
```

（7）别名

**语法**： 字段名、表达式、表名 as 别名       as是可以省略的

```mysql
# 起别名
SELECT stu_name '姓名',stu_salary,stu_salary*12 年薪 FROM stu;
# 查询出所有学生的姓名和学生的年薪，并且按照年薪排序 (用到了别名排序)
SELECT stu_name,stu_salary,stu_salary*12 sum FROM stu ORDER BY sum;
```

### 1.6.2 函数

（1）单行函数

```mysql
#A:字符相关
	SELECT LOWER('Abc') #将数据全部转为小写
	SELECT UPPER('Abc') #将数据全部转为大写
	#将books中的数据全部查出，并且将书名设置为全大写的形式
	SELECT b_id,UPPER(b_name),price,LOWER(note),num FROM books;
	SELECT CONCAT('java','mysql','jdbc') #将数据拼接在一起
	#将books中的数据全部查出,将书名和作者名拼接到一起
	SELECT b_id,CONCAT(b_name,authers),price,note,num FROM books;
	SELECT SUBSTR('helloworld',3)  #从第三个字符截取到末尾
	SELECT SUBSTR('helloworld',3,6) #从第三个字符截取,总共截取6个
	SELECT LENGTH('helloworld你好')  #返回的是字节数
	SELECT INSTR('helloworld你好','o') #返回第一次出现指定字符的位置
	SELECT LPAD('java',7,'a')
	SELECT RPAD('java',7,'ab')
	SELECT TRIM('    hello   world    ') #去除字符串的前后空格
	SELECT TRIM('a' from 'aahelloaaworldaa')#去除字符串的前后指定字符
	SELECT REPLACE('helloworld','o','j')#替换内容
#B:和数学相关
	SELECT FLOOR(-1.3)#向下取整
	SELECT CEIL(1.4)#向上取整
	select ROUND(1.5)#四舍五入
	SELECT TRUNCATE(12.3456,2)#截断
	SELECT RAND();#产生随机数
	SELECT MOD(10,6)  #求余数
#C:日期函数
	1. STR_TO_DATE(str,format) #str->date
	
		INSERT INTO stu VALUES(17,'周星驰',20,'香港','8888888',30000,STR_TO_DATE('1977年3月3日 13:34:56','%Y年%m月%d日 %H:%i:%s'));
		INSERT INTO stu VALUES(18,'周星驰',20,'香港','8888888',30000,STR_TO_DATE('5-8 1987','%m-%d %Y'));
	2. DATE_FORMAT(date,format) #date->format
		#查询时就显示成 xxxx年xx月xx日
		SELECT DATE_FORMAT(stu_birthday,'%Y年%m月%d日') FROM stu
	3. YEAR(date)  # 单独从生日中取出年份
	   MONTH(date) 
	   DAY(date) 
	   HOUR(date) 
	   MINUTE(date) 
	   SECOND(date) 
	   #案例：
		SELECT *,YEAR(stu_birthday),MONTH(stu_birthday),DAY(stu_birthday),
		HOUR(stu_birthday),MINUTE(stu_birthday),SECOND(stu_birthday)
 		FROM stu WHERE stu_id>14
	4. SELECT NOW() #获取当前系统的日期+时间
	   SELECT CURDATE()#获取当前系统的日期
	   SELECT CURTIME()#获取当前系统的时间
```

（2）组合函数

​	①count(列名/*) 数据的总条数    ps:null值不算总条数
​	②max(列名);  最大值
​	③min(列名);  最小值
​	④sum(列名);  总和
​	⑤avg(列名);  平均值

```mysql
# 查询一共有多少本不重名的书
	SELECT count(*) FROM books WHERE pubdate>2000;
# 求出库存最多的书的数量
	SELECT min(num) FROM books;
# 求图书馆中所有书的数量
	SELECT sum(num) FROM books;
# 求所有图书价格的平均值
	SELECT avg(price) avg,sum(num) FROM books;
```

（3）去重

```mysql
#查询出图书有多少种类   
#去重 DISTINCT 将相同的数据删除
	SELECT DISTINCT note FROM books

```

（4）分组查询  **将相同的数据自动归为一组**

**语法**：GROUP BY 需要分组的字段名;

```mysql
#查询出图书有多少种类
	SELECT note FROM books GROUP BY note
# 查询出每种图书种类的图书数    分组+组合函数
	# 组合函数在没有分组的sql语句中，对整张表格起作用
	# 组合函数在有分组的sql语句中，是对一个小组起作用
	SELECT note,sum(num) FROM books GROUP BY note
```

（5）HAVING

​	**功能**：在分组+组合函数之后添加筛选条件

```mysql
# 查询出价格在40以下每种图书种类的图书数  只显示数据大于0的
	SELECT note,sum(num) FROM books
	WHERE price<40 GROUP BY note HAVING sum(num)>0
```

### 1.6.3 复杂查询

（1）分支结构

​	A: 双分支

​		**语法**：if(条件,值1,值2) 条件成立返回值1，条件不成立返回值2

​		SELECT IF(5<3,1,2)

```mysql
#查询所有的图书，如果库存低于20本，价格显示为原来的2倍
SELECT b_id,b_name,IF(num<20,price*2,price),num from books;
```

​	B: 多分支

​		**语法**：SELECT
​				CASE
​				WHEN 条件1        THEN 值1
​				WHEN 条件2	THEN 值2
​				WHEN 条件3	THEN 值3

​				ELSE 其余情况

​				END

```mysql
# 查询所有的图书，如果库存为0本，价格显示为原来的10倍，如果库存低于20本，价格显示为原来的2倍
SELECT b_id,b_name,
	CASE
		WHEN num=0 THEN price*10
		WHEN num<20 then price*2
		WHEN num>=20 THEN price
	END
,num FROM books;
```

​	C: Switch结构

​		**语法**：SELECT
​				CASE 值
​				WHEN 1 THEN 10
​				WHEN 2 THEN 20
​				WHEN 3 THEN 30
​				END

（2）多表联查  (需要得到的结果或者你的查询条件来自于多张表格)

​	A: sql92语法 

```mysql
-- 1. 内连接  (只能查询出具有关联关系的数据)
 等值连接  多张表格中具有等值的条件
	# 查询出刘德华的全部信息(包括部门信息)
	SELECT * FROM emp,dept WHERE emp_name='刘德华';  
	# 笛卡尔积现象
	# 原因：就是没有添加关联关系
	# 如何解决：添加关联关系    
	# 多表查询如果出现重名，需要采用表名.字段名的形式去区分
	SELECT * FROM emp,dept WHERE emp_name='郭富城' AND emp.dept_id=dept.dept_id;  
	# 查询所有员工的信息(包括部门信息)
	SELECT * FROM emp,dept WHERE emp.dept_id=dept.dept_id;
	#三表联查
	#查询王宝强都选了哪些课程
	SELECT * FROM student,sc,course WHERE student.stu_name='张三' 
	AND sc.stu_id=student.stu_id AND sc.cou_id=course.course_id
	#心理学这门课被哪些学生选了
	SELECT * FROM student,sc,course WHERE course.course_name='心理学' 
	AND sc.stu_id=student.stu_id AND sc.cou_id=course.course_id
 非等值连接
	#查询出员工的信息(包括他的工资级别)
	SELECT * FROM emp,gr WHERE emp.emp_salary>=gr.min and emp.emp_salary<=gr.max
 自连接  
	# 查询出北京市下有哪些区
	SELECT * FROM city c1,city c2 WHERE c1.city_name='上海市' AND c1.city_id=c2.pid
	# 查询昌平区在那个市
	SELECT * FROM city c1,city c2 WHERE c1.city_name='宝安区' AND c1.pid=c2.city_id
外链接 (mysql不支持)
	左外连接
	右外连接
	全外连接
```

​	B: sql99语法

```mysql
-- 内连接
 等值连接 语法：表1 inner join 表2 on 管理关系 【where 查询条件】
	#查询出刘德华的全部信息(包括部门信息)
	SELECT * FROM emp JOIN dept ON emp.dept_id=dept.dept_id WHERE emp.emp_name='刘德华'
 非等值连接
	#查询出员工的信息(包括他的工资级别)
	SELECT * FROM emp JOIN gr ON emp.emp_salary>=min AND emp.emp_salary<=max
 自连接
	# 查询出北京市下有哪些区
	SELECT * FROM city c1,city c2 WHERE c1.city_name='上海市' AND c1.city_id=c2.pid
	SELECT * FROM city c1 JOIN city c2 on c1.city_id=c2.pid WHERE c1.city_name='上海市'
	#查询王宝强都选了哪些课程
	SELECT * FROM student join sc join course 
	ON sc.stu_id=student.stu_id AND sc.cou_id=course.course_id
	WHERE student.stu_name='王宝强' 
-- 外链接
 左外连接  # LEFT JOIN 将左侧的表信息全部显示，右侧的如果没有对应信息用null补齐
	#查询所有员工的信息(包括部门信息)
	SELECT * FROM emp LEFT JOIN dept ON emp.dept_id=dept.dept_id
	SELECT * FROM dept LEFT JOIN emp ON emp.dept_id=dept.dept_id
 右外连接 # RIGHT JOIN 将右侧的表信息全部显示，右侧的如果没有对应信息用null补齐
	SELECT * FROM emp RIGHT JOIN dept ON emp.dept_id=dept.dept_id
 全外连接 (不支持) 
```

（2）子查询(sql语句的嵌套(一个sql语句中存在另一个完整的sql语句))   

```mysql
# 查询出刘德华所在部门的员工信息
	#1. 先查询出刘德华所在的部门id
	SELECT dept_id FROM emp WHERE emp_name='刘德华'
	#2. 查询出员工信息
	SELECT * FROM emp WHERE dept_id=(SELECT dept_id FROM emp WHERE emp_name='刘德华')
# 查询出刘德华、王思聪所在部门的员工信息
	SELECT dept_id FROM emp WHERE emp_name='刘德华' OR emp_name='王思聪'
	SELECT * FROM emp WHERE dept_id in(
		SELECT dept_id FROM emp WHERE emp_name='刘德华' OR emp_name='王思聪'
	)
```

  **注意事项**：
	1. 子sql语句的结果必须是一列
	2. 等号只能等于一个结果：必须是一行
	3. 如果子sql返回多行结果，需要用in(结果),去实现功能

（3）分页

​	a.  limit n; 获取筛选后结果的前n个

​		SELECT * FROM stu WHERE stu_id!=1 LIMIT 5;

​	b.  limit n,m; 从n下标开始取，取m个  (mysql中需要从0开始的下标)

​		SELECT * FROM stu LIMIT 5,5; 

**总结：**

```mysql
SELECT 字段、表达式、函数、常量以上的组合型
FROM 表名
JOIN 表名 ON 关联条件
WHERE 查询条件
GROUP BY 分组
HAVING 分组后查询条件
ORDER BY 排序
LIMIT 分页
```

## 1.7 事务

**事务是由一个或者多个DML语句组成**

事务的特性 ACID特性：

​	**原子性(Atomicity)：**原子意为最小的粒子，或者说不能再分的事物。数据库事务的不可再分的原则即为原子		性。 组成事务的所有SQL必须：要么全部执行，要么全部取消

​	**一致性(Consistency)**：指数据的规则,在事务前/后应保持一致
​	**隔离性(Isolation)**：简单点说，某个事务的操作对其他事务不可见的.  Spring框架
​	**持久性(Durability)**：当事务提交完成后，其影响应该保留下来，不能撤消

实现步骤：

```mysql
#将mysql的自动提交关闭
set autocommit=false;
# 执行DML操作
UPDATE score SET grade=grade-10 WHERE id=1;
UPDATE score SET grade=gade+10 WHERE id=2;
# 提交
# COMMIT;
# 回滚
ROLLBACK;
```

# 第二章 JDBC

## 2.1 什么是JDBC?

​	**JDBC** (Java Database Connectivity) 独立于特定数据库管理系统、通用的SQL数据库存取和操作的公共接口
​	一套接口(一套标准)  SUN公司提出的一套标准   各大数据库厂商去实现标准

## 2.2 简单的JDBC案例

### 2.2.1 导入数据库驱动包

 		①在项目下建一个文件夹
 		②将数据库驱动包粘贴到文件夹中
 		③在驱动包上右键->build path->Add to build path

### 2.2.2 JDBC实现步骤

 		① 注册驱动  (加载Driver)
 			A: DriverManager.registerDriver(new Driver());

​			 B: 在查看Driver源码时发现，存在一个静态代码块,存在注册驱动的代码	

​				Class.forName("com.mysql.jdbc.Driver");

 		② 获得数据库连接(根据驱动管理器)

​			A: 自己去创建，效率比较低

 			Connectionconn= DriverManager.getConnection(String url, String username, String password);

​			B: 采用数据库连接池

```java
				//a. 创建数据库连接池的模板类
 					DruidDataSource ds=new DruidDataSource();
 				//b. 为数据库连接池提供需要连接数据库的连接信息 (dirver,url,username,password)
 					ds.setDriverClassName("com.mysql.jdbc.Driver");
					ds.setUrl("jdbc:mysql://127.0.0.1:3306/school");
					ds.setUsername("root");
					ds.setPassword("123");
					//数据库连接池支持个人定制
					ds.setInitialSize(10);//设置初始化数据库连接的个数
					ds.setMaxActive(20);//设置最大活跃数为20
					ds.setMinIdle(5);//设置最小空闲数为5
				//c. 从数据库连接池中获取数据库连接
					conn=ds.getConnection();
				//d. 释放资源
					conn.close();// 数据库连接由关闭转为释放(将数据库连接恢复到最初始状态，在供后续使用)
```

 		③ 获得通道对象

​			A: 获得状态通道对象
 				Statement sta = conn.createStatement();

​			问题：a. 拼接sql语句语法要求很严格
​			  	 b. sql注入的问题(从外界获取过来的数据存在特殊符号，或者存在特殊语句)
​			  	 c. 效率   较低

​			B: 获得预状态通道对象

​				PreparedStatement psta = conn.prepareStatement(String sql);

​				会在创建通道时需要参数sql语句(预编译的sql语句:需要通过占位符？进行占位)
​				String sql="insert into student values(null,?,?)";

 		④ 操作
 			增删改     **返回值是int,影响的行数**
 				statement .executeUpdate(String sql) 

​				 preparedStatement.executeUpdate()

```java
@Test
	public void update() {
		Connection conn =null;
		PreparedStatement psta =null;
		try {
			//1. 注册驱动
			DriverManager.registerDriver(new Driver());
			//2. 通过驱动管理器获得数据库连接
			conn= DriverManager.getConnection("jdbc:mysql://127.0.0.1:3306/school", "root", "123");
			System.out.println(conn);
			//3. 操作 (增删改查)
//			获得操作的对象 (通道)
			psta = conn.prepareStatement("insert into student values(null,?,?)");
			//新增步骤    为占位符设置值
			psta.setString(1, name);
			psta.setString(2, age);
            //执行
			int num=psta.executeUpdate();
			System.out.println(num);
			
		} catch (Exception e) {
			e.printStackTrace();
		}finally{
			//4. 关闭资源
			try {
				conn.close();
				psta.close();
			} catch (SQLException e) {
				e.printStackTrace();
			}
		}
```

查询   **查询方法返回的是ResultSet(结果集)   rs中封装的就是sql语句查询出来的结果集**

> statement .executeQuery(String sql)  
>  preparedStatement.executeQuery()
> ResultSet
> 		next()  判断下一个是否有数据，并且将rs指向下一个数据
> 		getXXX(int index) 获取第几列的数据
> 		getXXX(String name) 根据结果集的列名去获取数据
> Ps: 所有的数据类型都可以通过getString和getObject去获取(取过来之后可能需要类型转化)

```java
@Test
	public void query() {
		Connection conn =null;
		PreparedStatement psta =null;
		ResultSet rs=null;
		try {
			//1. 注册驱动
			DriverManager.registerDriver(new Driver());
			//2. 通过驱动管理器获得数据库连接
			conn= DriverManager.getConnection("jdbc:mysql://127.0.0.1:3306/school", "root", "123");
			//3. 操作 (增删改查)
//			获得操作的对象 (通道)
			psta = conn.prepareStatement("select * from student where stu_id=?");
            //绑定参数
            psta.setInt(1,1);
//			查询操作
			//a. 返回一条数据     一行n列       对象
			//b. 返回多条数据     n行n列        集合
			//c. 返回一个数据    一行一列	      变量
			//查询方法返回的是ResultSet(结果集)   rs中封装的就是sql语句查询出来的结果集
			rs = psta.executeQuery();
			//将rs中的数据取出，构建一个Student对象
			while(rs.next()){
//				int int1 = rs.getInt(1);//根据第几列取值
				int int1 = rs.getInt("id");//根据结果集中的列名取值
//				System.out.println("id:"+int1);
				String name = rs.getString(2);
//				String name=rs.getString("stu_name");
//				System.out.println("name:"+name);
				int age = rs.getInt(3);
//				System.out.println("age:"+age);
				Student stu=new Student(int1, name, age);
				System.out.println(stu);
			}
		} catch (Exception e) {
			e.printStackTrace();
		}finally{
			//4. 关闭资源
			try {
				conn.close();
				psta.close();
				rs.close();
			} catch (SQLException e) {
				e.printStackTrace();
			}
		}
	}
```

 		⑤ 关闭资源
 			close();

## 2.3 JDBC的相关API

（1）java.sql.DriverManager
		registerDriver(Driver driver)注册驱动
 		getConnection(String url,String username,String passwrod)获得数据库连接

（2）java.sql.Connection
 		createStatement();获得状态通道对象
 		prepareStatement(String sql)获得预状态通对象
 		close()  关闭资源

（3）java.sql.Statement
 		executeUpdate(String sql)增删改的操作
 		executeQuery(String sql)查询的操作
 		close();

（4）PreparedStatement
 		setXXX(int index,XXX x) 绑定参数
 		executeUpdate()增删改的操作
 		executeQuery()查询的操作
 		close()

（5）ResultSet 结果集对象
 		next() 判断下一条是否有值，并且指向下一条数据
 		getXXX(int index) 根据第几列取值
 		getXXX(String name) 根据结果集的列名取值
 		close()

## 2.4 数据库连接池

**功能**：创建和维护数据库连接

**产品**：德鲁伊数据库连接池(阿里巴巴)

**实现步骤**：

```java
	//让数据库连接池去创建    需要创建一个数据库连接池的模板类
	DruidDataSource dds=new DruidDataSource();
	dds.setDriverClassName("com.mysql.jdbc.Driver");
	dds.setUrl("jdbc:mysql://127.0.0.1:3306/bigdata1021");
	dds.setUsername("root");
	dds.setPassword("root");
	//从数据库连接池中获得
	conn=dds.getConnection();
	//通过数据库连接池获得的conn,该对象的close方法时释放该对象(将该对象洗脑后在重新放入连接池中)
	conn.close();
```

## 2.5 数据库连接信息提取到配置文件

（1）配置文件信息
```
driver=com.mysql.jdbc.Driver
url=jdbc:mysql://127.0.0.1:3306/bigdata1021
username=root
password=root
```
（2）采用Properties读取

```java
Properties pro=new Properties();
pro.load(new FileInputStream("src\\db.properties"));
dds.setDriverClassName(pro.getProperty("driver"));
dds.setUrl(pro.getProperty("url"));
dds.setUsername(pro.getProperty("username"));
dds.setPassword(pro.getProperty("password"));
```

## 2.6 预命令对象

* 编写sql，并用？做占位符
* 调用connection的prepareStatement()方法
* 绑定参数
* 执行

```java
//1.预编译的sql语句   ?:占位符
String sql="insert into user values(?,?,?)";
//2. 获取预命令对象
PreparedStatement psta = conn.prepareStatement(sql);
//3. 绑定参数(设置占位符位置的值)
psta.setString(1, "5");
psta.setString(2, input.next());
psta.setString(3, "123");
//4. 执行
int i = psta.executeUpdate();
```

## 2.7 JDBCUtils

* 初始化连接池
* 提供getConnection
* close() 关闭资源

```java
public class JDBCUtils {
	
//	private static DruidDataSource dataSource=null;
	private static DataSource dataSource=null;
	static{
		//实现数据库连接池
//		dataSource=new DruidDataSource();
        
//		dataSource.setDriverClassName("com.mysql.jdbc.Driver");
//		dataSource.setUrl("jdbc:mysql://127.0.0.1:3306/bigdata1021");
//		dataSource.setUsername("root");
//		dataSource.setPassword("root");
        
		Properties pro=new Properties();
		try {
			pro.load(new FileInputStream("src\\com\\atguigu\\config\\db.properties"));
			dataSource=DruidDataSourceFactory.createDataSource(pro);
		} catch (Exception e) {
			e.printStackTrace();
		} 
		
//		dataSource.setDriverClassName(pro.getProperty("abc"));
//		dataSource.setUrl(pro.getProperty("url"));
//		dataSource.setUsername(pro.getProperty("username"));
//		dataSource.setPassword(pro.getProperty("password"));
		
	}
	/**
	 * 功能：获得数据库连接
	 * @return
	 */
	public static Connection getConnection(){
		try {
			return dataSource.getConnection();
		} catch (SQLException e) {
			e.printStackTrace();
		}
		return null;
	}
	/**
	 * 关闭资源
	 */
	public static void close(Connection conn,Statement sta,PreparedStatement psta,ResultSet rs){
		try {
			if(conn!=null){
				conn.close();
			}
			if(sta!=null){
				sta.close();
			}
			if(psta!=null){
				psta.close();
			}
			if(rs!=null){
				rs.close();
			}
		} catch (SQLException e) {
			e.printStackTrace();
		}
	} 
	public static void close(Connection conn){
		try {
			if(conn!=null){
				conn.close();
			}
		} catch (SQLException e) {
			e.printStackTrace();
		}
	} 
}
```

## 2.8 事务

* 提交 conn.commit();
* 回滚 conn.rollback();

```java
try {
			//1. 获得数据库连接
			conn=JDBCUtils.getConnection();
			//设置自动提交为手动提交
			conn.setAutoCommit(false);
			//2. 获取预命令对象
			String sql="update emp set emp_salary=? where emp_name=?";
			psta = conn.prepareStatement(sql);
			//3. 执行
			psta.setDouble(1, 1500);
			psta.setString(2, "李四");
			int i = psta.executeUpdate();
			System.out.println(i);
    	//**************************
    		// 提交操作
			conn.commit();
		} catch (Exception e) {
			e.printStackTrace();
			try {
				conn.rollback();//回滚操作
			} catch (SQLException e1) {
				e1.printStackTrace();
			}
		}finally{
			//4. 关闭资源
			JDBCUtils.close(conn, null, psta, null);
		}
```

## 2.9 批处理

* psta.addBatch()

```java
Connection conn=null;
PreparedStatement psta =null;
	try {
		conn=JDBCUtils.getConnection();
		String sql="insert into user values(null,?,?)";
		psta = conn.prepareStatement(sql);
		
		for (int i = 1; i <= 50000; i++) {
			psta.setString(1, "bigdata"+i);
			psta.setString(2, "java"+i);
			//将新增数据放在容器中
			psta.addBatch();
			//当容器中的数据达到1000个就进行批量添加
			if(i%1000==0){
				psta.executeBatch();
			}
		}
	} catch (SQLException e) {
		e.printStackTrace();
	}finally{
		JDBCUtils.close(conn, null, psta, null);
	}
```

## 2.10 DBUtils

### 核心

* QueryRunner 类
* update()，query()方法

```java
QueryRunner qr=new QueryRunner();

方法：
int update(Connection conn,String sql,Object...params);
T query(Connection conn,String sql,ResultSetHandler handler,Object...params)
    
//只有一条数据--->一个对象
BeanHandler<User> bh=new BeanHandler<>(User.class);
	//泛型是设置query方法的返回值的
	//Class类型的参数是设置自动装配信息    结果集的列名一定要和对象的属性名保持一致

//多条数据       --->集合或者对象数组
BeanListHandler<User> blh=new BeanListHandler<>(User.class);
	//泛型是设置query方法返回值集合的泛型
	//Class类型的参数是设置自动装配信息    结果集的列名一定要和对象的属性名保持一致

//只有一个值   --->一个变量
ScalarHandler<Long> sh=new ScalarHandler<>();
	//泛型是设置query方法的返回值类型
```

### 增删改

```java
public void test1(){
		//1. 创建核心类的对象
		QueryRunner qr=new QueryRunner();
		Connection conn=JDBCUtils.getConnection();
		String sql="update user set password=? where user_id=?";
		try {
			int i = qr.update(conn, sql, "admin",11);
			System.out.println(i);
		} catch (SQLException e) {
			e.printStackTrace();
		}finally{
			JDBCUtils.close(conn);
		}
	}
```

### 查询

```java
public void test3(){
		QueryRunner qr=new QueryRunner();
		Connection conn=JDBCUtils.getConnection();
		String sql="select user_id userId,username,password passwrod from user where user_id<?";
		BeanListHandler<User> blh=new BeanListHandler<>(User.class);
		try {
			List<User> list = qr.query(conn, sql, blh, 15);
			list.stream().forEach(System.out::println);
		} catch (SQLException e) {
			e.printStackTrace();
		}finally{
			JDBCUtils.close(conn);
		}
	}
```

## 2.11 BaseDao

* 利用DBUtils封装几个方法
* **update** 数据的增删改
* **getBean** 查询一条数据，返回一个对象
* **getBeanList ** 查询多条数据，返回一个List
* **getOnly ** 查询一个值，返回一个值

```java
public class BaseDao<T> {
	private QueryRunner qr=new QueryRunner();
	/**
	 * 功能：完成数据的增删改
	 * @param sql
	 * @param params
	 * @return
	 */
	public boolean update(String sql,Object...params){
		Connection conn=JDBCUtils.getConnection();
		try {
			int i = qr.update(conn, sql, params);
			if(i<=0)
				return false;
		} catch (SQLException e) {
			e.printStackTrace();
		}finally{
			JDBCUtils.close(conn);
		}
		return true;
	}
	/**
	 * 功能：查询单条结果：封装成一个对象
	 * @param sql
	 * @param type
	 * @param params
	 * @return
	 */
	public T getBean(String sql,Class type,Object...params){
		Connection conn=JDBCUtils.getConnection();
		BeanHandler<T> bh=new BeanHandler<>(type);
		try {
			return qr.query(conn, sql, bh, params);
		} catch (SQLException e) {
			e.printStackTrace();
		}finally{
			JDBCUtils.close(conn);
		}
		return null;
	}
	/**
	 * 功能：查询多条结果：封装成一个集合对象
	 * @param sql
	 * @param type
	 * @param params
	 * @return
	 */
	public List<T> getBeanList(String sql,Class type,Object...params){
		Connection conn=JDBCUtils.getConnection();
		BeanListHandler<T> blh=new BeanListHandler<>(type);
		try {
			return qr.query(conn, sql, blh, params);
		} catch (SQLException e) {
			e.printStackTrace();
		}finally{
			JDBCUtils.close(conn);
		}
		return null;
	}
	/**
	 * 功能：返回单个数据   直接用Object去接收(U)
	 * @param sql
	 * @param params
	 * @return
	 */
	public<U> U getOnly(String sql,Object...params){
		Connection conn=JDBCUtils.getConnection();
		ScalarHandler<U> sh=new ScalarHandler<>();
		try {
			return qr.query(conn, sql, sh, params);
		} catch (SQLException e) {
			e.printStackTrace();
		}finally{
			JDBCUtils.close(conn);
		}
		return null;
	}
	
}
```

