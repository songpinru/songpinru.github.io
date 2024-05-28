# Shell变量

## 基础用法

* 定义变量：变量=值 
* 撤销变量：unset 变量
* 声明静态变量：readonly变量，注意：不能unset

PS：

* 变量名称可以由字母、数字和下划线组成，但是不能以数字开头，环境变量名建议大写。
* ==等号两侧不能有空格==
* 在bash中，变量默认类型都是字符串类型，无法直接进行数值运算。
* 变量的值如果有空格，需要使用双引号或单引号括起来。

## 生命周期

脚本执行方式：

* fork
  * 新开一个shell执行，就是调用sh或者直接执行可执行的shell脚本
* exec
  * exec命令，把执行权交给新的脚本，当前脚本后续内容不执行，全局变量会清空（所以表现和新shell一样）
* source
  * source（.），还在本shell执行，复用变量



变量分为3种：

* 局部变量
  * local
  * 只在当前函数内生效
* 全局变量
  * 默认就是全局变量，在当前shell生效（source）
* 环境变量
  * export
  * 可以被子shell继承的变量（exec，fork）

## 特殊变量

### $n

```
n为数字，$0代表该脚本名称，$1-$9代表第一到第九个参数，十以上的参数，十以上的参数需要用大括号包含，如${10}
```

### $#

```
获取所有输入参数个数
```

### $*  , \$@

```
$* 这个变量代表命令行中所有的参数，$*把所有的参数看成一个整体
$@ 这个变量也代表命令行中所有的参数，不过$@把每个参数区分对待
```

### $?

```
最后一次执行的命令的返回状态。如果这个变量的值为0，证明上一个命令正确执行；如果这个变量的值为非0（具体是哪个数，由命令自己来决定），则证明上一个命令执行不正确了。
```

### $$

```
脚本运行的当前进程ID号
```

### $!

```
后台运行的最后一个进程的ID号
```

# 字符串

双引号的优点：

- 双引号里可以有变量
- 双引号里可以出现转义字符

## 拼接字符串

```bash
your_name="runoob"
# 使用双引号拼接
greeting="hello, "$your_name" !"
greeting_1="hello, ${your_name} !"
echo $greeting  $greeting_1
# 使用单引号拼接
greeting_2='hello, '$your_name' !'
greeting_3='hello, ${your_name} !'
echo $greeting_2  $greeting_3
```

输出结果为：

```bash
hello, runoob ! hello, runoob !
hello, runoob ! hello, ${your_name} !
```

## 获取字符串长度

```bash
string="abcd"
echo ${#string} #输出 4
```

提取子字符串

以下实例从字符串第 **2** 个字符开始截取 **4** 个字符：

```bash
string="runoob is a great site"
echo ${string:1:4} # 输出 unoo
```

**注意**：第一个字符的索引值为 **0**。

## 查找子字符串

查找字符 **i** 或 **o** 的位置(哪个字母先出现就计算哪个)：

```bash
string="runoob is a great site"
echo `expr index "$string" io`  # 输出 4
```

