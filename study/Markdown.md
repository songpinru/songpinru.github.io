[TOC]

# Markdown 标题

```md
# 一级标题
## 二级标题
### 三级标题
#### 四级标题
##### 五级标题
###### 六级标题
```

# 一级标题

## 二级标题
### 三级标题
#### 四级标题
##### 五级标题
###### 六级标题

----

# Markdown 字体
### 字体
```md
*斜体文本*
_斜体文本_
**粗体文本**
__粗体文本__
***粗斜体文本***
___粗斜体文本___
```

*斜体文本*
_斜体文本_
**粗体文本**
__粗体文本__
***粗斜体文本***
___粗斜体文本___

----
###  删除线
```md
RUNOOB.COM
GOOGLE.COM
~~BAIDU.COM~~
```
RUNOOB.COM
GOOGLE.COM
~~BAIDU.COM~~

*****
### 下划线

```
<u>带下划线文本</u>
```
<u>带下划线文本</u>

******
###  脚注

```
[^RUNOOB]。

[^RUNOOB]: 菜鸟教程 -- 学的不仅是技术，更是梦想！！！
```

 [^RUNOOB]





[^RUNOOB]: 菜鸟教程 -- 学的不仅是技术，更是梦想！！！

# Markdown 列表

### 列表
```md
* 第一项
* 第二项
* 第三项
+ 第一项
+ 第二项
+ 第三项
- 第一项
- 第二项
- 第三项
```

* 第一项
* 第二项
* 第三项
* 第一项

+ 第二项
+ 第三项


- 第一项
- 第二项
- 第三项

----

```md
1. 第一项
2. 第二项
3. 第三项
```
1. 第一项
2. 第二项
3. 第三项

```md
- 第一项：
    1. 第一项嵌套的第一个元素
    1. 第一项嵌套的第二个元素
2. 第二项：
    - 第二项嵌套的第一个元素
    - 第二项嵌套的第二个元素
```
-  第一项：
    1.  第一项嵌套的第一个元素
    2. 第一项嵌套的第二个元素
-  第二项：
    - 第二项嵌套的第一个元素
    - 第二项嵌套的第二个元素
### 区块
```md
> 区块引用
> 菜鸟教程
> 学的不仅是技术更是梦想
> 最外层
> > 第一层嵌套
> > > 第二层嵌套
```

> 区块引用
> 菜鸟教程
> 学的不仅是技术更是梦想
> 最外层
> > 第一层嵌套
> >
> > > 第二层嵌套

# Markdown 代码

```
`printf()` 函数
```
`printf()` 函数

```javascript
$(document).ready(function () {
    alert('RUNOOB');
});
```

# Markdown 链接
### 一般使用
```md
[链接名称](链接地址)
这是一个链接 [菜鸟教程](https://www.runoob.com)
或者
<链接地址>
<https://www.runoob.com>
```

