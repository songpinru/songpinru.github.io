# 结构

HTML基本同xml

主要有两个部分

`head`和`body`组成

head里主要放元数据

body才是显示的内容

```html
<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8">
<title>菜鸟教程(runoob.com)</title>
</head>
<body>
 
<h1>我的第一个标题</h1>
 
<p>我的第一个段落。</p>
 
</body>
</html>
```

# head

可以使用的标签
```html
<!--标题-->
<title></title>
<!--css样式或css引用地址-->
<style></style>
<!--元数据-->
<meta>
<!--资源链接-->
<link>
<!--脚本，js-->
<script></script>
<!--禁用脚本-->
<noscript/> 
<!--页面的默认链接地址uri-->
<base/>
```

# body

| 标签   | desc     |
| ------ | -------- |
| b      | 加粗     |
| i      | 斜体     |
| u/ins  | 下划线   |
| s/del  | 删除线   |
| em     | 斜体加重 |
| strong | 粗体加重 |
| big    | 大号字   |
| small  | 小号字   |
| sup    | 上标     |
| sub    | 下标     |

## HTML 分组标签

块级元素起新行

内联元素不起新行

| 标签 | 描述                                        |
| :--- | :------------------------------------------ |
| div  | 定义了文档的区域，块级 (block-level)        |
| span | 用来组合文档中的行内元素， 内联元素(inline) |

## img

图片标签

```html
<img src="./main/resources/数据流程.png" alt="图片" title="悬停文字"width="300" height="300" border="1" >
```

## video

视频标签

<video width="320" height="240" controls>   <source src="movie.mp4" type="video/mp4">   <source src="movie.ogg" type="video/ogg"> 您的浏览器不支持Video标签。 </video>

## audio

音频标签

<audio controls>   <source src="horse.ogg" type="audio/ogg">   <source src="horse.mp3" type="audio/mpeg"> 您的浏览器不支持 audio 元素。 </audio>

## 链接标签

href:链接，不加协议就是相对链接，#就是锚连接

> 锚链接#前面是资源地址，后面是标签id

target：默认是本网页跳转，blank是新页面

```html
<a href="https://www.baidu.com" target="_blank"></a>
```

## 列表

无序列表（unoderlist）

li：list


<ul>
    <li>kkk</li>
    <li>lll</li>
    <li>mmm</li>
</ul>
有序列表（orderlist）
<ol>
    <li>111</li>
    <li>222</li>
    <li>333</li>
</ol>
自定义列表（define list）

dt：define term

dd：define detail

<dl>
    <dt>drink</dt>
    <dd>coffee</dd>
    <dd>coffee</dd>
    <dd>coffee</dd>
    <dt>kkk</dt>
    <dd>121</dd>
    <dd>121</dd>
    <dd>121</dd>
    <dd>121</dd>
</dl>
## 表格

table

tr：table row

td：table data

th：table header 

> span：跨行或者跨列

<table border="1">
    <th>sss</th>
    <th>sss</th>
    <th>sss</th>
    <th>sss</th>
    <tr>
        <td>xxx</td>
        <td>xxx</td>
        <td>xxx</td>
        <td>xxx</td>
    </tr>
    <tr>
        <td>xxx</td>
        <td>xxx</td>
        <td>xxx</td>
        <td>xxx</td>
    </tr>
</table>
## HTML 表单标签

| 标签     | 描述                                         |
| :------- | :------------------------------------------- |
| form     | 定义供用户输入的表单                         |
| input    | 定义输入域                                   |
| textarea | 定义文本域 (一个多行的输入控件)              |
| label    | 定义了 <input> 元素的标签，一般为输入标题    |
| fiedset  | 定义了一组相关的表单元素，并使用外框包含起来 |
| legend   | 定义了 <fieldset> 元素的标题                 |
| select   | 定义了下拉选项列表                           |
| optgroup | 定义选项组                                   |
| option   | 定义下拉列表中的选项                         |
| button   | 定义一个点击按钮                             |
| datalist | 指定一个预先定义的输入控件选项列表           |
| keygen   | 定义了表单的密钥对生成器字段                 |
| output   | 定义一个计算结果                             |

tinput:type

```
button
checkbox
color
date
datetime
datetime-local
email
file
hidden
image
month
number
password
radio
range
reset
search
submit
tel
text
time
url
week
```



## Iframe

内嵌一个panel，展示不同的东西

target和name绑定


```html
<iframe src="demo_iframe.htm" name="iframe_a"></iframe>
<br>
<a href="//www.runoob.com" target="iframe_a">RUNOOB.COM</a>
```

# 网页结构

| 元素名  | desc                 |
| ------- | -------------------- |
| header  | 头部内容             |
| footer  | 脚部内容             |
| section | 页面中的一个独立区域 |
| article | 文章内容             |
| aside   | 侧边栏               |
| nav     | 导航栏               |