**注意：** 以上脚本中 **`** 是反引号，而不是单引号 **'**，不要看错了哦。

## 字符串操作

**判断读取字符串值**

> | 表达式          | 含义                                                        |
> | --------------- | ----------------------------------------------------------- |
> | ${var}          | 变量var的值, 与$var相同                                     |
> |                 |                                                             |
> | ${var-DEFAULT}  | 如果var没有被声明, 那么就以$DEFAULT作为其值 *               |
> | ${var:-DEFAULT} | 如果var没有被声明, 或者其值为空, 那么就以$DEFAULT作为其值 * |
> |                 |                                                             |
> | ${var=DEFAULT}  | 如果var没有被声明, 那么就以$DEFAULT作为其值 *               |
> | ${var:=DEFAULT} | 如果var没有被声明, 或者其值为空, 那么就以$DEFAULT作为其值 * |
> |                 |                                                             |
> | ${var+OTHER}    | 如果var声明了, 那么其值就是$OTHER, 否则就为null字符串       |
> | ${var:+OTHER}   | 如果var被设置了, 那么其值就是$OTHER, 否则就为null字符串     |
> |                 |                                                             |
> | ${var?ERR_MSG}  | 如果var没被声明, 那么就打印$ERR_MSG *                       |
> | ${var:?ERR_MSG} | 如果var没被设置, 那么就打印$ERR_MSG *                       |
> |                 |                                                             |
> | ${!varprefix*}  | 匹配之前所有以varprefix开头进行声明的变量                   |
> | ${!varprefix@}  | 匹配之前所有以varprefix开头进行声明的变量                   |

加入了“*” 不是意思是： 当然, 如果变量var已经被设置的话, 那么其值就是$var.

**字符串操作（长度，读取，替换）**

> | 表达式                                                       | 含义 |
> | ------------------------------------------------------------ | ---- |
> | ${#string}                       | $string的长度             |      |
> |                                                              |      |
> | ${string:position}               | 在$string中, 从位置$position开始提取子串 |      |
> | ${string:position:length}        | 在$string中, 从位置$position开始提取长度为$length的子串 |      |
> |                                                              |      |
> | \${string#substring}              \| 从变量$string的开头, 删除最短匹配substring的子串 |      |
> | \${string##substring}             \| 从变量$string的开头, 删除最长匹配substring的子串 |      |
> | \${string%substring}              \| 从变量$string的结尾, 删除最短匹配substring的子串 |      |
> | \${string%%substring}             \| 从变量$string的结尾, 删除最长匹配substring的子串 |      |
> |                                                              |      |
> | ${string/substring/replacement}  | 使用$replacement, 来代替第一个匹配的substring |      |
> | ${string//substring/replacement} | 使用$replacement, 代替所有匹配的substring |      |
> | \${string/#substring/replacement} \| 如果$string的*前缀*匹配substring, 那么就用replacement来代替匹配到的substring |      |
> | \${string/%substring/replacement} \| 如果$string的*后缀*匹配substring, 那么就用replacement来代替匹配到的substring |      |
> |                                                              |      |

**说明：substring可以是一个正则表达式**



## printf

```bash
#!/bin/bash
# author:菜鸟教程
# url:www.runoob.com
 
printf "%-10s %-8s %-4s\n" 姓名 性别 体重kg  
printf "%-10s %-8s %-4.2f\n" 郭靖 男 66.1234
printf "%-10s %-8s %-4.2f\n" 杨过 男 48.6543
printf "%-10s %-8s %-4.2f\n" 郭芙 女 47.9876
```

执行脚本，输出结果如下所示：

```
姓名     性别   体重kg
郭靖     男      66.12
杨过     男      48.65
郭芙     女      47.99
```

**%s %c %d %f** 都是格式替代符，**％s** 输出一个字符串，**％d** 整型输出，**％c** 输出一个字符，**％f** 输出实数，以小数形式输出。

**%-10s** 指一个宽度为 10 个字符（**-** 表示左对齐，没有则表示右对齐），任何字符都会被显示在 10 个字符宽的字符内，如果不足则自动以空格填充，超过也会将内容全部显示出来。

**%-4.2f** 指格式化为小数，其中 **.2** 指保留2位小数。

## printf 的转义序列

| 序列  | 说明                                                         |
| ----- | ------------------------------------------------------------ |
| \a    | 警告字符，通常为ASCII的BEL字符                               |
| \b    | 后退                                                         |
| \c    | 抑制（不显示）输出结果中任何结尾的换行字符（只在%b格式指示符控制下的参数字符串中有效），而且，任何留在参数里的字符、任何接下来的参数以及任何留在格式字符串中的字符，都被忽略 |
| \f    | 换页（formfeed）                                             |
| \n    | 换行                                                         |
| \r    | 回车（Carriage return）                                      |
| \t    | 水平制表符                                                   |
| \v    | 垂直制表符                                                   |
| \\    | 一个字面上的反斜杠字符                                       |
| \ddd  | 表示1到3位数八进制值的字符。仅在格式字符串中有效             |
| \0ddd | 表示1到3位的八进制值字符                                     |

# 数组

```
array_name=(value1 value2 ... valuen)
```

## 实例

```bash
#!/bin/bash
# author:菜鸟教程
# url:www.runoob.com