[链接名称](链接地址)
这是一个链接 [菜鸟教程](https://www.runoob.com)

或者

<链接地址>
<https://www.runoob.com>

-----
### 高级

```
链接也可以用变量来代替，文档末尾附带变量地址：
这个链接用 1 作为网址变量 [Google][1]
这个链接用 runoob 作为网址变量 [Runoob][runoob]
然后在文档的结尾为变量赋值（网址）

  [1]: http://www.google.com/
  [runoob]: http://www.runoob.com/
```

链接也可以用变量来代替，文档末尾附带变量地址：
这个链接用 1 作为网址变量 [Google][1]
这个链接用 runoob 作为网址变量 [Runoob][runoob]
然后在文档的结尾为变量赋值（网址）

[1]: http://www.google.com/
[runoob]: http://www.runoob.com/

# Markdown 图片

### 图片
```
![alt 属性文本](图片地址)
![RUNOOB 图标](http://static.runoob.com/images/runoob-logo.png)

![alt 属性文本](图片地址 "可选标题")
![RUNOOB 图标](http://static.runoob.com/images/runoob-logo.png "RUNOOB")
```
![RUNOOB 图标](http://static.runoob.com/images/runoob-logo.png)

![RUNOOB 图标](http://static.runoob.com/images/runoob-logo.png "RUNOOB")

```
这个链接用 1 作为网址变量 [RUNOOB][1].
然后在文档的结尾为变量赋值（网址）

[1]: http://static.runoob.com/images/runoob-logo.png
```
[RUNOOB][3]

[3]: http://static.runoob.com/images/runoob-logo.png

### 视频

您可以使用 <video> HTML标记嵌入视频，例如：

<video src="xxx.mp4" />

# Markdown 表格

```
| 左对齐 | 右对齐 | 居中对齐 |
| :-----| ----: | :----: |
| 单元格 | 单元格 | 单元格 |
| 单元格 | 单元格 | 单元格 |
```

| 左对齐 | 右对齐 | 居中对齐 |
| :----- | -----: |: -----: |
| 单元格 | 单元格 |   单元格 |
| 单元格 | 单元格 |   单元格 |

# Markdown 高级技巧

### 支持的 HTML 元素

不在 Markdown 涵盖范围之内的标签，都可以直接在文档里面用 HTML 撰写。

目前支持的 HTML 元素有：`<kbd> <b> <i> <em> <sup> <sub> <br>`等 ，如：

```
使用 <kbd>Ctrl</kbd>+<kbd>Alt</kbd>+<kbd>Del</kbd> 重启电脑
```

使用 <kbd>Ctrl</kbd>+<kbd>Alt</kbd>+<kbd>Del</kbd> 重启电脑

#### 折叠

<details>
<summary>CLICK ME</summary>
**<summary>标签与正文间一定要空一行！！！**
</details>

```xml
<details>
<summary>CLICK ME</summary>

**<summary>标签与正文间一定要空一行！！！**
</details>
```

### YAML Front Matter
Typora 现在支持 YAML Front Matter 。 在文章顶部输入 --- 然后按 Enter 键将引入一个，或者从菜单中插入一个元数据块。
```
---

---
```



### 目录 (TOC)
输入 [toc] 然后按 Return 键将创建一个“目录”部分，自动从文档内容中提取所有标题，其内容会自动更新。
[TOC]

### 公式

当你需要在编辑器中插入数学公式时，可以使用两个美元符 $$ 包裹 TeX 或 LaTeX 格式的数学公式来实现。提交后，问答和文章页会根据需要加载 Mathjax 对数学公式进行渲染。如：

```
$$
\mathbf{V}_1 \times \mathbf{V}_2 =  \begin{vmatrix} 
\mathbf{i} & \mathbf{j} & \mathbf{k} \\
\frac{\partial X}{\partial u} &  \frac{\partial Y}{\partial u} & 0 \\
\frac{\partial X}{\partial v} &  \frac{\partial Y}{\partial v} & 0 \\
\end{vmatrix}
${$tep1}{\style{visibility:hidden}{(x+1)(x+1)}}
$$
```

$$
\mathbf{V}_1 \times \mathbf{V}_2 =  \begin{vmatrix} 
\mathbf{i} & \mathbf{j} & \mathbf{k} \\
\frac{\partial X}{\partial u} &  \frac{\partial Y}{\partial u} & 0 \\
\frac{\partial X}{\partial v} &  \frac{\partial Y}{\partial v} & 0 \\
\end{vmatrix}
${fs}{\style{visibility:hidden}{(x+1)(x+1)}}
$$
# 图形和流

https://mermaid-js.github.io/mermaid

以下几个实例效果图如下：

**1、横向流程图源码格式：**

```
```mermaid
graph LR
[方形] -->B(圆角)
    B --> C{条件a}
    C -->|a=1| D[结果1]
    C -->|a=2| E[结果2]
    F[横向流程图]
​```
```

```mermaid
graph LR
A[方形] -->B(圆角)
    B --> C{条件a}
    C -->|a=1| D[结果1]
    C -->|a=2| E[结果2]
    F[横向流程图]
```

**2、竖向流程图源码格式：**

```
````mermaid
graph TD
A[方形] --> B(圆角)
    B --> C{条件a}
    C --> |a=1| D[结果1]
    C --> |a=2| E[结果2]
    F[竖向流程图]
​````
```

```mermaid
graph TD
A[方形] --> B(圆角)
    B --> C{条件a}
    C --> |a=1| D[结果1]
    C --> |a=2| E[结果2]
    F[竖向流程图]
```

**3、标准流程图源码格式：**

```
```flow
st=>start: 开始框:>url
op=>operation: 处理框
cond=>condition: 判断框(是或否?)
sub1=>subroutine: 子流程
io=>inputoutput: 输入输出框
e=>end: 结束框
st->op->cond
cond(yes)->io->e
cond(no)->sub1(right)->op
​```
```

```flow
st=>start: 开始框
op=>operation: 处理框
cond=>condition: 判断框(是或否?)
sub1=>subroutine: 子流程
io=>inputoutput: 输入输出框
e=>end: 结束框
st->op->cond
cond(yes)->io->e
cond(no)->sub1(right)->op
```

**4、标准流程图源码格式（横向）：**

```
```flow
st=>start: 开始框
op=>operation: 处理框
cond=>condition: 判断框(是或否?)
sub1=>subroutine: 子流程
io=>inputoutput: 输入输出框
e=>end: 结束框
st(right)->op(right)->cond
cond(yes)->io(bottom)->e
cond(no)->sub1(right)->op
​```
```

```flow
st=>start: 开始框
op=>operation: 处理框
cond=>condition: 判断框(是或否?)
sub1=>subroutine: 子流程
io=>inputoutput: 输入输出框
e=>end: 结束框
st(right)->op(right)->cond
cond(no)->sub1(right)->op
cond(yes)->io(bottom)->e
```

**5、UML时序图源码样例：**

```
```sequence
对象A->对象B: 对象B你好吗?（请求）
Note right of 对象B: 对象B的描述
Note left of 对象A: 对象A的描述(提示)
对象B-->对象A: 我很好(响应)
对象A->对象B: 你真的好吗？
​```
```

```sequence
对象A->对象B: 对象B你好吗?（请求）
Note right of 对象B: 对象B的描述
Note left of 对象A: 对象A的描述(提示)
对象B-->对象A: 我很好(响应)
对象A->对象B: 你真的好吗？
```

**6、UML时序图源码复杂样例：**

```
```sequence
Title: 标题：复杂使用
对象A->对象B: 对象B你好吗?（请求）
Note right of 对象B: 对象B的描述
Note left of 对象A: 对象A的描述(提示)
对象B-->对象A: 我很好(响应)
对象B->小三: 你好吗
小三-->>对象A: 对象B找我了
对象A->对象B: 你真的好吗？
Note over 小三,对象B: 我们是朋友
participant C
Note right of C: 没人陪我玩
​```
```

```sequence
Title: 标题：复杂使用
对象A->对象B: 对象B你好吗?（请求）
Note right of 对象B: 对象B的描述
Note left of 对象A: 对象A的描述(提示)
对象B-->对象A: 我很好(响应)
对象B->小三: 你好吗
小三-->>对象A: 对象B找我了
对象A->对象B: 你真的好吗？
Note over 小三,对象B: 我们是朋友
participant C
Note right of C: 没人陪我玩
```

**7、UML标准时序图样例：**

```
```mermaid
%% 时序图例子,-> 直线，-->虚线，->>实线箭头
  sequenceDiagram
    participant 张三
    participant 李四
    张三->王五: 王五你好吗？
    loop 健康检查
        王五->王五: 与疾病战斗
    end
    Note right of 王五: 合理 食物 <br/>看医生...
    李四-->>张三: 很好!
    王五->李四: 你怎么样?
    李四-->王五: 很好!
​```
```

```mermaid
%% 时序图例子,-> 直线，-->虚线，->>实线箭头
  sequenceDiagram
    participant 张三
    participant 李四
    张三->王五: 王五你好吗？
    loop 健康检查
        王五->王五: 与疾病战斗
    end
    Note right of 王五: 合理 食物 <br/>看医生...
    李四-->>张三: 很好!
    王五->李四: 你怎么样?
    李四-->王五: 很好!
```

**8、甘特图样例：**

```
```mermaid
%% 语法示例
        gantt
        dateFormat  YYYY-MM-DD
        title 软件开发甘特图
        section 设计
        需求                      :done,    des1, 2014-01-06,2014-01-08
        原型                      :active,  des2, 2014-01-09, 3d
        UI设计                     :         des3, after des2, 5d
    	未来任务                     :         des4, after des3, 5d
        section 开发
        学习准备理解需求                      :crit, done, 2014-01-06,24h
        设计框架                             :crit, done, after des2, 2d
        开发                                 :crit, active, 3d
        未来任务                              :crit, 5d
        耍                                   :2d
        section 测试
        功能测试                              :active, a1, after des3, 3d
        压力测试                               :after a1  , 20h
        测试报告                               : 48h
​```
```

```mermaid
%% 语法示例
        gantt
        dateFormat  YYYY-MM-DD
        title 软件开发甘特图
        section 设计
        需求                      :done,    des1, 2014-01-06,2014-01-08
        原型                      :active,  des2, 2014-01-09, 3d
        UI设计                     :         des3, after des2, 5d
    	未来任务                     :         des4, after des3, 5d
        section 开发
        学习准备理解需求                      :crit, done, 2014-01-06,24h
        设计框架                             :crit, done, after des2, 2d
        开发                                 :crit, active, 3d
        未来任务                              :crit, 5d
        耍                                   :2d
        section 测试
        功能测试                              :active, a1, after des3, 3d
        压力测试                               :after a1  , 20h
        测试报告                               : 48h
```

# 备注

```
* [ ]   ]后接空格
```
* [ ] 

* [ ] 理解Java语言的跨平台实现原理

###### text[^referenceID]

[^referenceID]: Computer Aided Translation[EB/OL].  https://www.microsoft.com/en-us/research/video/computer-aided-translation/, 2016/2018-8.19.

## Markdown 增加文献引用

这边文章是介绍如何在 Markdown 中增加文献引用。[xxx][1]

## 参考



[1] [百度学术](http://xueshu.baidu.com/)

[百度学术]:http://xueshu.baidu.com/	"ssssssssss"



[ssss]: http://kkkkk.com



[2] [Wikipedia](https://en.wikipedia.org/wiki/Main_Page)



## Markdown 增加文献引用

这边文章是介绍如何在 Markdown 中增加文献引用。[<sup>1</sup>](#refer-anchor-1)

## 参考

<div id="refer-anchor-1"></div>

- [1] [百度学术](http://xueshu.baidu.com/)

<div id="refer-anchor-2"></div>

- [2] [Wikipedia](https://en.wikipedia.org/wiki/Main_Page)

```js
> Quote here.
>
> -- <cite>Benjamin Franklin</cite>
```

> Quote here.
>
> -- <cite>Benjamin Franklin</cite>

