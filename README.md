# 个人主页

用于存放自己写的学习笔记以及工作经验，也可以当作个人博客，或者云笔记。

主要用于：
* 工作自查
* 事故分析记录
* 同事朋友间的分享

## github pages

白嫖github服务器资源，使用github pages搭建一个自己的云笔记

1. 创建一个{username}.github.io 仓库（普通仓库也可以，但是用自己的名字访问的时候更简单）
2. setting中开启pages
3. 选择jekyll pages action
4. 提交

## jekyll

使用jekyll + github page + github action，实现提交即发布的个人云笔记。

github page支持多个jekyll主题（不是全部都支持，有很多特性还是缺失的）。

从GitHub官方文档找了一个极简风的主题：slate
> 黑白配极简风，体验max

在根目录增加一个`_config.yml`配置文件
```yaml
# 站点设置
theme: jekyll-theme-slate
title: "个人笔记"
description: SongPinru 的小仓库
show_downloads: false

github:
  is_project_page: true

# 构建选项
exclude: ["README.md", "LICENSE.md", "CNAME", "vendor","test"] # 在构建时排除的文件和目录
timezone: "Asia/Shanghai" # 网站时区
```

此时基本功能全了，提交markdown格式的笔记已经可以访问了。

## 优化
使用起来还是不够方便，缺点东西：
* 没有目录，看哪个文档全凭输入url
* 输错url跑到github的404页面了，很不和谐
* 文档内容没有索引

### 主页与目录

使用目录的url访问的是目录下的`index.html`,也就是我们的`index.md`。

所以我们在根目录和每个子目录下新建一个`index.md`文件，里面写上当前目录的文档链接，就拥有一个基本的目录功能了。

```markdown
# Index

我的Pages，存放自己的文档


# 目录

* [notes](./notes)
* [study](./study)
* [docs](./docs)
```

如上。

### 404

利用jekyll的permalink特性，让404自动转到根目录下的`404.html`，所以我们需要一个`404.md`文件

```markdown
---
permalink: /404.html
---

## 404 Not Found

页面丢失，请检查地址。
```

### 文档内索引

有其他jekyll主题支持toc，但是github基本不支持，我们只能使用纯js的方式实现。

github上有人写过了：[Toc Github](https://github.com/songpinru/github-slideshow)，非常感谢！ 

1. 把`toc.js`放入我们的仓库
2. 根据[slate 主题](https://github.com/pages-themes/slate)的文档，新增几个文件:
   * `_layouts/default.html`
       * 自定义的文档结构
   * `assets/css/style.scss`
       * 自定义的样式
3. 修改`_layouts/default.html`,引入我们自己的布局和js代码

**布局**

`_layouts/default.html` 中的header增加js：
```html
<script src="https://apps.bdimg.com/libs/jquery/2.1.4/jquery.min.js"></script>
<script src="/javascript/toc.js" type="text/javascript"></script>
```
body增加布局，侧边栏，script
```html
<body>
<div id="layout">
    <!-- SIDEBAR/NAVIGATION -->
    <div id="sidebar" >
        <aside id="toc" ></aside>
    </div>
    <button id="sidebar_hide"> > </button>
    <div id="main_wrap">
        <!-- HEADER -->
        <div id="header_wrap" class="outer">
            <header class="inner">
                {% if site.github.is_project_page %}
                <a id="forkme_banner" href="{{ site.github.repository_url }}">View on GitHub</a>
                {% endif %}

                <p id="project_title">{{ site.title | default: site.github.repository_name }}</p>
                <p id="project_tagline">{{ site.description | default: site.github.project_tagline }}</p>

                {% if site.show_downloads %}
                <section id="downloads">
                    <a class="zip_download_link" href="{{ site.github.zip_url }}">Download this project as a .zip file</a>
                    <a class="tar_download_link" href="{{ site.github.tar_url }}">Download this project as a tar.gz file</a>
                </section>
                {% endif %}
            </header>
        </div>

        <!-- MAIN CONTENT -->
        <div id="main_content_wrap" class="outer">
            <section id="main_content" class="inner">
                {{ content }}
            </section>
        </div>

        <!-- FOOTER  -->
        <div id="footer_wrap" class="outer">
            <footer class="inner">
                {% if site.github.is_project_page %}
                <p class="copyright">{{ site.title | default: site.github.repository_name }} maintained by <a
                        href="{{ site.github.owner_url }}">{{ site.github.owner_name }}</a></p>
                {% endif %}
                <p>Published with <a href="https://pages.github.com">GitHub Pages</a></p>
            </footer>
        </div>
    </div>
</div>
<script type="text/javascript">
    $(document).ready(function () {
        $('#toc').toc({ listType: 'ul' });
        var isHide=true;
        $("#sidebar_hide").click(function (){
            // let sidebar_width = $("#sidebar").width();
            if(isHide){
                $(this).text("<");
                // $(this).css("left","+="+sidebar_width);
                // $("#main_wrap").css("padding-left","+="+sidebar_width);
                $("#sidebar").toggle()
                isHide=false;
            }else{
                $(this).text(">");
                // $(this).css("left",0);
                // $("#main_wrap").css("padding-left","-="+sidebar_width);
                $("#sidebar").toggle()
                isHide=true;
            }
        });
    });
</script>
</body>
```

**样式**

调整侧边栏的和布局的样式：

```css
#layout {
  max-width: 100vw;
  height: 100vh; /* 设置父元素的高度 */
  display: flex;
}

#sidebar {
  max-width: 300px;
  width: 100%;
  height: 100%;
  background-color: #dfdfdf;
  display:none;
  overflow-y: auto;
}
#sidebar_hide{
  height: 100%;
  width: 20px;
  background-color: #dfdfdf;
  border: 1px solid #dfdfdf;
}

#sidebar_hide:active{
  opacity: 0.5;
  border: 1px;
}

#toc {
  width: 100%;
  height: 100%;
}

#main_wrap{
  display: flex;
  flex-direction: column;
  width:100%;
  height:100%;
  overflow: auto;
  background-color: #f2f2f2;
}
#main_content_wrap{
  flex: 1;
}
```

完成！