my_array=(A B "C" D)
```

我们也可以使用下标来定义数组:

```bash
array_name[0]=value0
array_name[1]=value1
array_name[2]=value2
```

## 读取数组

读取数组元素值的一般格式是：

```bash
${array_name[index]}
```

## 获取数组中的所有元素

使用@ 或 * 可以获取数组中的所有元素，例如：

```bash
#!/bin/bash
# author:菜鸟教程
# url:www.runoob.com

my_array[0]=A
my_array[1]=B
my_array[2]=C
my_array[3]=D

echo "数组的元素为: ${my_array[*]}"
echo "数组的元素为: ${my_array[@]}"
```

执行脚本，输出结果如下所示：

```bash
$ chmod +x test.sh 
$ ./test.sh
数组的元素为: A B C D
数组的元素为: A B C D
```

## 获取数组的长度

获取数组长度的方法与获取字符串长度的方法相同，例如：

```bash
#!/bin/bash
# author:菜鸟教程
# url:www.runoob.com

my_array[0]=A
my_array[1]=B
my_array[2]=C
my_array[3]=D

echo "数组元素个数为: ${#my_array[*]}"
echo "数组元素个数为: ${#my_array[@]}"
```

执行脚本，输出结果如下所示：

```bash
$ chmod +x test.sh 
$ ./test.sh
数组元素个数为: 4
数组元素个数为: 4
```

# 运算式

**基本语法**

1. \$((运算式)) 或 \$[运算式]
2. expr  + , - , \*,  /,  %    加，减，乘，除，取余

注意：expr运算符间要有空格

# 条件判断

## 基本语法

[ condition ]（注意condition前后要有空格）

注意：条件非空即为true，[ test ]返回true，[] 返回false。

## 常用判断条件

1. 两个整数之间比较
```bash
= 字符串比较

-lt 小于（less than）                  -le 小于等于（less equal）

-eq 等于（equal）                       -gt 大于（greater than）

-ge 大于等于（greater equal）    -ne 不等于（Not equal）
```
2. 按照文件权限进行判断
```bash
-r 有读的权限（read）                     -w 有写的权限（write）

-x 有执行的权限（execute）
```
3. 按照文件类型进行判断
```bash
-f 文件存在并且是一个常规的文件（file）

-e 文件存在（existence）           -d 文件存在并是一个目录（directory）
```

## shell 条件判断语句整理

#### 常用系统变量

1)     $0 当前程式的名称

2)     $n 当前程式的第n个参数,n=1,2,…9

3)     $* 当前程式的任何参数(不包括程式本身)

4)     $# 当前程式的参数个数(不包括程式本身)

5)     $$ 当前程式的PID

6)     $! 执行上一个指令的PID(似乎不行?)

7)     $? 执行上一个指令的返回值

 

#### 条件判断:expression为字符串操作

1)     -n str 	字符串str是否不为空

2)     -z str 	字符串str是否为空

3)     str1 =str2 	str1是否和str2相同

4)     str1!=str2 	str1是否和str2不同

 

#### 条件判断:expression为整数操作

1)     expr1 -a expr2 假如 expr1 和 expr2 评估为真，则为真

2)     expr1 -o expr2 假如 expr1 或 expr2 评估为真，则为真

 

#### 条件判断:expression为bool操作

1)     int1 -eq int2 假如int1等于int2,则为真

2)     int1 -ge int2 假如int1大于或等于int2,则为真

3)     int1 -gt int2 假如int1大于int2 ,则为真

4)     int1 -le int2 假如int1小于或等于int2 ,则为真

5)     int1 -lt int2 假如int1小于int2 ,则为真

6)     int1 -ne int2 假如int1不等于int2 ,则为真

 

#### 条件判断:expression为文档操作

1)     -b 是否块文档

2)     -p 文档是否为一个命名管道

3)     -c 是否字符文档

4)     -r 文档是否可读

5)     -d 是否一个目录

6)     -s 文档的长度是否不为零

7)     -e 文档是否存在

8)     -S 是否为套接字文档

9)     -f 是否普通文档

10)   -x 文档是否可执行，则为真

11)   -g 是否配置了文档的 SGID 位

12)   -u 是否配置了文档的 SUID 位

13)   -G 文档是否存在且归该组任何

14)   -w 文档是否可写，则为真

15)   -k 文档是否配置了的粘贴位

16)   -t fd fd 是否是个和终端相连的打开的文档描述符（fd 默认为 1）

17)   -O 文档是否存在且归该用户任何

# 流程控制

## IF
```bash
if [ 条件判断式 ];then 
  程序 
