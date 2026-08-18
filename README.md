# Pinru's Home

个人技术笔记、故障复盘与学习记录。站点使用 [Hugo](https://gohugo.io/) 与 [Hextra](https://imfing.github.io/hextra/) 主题。

## 内容目录

```
content/
├── blog/     # 博客文章
├── docs/     # 技术文档
└── _index.md
```

## 本地预览

依赖：Hugo Extended 0.146.0+、Go 1.26+

```bash
hugo server
```

浏览器打开 <http://localhost:1313/>。

## 构建

```bash
hugo --gc --minify --baseURL "https://songpinru.github.io/"
```

推送 `main` 分支自动部署，详见 [AGENTS.md](AGENTS.md)。

主题版本记录在 `go.mod` 中。
