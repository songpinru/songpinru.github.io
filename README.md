# Pinru's Home

个人技术笔记、故障复盘与学习记录。站点使用 [Hugo](https://gohugo.io/) 与 [Hextra](https://imfing.github.io/hextra/) 主题。

## 内容目录

```text
content/
├── blog/        # 全部文章及同目录图片资源
└── _index.md    # 首页
```

所有正文统一放在 `content/blog/`。文章列表位于 `/blog/`，侧边栏支持分类和标签多选筛选，标签及其计数会随已选分类动态更新。筛选条件保存在 URL 的 `category` 和 `tag` 参数中；不再单独生成分类、标签归档页。

分类表示写作目的，每篇从 7 个固定分类中选择一个：`essays`（随笔）、`incident-review`（故障复盘）、`operations-guide`（运维手册）、`tech-sharing`（技术分享）、`design-research`（设计调研）、`learning-notes`（学习记录，包括自己的学习实践、知识归纳和外部教程资料整理）、`other`（其他）。标签表示涉及的技术、组件、机制或技术主题，每篇设置 3～5 个，避免与分类重复或创建过细、同义标签。

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