fi 
#或者 
if [ 条件判断式 ] 
  then 
程序 
elif [ 条件判断式 ]
       then
              程序
else
       程序
fi
```
注意事项：

1. [ 条件判断式 ]，中括号和条件判断式之间必须有空格

2. ==if后要有空格==

## CASE

```bash
case $变量名 in 
  "值1"） 
    如果变量的值等于值1，则执行程序1 
    ;; 
  "值2"） 
    如果变量的值等于值2，则执行程序2 
    ;; 
  …省略其他分支… 
  *） 
    如果变量的值都不是以上的值，则执行此程序 
    ;; 
esac
```

注意事项：

1.  case行尾必须为单词“in”，每一个模式匹配必须以右括号“）”结束。

2.  双分号“**;;**”表示命令序列结束，相当于java中的break。

## FOR

```bash
for (( 初始值;循环控制条件;变量变化 )) 
  do 
    程序 
  done
  
for 变量 in 值1 值2 值3… 
  do 
    程序 
  done

```

## WHILE

```bash
while [ 条件判断式 ] 
  do 
    程序
  done
```

## until

```bash
until condition
do
    command
done
```

## 无限循环

无限循环语法格式：

```bash
while :
do
    command
done
```

或者

```bash
while true
do
    command
done
```

或者

```bash
for (( ; ; ))
```

## 跳出循环

break

continue

# 函数

```bash
[ function ] funname [()] #如果有function，可以省略括号，但是和{必须要有空格

{

    action;

    [return int;]

}
```

# 输入输出

## read

```bash
read(选项)(参数)
	选项：
-p：指定读取值时的提示符；
-t：指定读取值时等待的时间（秒）。
参数
	变量：指定读取值的变量名
read -t 7 -p "Enter your name in 7 seconds " NAME
```

## 重定向

| 命令            | 说明                                               |
| --------------- | -------------------------------------------------- |
| command > file  | 将输出重定向到 file。                              |
| command < file  | 将输入重定向到 file。                              |
| command >> file | 将输出以追加的方式重定向到 file。                  |
| n > file        | 将文件描述符为 n 的文件重定向到 file。             |
| n >> file       | 将文件描述符为 n 的文件以追加的方式重定向到 file。 |
| n >& m          | 将n导至m                                           |
| n <& m          | 将 m 和 n 合并。                                   |
| << tag          | 将开始标记 tag 和结束标记 tag 之间的内容作为输入。 |

> 需要注意的是文件描述符 0 通常是标准输入（STDIN），1 是标准输出（STDOUT），2 是标准错误输出（STDERR）。

# 工具

## 正则

linux的正则分为基本正则和扩展正则

扩展正则即日常使用的正则，需要使用-r或-E开启

基本正则遇到以下字符需要加转义（为了避免混乱，使用扩展正则即可）：

```
? + { } ( )
```



## cut

```bash
cut [选项]... [文件]...
  -b, --bytes=列表		只选中指定的这些字节
  -c, --characters=列表		只选中指定的这些字符
  -d, --delimiter=分界符	使用指定分界符代替制表符作为区域分界
  -f, --fields=LIST       列
  -n                      with -b: don't split multibyte characters
      --complement		补全选中的字节、字符或域
  -s, --only-delimited		不打印没有包含分界符的行
      --output-delimiter=字符串	使用指定的字符串作为输出分界符，默认采用输入
				的分界符

