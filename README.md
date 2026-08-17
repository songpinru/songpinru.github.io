# Pinru's Home

个人技术笔记、故障复盘与学习记录。站点使用 [Hugo](https://gohugo.io/) 和 [Hextra](https://imfing.github.io/hextra/) 构建，部署到 GitHub Pages。

## 本地预览

需要先安装：

- Hugo Extended 0.146.0 或更高版本
- Go 1.26 或兼容版本

在仓库根目录执行：

```bash
hugo server
```

浏览器打开终端提示的地址，通常为 <http://localhost:1313/>。

## 内容目录

内容采用 Hextra 原版结构：

- `content/docs/notes/`：工作笔记
- `content/docs/study/`：学习记录
- `content/docs/源码编译/`：框架源码编译记录
- `content/docs/JavaSE葵花宝典.md`、`content/docs/MySQL九阴真经.md`：参考文档
- `content/blog/`：博客，当前为空

Markdown 与对应图片资源放在同一 `content/docs/` 目录下。`static/` 仅用于全站级静态文件，目前不需要维护。

## 构建

```bash
hugo --minify --cleanDestinationDir
```

生成的站点位于 `public/`，该目录与 `resources/`、`.hugo_build.lock` 已加入 `.gitignore`。

## GitHub Pages

仓库使用 `.github/workflows/hugo.yml` 自动构建并部署。推送 `main` 分支后，GitHub Actions 会安装 Hugo、下载 Hextra 模块、生成站点并发布到 Pages。

主题版本记录在 `go.mod`：

```text
github.com/imfing/hextra v0.12.3
```

本地遇到模块下载问题时，可以临时使用本机代理；不要将代理配置提交到仓库。
