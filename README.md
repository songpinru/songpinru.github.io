# SongPinru 知识库

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

原有内容目录保持不变：

- `notes/`：工作笔记
- `study/`：学习记录
- `源码编译/`：框架源码编译记录
- `docs/`：文档资料

Markdown 文件会被 Hugo 挂载为内容页面，图片等非 Markdown 文件会保留在 `static/` 中。文章内原有的相对图片路径可以继续使用。

## 构建

```bash
hugo --gc --minify
```

生成的站点位于 `public/`，该目录与 `resources/`、`.hugo_build.lock` 已加入 `.gitignore`。

## GitHub Pages

仓库使用 `.github/workflows/hugo.yml` 自动构建并部署。推送 `main` 分支后，GitHub Actions 会安装 Hugo、下载 Hextra 模块、生成站点并发布到 Pages。

主题版本记录在 `go.mod`：

```text
github.com/imfing/hextra v0.12.3
```

本地遇到模块下载问题时，可以临时使用本机代理；不要将代理配置提交到仓库。