仅使用f -b, -c 或-f 中的一个。每一个列表都是专门为一个类别作出的，或者您可以用逗号隔
开要同时显示的不同类别。您的输入顺序将作为读取顺序，每个仅能输入一次。
每种参数格式表示范围如下：
    N	从第1 个开始数的第N 个字节、字符或域
    N-	从第N 个开始到所在行结束的所有字符、字节或域
    N-M	从第N 个开始到第M 个之间(包括第M 个)的所有字符、字节或域
    -M	从第1 个开始到第M 个之间(包括第M 个)的所有字符、字节或域

当没有文件参数，或者文件不存在时，从标准输入读取
```

```bash
cut -d " " -f 2,3 cut.txt 
shen
zhen
 wo
 lai
 le
```

## sed

```
sed [-hnV][-e<script>][-f<script文件>][文本文件]
```

**参数说明**：

- -e<script>或--expression=<script> 以选项中指定的script来处理输入的文本文件。多个指令时使用
- -f<script文件>或--file=<script文件> 以选项中指定的script文件来处理输入的文本文件。
- -i 直接修改源文件，慎用
- -h或--help 显示帮助。
- -n或--quiet或--silent 仅显示script处理后的结果。
- -V或--version 显示版本信息。

**动作说明**：

- a ：新增， a 的后面可以接字串，而这些字串会在新的一行出现(目前的下一行)～
- c ：取代， c 的后面可以接字串，这些字串可以取代 n1,n2 之间的行！
- d ：删除，因为是删除啊，所以 d 后面通常不接任何咚咚；
- i ：插入， i 的后面可以接字串，而这些字串会在新的一行出现(目前的上一行)；
- p ：打印，亦即将某个选择的数据印出。通常 p 会与参数 sed -n 一起运行～
- s ：取代，可以直接进行取代的工作哩！通常这个 s 的动作可以搭配正规表示法！例如 1,20s/old/new/g 就是啦！
  - /g 全局，否则就是每行第一个匹配到的

### 用法

q 命令的作用是使 sed 命令在第一次匹配任务结束后，退出 sed 程序，不再进行对后续数据的处理。

```bash
[address]s/pattern/replacement/flags #正则替换（匹配的内容）
[address]d #删除行
[address]a（或 i）\新文本内容 #新增行
[address]c\用于替换的新文本 #替换行
[address]y/inchars/outchars/ #字符映射替换
[address]p #打印print
[address]w filename #写入文件write
[address]r filename #文件内容插入（追加）


[address]脚本命令
或者
address {
    多个脚本命令
}

address：
数字表示，1:首行，$:尾行，表示范围用,
sed '2,$s/dog/cat/'
正则表示：/pattern/,前后加/
sed '/demo/s/bash/csh/'
sed '/Mar  8 14:14/,/Mar  8 14:15/p'
```

| flags 标记 | 功能                                                         |
| ---------- | ------------------------------------------------------------ |
| n          | 1~512 之间的数字，表示指定要替换的字符串出现第几次时才进行替换，例如，一行中有 3 个 A，但用户只想替换第二个 A，这是就用到这个标记； |
| g          | 对数据中所有匹配到的内容进行替换，如果没有 g，则只会在第一次匹配成功时做替换操作。例如，一行数据中有 3 个 A，则只会替换第一个 A； |
| p          | 会打印与替换命令中指定的模式匹配的行。此标记通常与 -n 选项一起使用。 |
| w file     | 将缓冲区中的内容写到指定的 file 文件中；                     |
|            |                                                              |
| &          | 用正则表达式匹配的内容进行替换；                             |
| \n         | 匹配第 n 个子串(正则表达式的字串)，该子串之前在 pattern 中用 \(\) 指定。 |
| \          | 转义（转义替换部分包含：&、\ 等）。                          |

```bash

