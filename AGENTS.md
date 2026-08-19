# Repository Guidelines

## Project Structure

```
content/
├── _index.md          # Homepage
├── blog/              # Blog posts (Chinese, dated)
│   ├── _index.md
│   └── *.md           # Markdown source files
└── docs/              # Technical documentation
    ├── _index.md
    ├── engineering/   # 工程基础
    │   ├── languages/     # 编程语言 (Java, Python, Rust, Go, Shell, Markdown)
    │   ├── systems/       # 操作系统与系统维护
    │   ├── infrastructure/# 基础设施与工具 (容器、CI/CD、Git、监控、代理)
    │   ├── 数据结构与算法.md
    │   └── 源码编译通用步骤.md
    ├── databases/
    ├── data-platform/     # 数据平台 (Hadoop, Kafka, Flink, Spark, Pandas)
    └── application-development/
layouts/               # Hugo template overrides
public/                # Generated site (gitignored)
hugo.yaml              # Site configuration
.github/workflows/     # CI/CD pipeline
```

Assets are co-located with content under `content/blog/post-name.assets/` for images.

## Build & Development

```bash
# Start local dev server with live reload
hugo server

# Build for GitHub Pages
hugo --gc --minify --baseURL "https://songpinru.github.io/"

# Build for Cloudflare Pages (base URL set via env var at deploy time)
hugo --gc --minify --baseURL "$CF_PAGES_URL"
```

The site uses [Hugo](https://gohugo.io) with the [Hextra](https://github.com/imfing/hextra) theme. Hugo extended edition is required. KaTeX math rendering is enabled.

## Content Guidelines

- Write in Markdown; frontmatter requires `title` and may include `description`, `date`, `tags`.
- Blog posts go under `content/blog/`, docs under `content/docs/` with a `_index.md` for section indexes.
- File names use Chinese characters where appropriate; keep them descriptive.
- Use `![alt](path.assets/image.png)` for co-located images.
- KaTeX math is supported via `$$` block delimiters.
- **Heading structure**: Each document must have exactly one `h1` (`#`) that matches the frontmatter `title`. Use `h2` (`##`) and below for section headings within the document. This ensures the Hextra theme's table of contents renders correctly — `h1` serves as the document title and is not displayed in the TOC; `h2+` chapters appear in the TOC sidebar.

## Commit Style

Messages follow conventional-commits style observed in history:

```
<type>(<scope>): <description>
```

Types: `feat`, `docs`, `chore`, `refactor`, `fix`. Use `docs` for documentation additions, `feat` for new content sections.

## CI/CD & Deployment

The project deploys to two platforms:

- **GitHub Pages** -- via `.github/workflows/hugo.yml` (triggered on push to `main`). Uses `actions/deploy-pages@v4`.
- **Cloudflare Pages** -- connected directly to the GitHub repo. Build command: `hugo --gc --minify --baseURL "$CF_PAGES_URL"`. No retention limit on deployments by default; old deployments must be cleaned manually or via a scheduled script.

## Agent Instructions

When making edits to existing files, read the full file first. Do not modify files outside the scope of the task. Prefer existing patterns (frontmatter, directory layout, image co-location) over introducing new conventions.
