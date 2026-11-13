# 仓库规范

## 项目结构

```
content/
├── _index.md          # 首页
├── blog/              # 全部文章（中文，按日期归档）
│   ├── _index.md
│   ├── *.md           # Markdown 源文件
│   └── *.assets/      # 与文章同目录的图片资源
public/                # 生成站点（已加入 gitignore）
hugo.yaml              # 站点配置
.github/workflows/     # CI/CD 流水线
```

图片等资源与内容一同存放，路径为 `content/blog/post-name.assets/`。

## 构建与开发

```bash
# 启动本地开发服务器并启用热重载
hugo server

# 为 GitHub Pages 构建
hugo --gc --minify --baseURL "https://songpinru.github.io/"

# 为 Cloudflare Pages 构建（部署时通过环境变量设置 baseURL）
hugo --gc --minify --baseURL "$CF_PAGES_URL"
```

本站使用 [Hugo](https://gohugo.io) 与 [Hextra](https://github.com/imfing/hextra) 主题，需要安装 Hugo 扩展版。站点已启用 KaTeX 数学公式渲染。文章列表通过 `content/blog/_index.md` 中的内联 shortcode 实现，侧边栏支持分类和标签多选筛选，标签及其计数随已选分类动态更新；不新增 `layouts/` 覆盖，也不单独生成分类、标签归档页。

## 内容规范

- 内容使用 Markdown 编写；frontmatter 必须包含 `title`、`date`、`categories` 和 `tags`，可选字段包括 `description`。
- 所有正文统一放在 `content/blog/` 根目录，不再按技术主题嵌套目录；图片等资源与文章一同移动并保持相对路径不变。
- 分类只表示写作目的，不表示技术领域。每篇只设置一个分类：`essays`（随笔）、`incident-review`（故障复盘）、`operations-guide`（运维手册）、`tech-sharing`（技术分享）、`design-research`（设计调研）、`learning-notes`（学习记录，包括自己的学习实践、知识归纳和外部教程资料整理）、`other`（其他）。新增文章优先归入现有分类，不要为技术、内容来源或单篇文章新建分类。
- 标签只表示涉及的技术、组件、机制或技术主题，每篇设置 3～5 个，如 `kafka`、`kraft`、`性能优化`。不要使用“故障排查”“运维”“方案设计”“技术调研”“入门教程”“速查表”等与目的分类重复的标签。主要技术标签应能跨分类聚合同一技术的全部文章；使用受控、可复用的标签，避免过细和同义标签。
- 文件名在合适时使用中文，并保持描述性。
- 与内容同目录存放的图片使用 `![alt](path.assets/image.png)`。
- KaTeX 数学公式使用 `$$` 块级分隔符。
- **标题结构**：文档正文不要写 `h1`（`#`），页面标题由 frontmatter `title` 自动渲染。文档内章节从 `h2`（`##`）开始，以保证 Hextra 主题目录渲染正确：`h1` 作为文档标题且不显示在目录中，`h2+` 章节会显示在目录侧边栏。

## 提交规范

提交信息遵循仓库历史中使用的 conventional commits 风格：

```
<type>(<scope>): <description>
```

类型包括 `feat`、`docs`、`chore`、`refactor`、`fix`。新增文档使用 `docs`，新增内容栏目使用 `feat`。

## CI/CD 与部署

项目部署到两个平台：

- **GitHub Pages** -- 通过 `.github/workflows/hugo.yml` 触发，推送到 `main` 分支时执行，并使用 `actions/deploy-pages@v4`。
- **Cloudflare Pages** -- 直接连接 GitHub 仓库。构建命令为 `hugo --gc --minify --baseURL "$CF_PAGES_URL"`。默认不对部署数量设置保留上限，旧部署需要手动清理或通过定时脚本清理。

## 代理规范

- 修改现有文件前，先完整读取文件内容。不要修改任务范围之外的文件。优先沿用现有模式，包括 frontmatter、目录结构和图片同目录存放，而不是引入新的约定。
- 尽量不要修改 `layouts/`、CSS 等 Hugo 与主题相关文件；优先通过修改 `hugo.yaml` 参数这类官方支持的配置方案处理。确实需要特殊处理时，先征求用户同意后再修改。