sed 's/要被取代的字串/新的字串/g'
sed 's/test/trial/w test.txt' data5.txt
sed 's/[a-z]*/(&)/g' 
 sed 's/\(a\)dfs/\1/g' demo.txt
## 按行替换
sed '2,5c No 2-5 number'
sed 'y/123/789/' data8.txt
```

## awk

**awk命令形式:**

```bash
awk [-F|-f|-v] ‘BEGIN{} //{command1; command2} END{}’ file
```



 [-F|-f|-v]  大参数，-F指定分隔符，-f调用脚本，-v定义变量 var=value

'  '      引用代码块

BEGIN  初始化代码块，在对每一行进行处理之前，初始化代码，主要是引用全局变量，设置FS分隔符

//      匹配代码块，可以是字符串或正则表达式，区间匹配是//,//

{}      命令代码块，包含一条或多条命令

；      多条命令使用分号分隔

END    结尾代码块，在对每一行进行处理之后再执行的代码块，主要是进行最终计算或输出结尾摘要信息

 

**特殊要点:**

$0      表示整个当前行

$1      每行第一个字段

NF      字段数量变量

NR      每行的记录号，多文件记录递增

FNR     与NR类似，不过多文件记录不递增，每个文件都从1开始

\t       制表符

\n      换行符

FS      BEGIN时定义分隔符

RS    输入的记录分隔符， 默认为换行符(即文本是按一行一行输入)

~       匹配，与==相比不是精确比较

!~      不匹配，不精确比较

==     等于，必须全部相等，精确比较

!=      不等于，精确比较

&&　   逻辑与

||       逻辑或

\+       匹配时表示1个或1个以上

/[0-9][0-9]+/  两个或两个以上数字

/[0-9][0-9]*/   一个或一个以上数字

FILENAME 文件名

OFS    输出字段分隔符， 默认也是空格，可以改为制表符等

ORS     输出的记录分隔符，默认为换行符,即处理结果也是一行一行输出到屏幕

-F'[:#/]'  定义三个分隔符



用法一：

```bash
awk '[pattern] {action}' {filenames}   # 行匹配语句 awk '' 只能用单引号
```

实例：

NF 表示的是浏览记录的域的个数
$NF 表示的最后一个Field（列），即输出最后一个字段的内容

```bash
# 每行按空格或TAB分割，输出文本中的1、4项
awk '{print $1,$4}' log.txt
# 格式化输出
awk '{printf "%-8s %-10s\n",$1,$4}' log.txt
# 区间匹配
awk '/14:14/,/14:14/{print NF}' syslog
```

用法二：

```bash
awk -F  #-F相当于内置变量FS, 指定分割字符
```

实例：

```bash
# 使用","分割
awk -F, '{print $1,$2}'   log.txt
# 或者使用内建变量
awk 'BEGIN{FS=","} {print $1,$2}'     log.txt
# 使用多个分隔符.先使用空格分割，然后对分割结果再使用","分割
awk -F '[ ,]'  '{print $1,$2,$5}'   log.txt
```

用法三：

```bash
awk -v  # 设置变量
```

实例：

```bash
awk -va=1 '{print $1,$1+a}' log.txt

awk -va=1 -vb=s '{print $1,$1+a,$1b}' log.txt
```

[linux之awk用法详解](https://blog.csdn.net/lukabruce/article/details/86692471)

## sort

sort(选项)(参数)

表1-57

| 选项 | 说明                     |
| ---- | ------------------------ |
| -n   | 依照数值的大小排序       |
| -r   | 以相反的顺序来排序       |
| -t   | 设置排序时所用的分隔字符 |
| -k   | 指定需要排序的列         |

参数：指定待排序的文件列表



按照“：”分割后的第三列倒序排序

```bash
sort -t : -nrk 3  sort.sh 
bb:40:5.4
bd:20:4.2
cls:10:3.5
xz:50:2.3
ss:30:1.6

```

## find

**语法**

```bash
find   path   -option  [  -print | -delete | -ls | -exec | -ok   command {} \; ]  

可以使用-and (-a)和 -or (-o)连接多个option，默认是and

在条件表达式前面加!表示对表达式取非。同样的也可以用-not参数。另外如果表达式很多，可以使用( expr )确定优先级，如：

find / ( -name "passwd" -a -type f ) -o ( -name "shadow" -a -type f )
```



**find命令的参数：**

1）path：要查找的目录路径。 

      ~ 表示$HOME目录
       . 表示当前目录
       / 表示根目录 
2）print：表示将结果输出到标准输出，默认就是print

3）exec：对匹配的文件执行该参数所给出的shell命令。 
      形式为command {} \;，注意{}与\;之间有空格 

4）ok：与exec作用相同，
      区别在于，在执行命令之前，都会给出提示，让用户确认是否执行 

6）options ：表示查找方式

options常用的有下选项：

```bash
-name   filename               #查找名为filename的文件
-perm   [-/]mode               #按执行权限来查找，-或/是非精确的意思
-user    username              #按文件属主来查找
-uid N                         #按用户id来查找，大于或者小于的
-group groupname               #按组来查找
-gid N                         #按组id来查找，大于或者小于的
-nogroup                       #查无有效属组的文件，即文件的属组在/etc/groups中不存在
-nouser                        #查无有效属主的文件，即文件的属主在/etc/passwd中不存

-atime    N                    #按文件访问时间来查找文件，-n指n天以内，+n指n天以前
-ctime    N                    #按文件创建时间来查找文件，-n指n天以内，+n指n天以前
-mtime   N                     #按文件更改时间来查找文件，-n指n天以内，+n指n天以前
-amin N                        #按文件访问时间来查找文件，分钟
-cmin N                        #按文件创建时间来查找文件，分钟
-mmin N                        #按文件更改时间来查找文件，分钟

-used N                        #上次更改后第N天访问的文件

-anewer FILE                   #按文件访问时间来查找文件，查找比file新的文件
-cnewer FILE                   #按文件创建时间来查找文件，查找比file新的文件
-newer FILE                    #按文件更改时间来查找文件，查找比file新的文件

-size      -n[bcwkMG]  +n[bcwkMG]            #根据文件大小查找，-n是小于，+n是大于
-type    b/d/c/p/l/f           #查是块设备、目录、字符设备、管道、符号链接、普通文件

-name PATTERN                  #根据文件名查找
-iname PATTERN                 #根据文件名查找，忽略大小写
-path PATTERN 				   #根据路径查找
-wholename PATTERN             #根据路径查找，同上
-iwholename PATTERN            #根据路径查找，忽略大小写
-regex PATTERN                 #文件名匹配，正则
-iregex PATTERN                #文件名匹配，正则，忽略大小写
-lname PATTERN                 #软连接匹配
-ilname PATTERN                #软连接匹配，忽略大小写

-inum N                        #根据inode id查找 
-links N                       #hard links数量 

-empty                          #空文件
-readable                       #可读
-writable                       #可写
-executable                     #可执行

-false                          #查找条件取反
-true

-depth -d                       #优先进入目录内查找（深度遍历）
-maxdepth LEVELS 
-mindepth LEVELS 
-mount                          #不去其他硬盘查找
-noleaf                         #查找没有子节点的文件（用来查找非unix设备的文件或）
-follow                         #如果遇到符号链接文件，就跟踪链接所指的文件
```

7）action：操作

默认是print

{}是占位符，会把找到的文件拼接到command里

常用的有：

```bash
-delete 
-ls 
-prune      #忽略f

-print
-printf FORMAT 
-fprint FILE 
-fprintf FILE FORMAT 

-exec COMMAND ; 
-exec COMMAND {} + -ok COMMAND ;
-execdir COMMAND ; 
-execdir COMMAND {} +  -okdir COMMAND ;
```





[ Linux 命令大全 ](https://www.runoob.com/linux/linux-command-manual.html)

