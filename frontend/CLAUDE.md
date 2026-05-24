# Frontend Knowledge Base

## Domain

Frontend Engineering — 前端工程领域的技术生态，涵盖包管理、注册表体验、开发工具链、Web 平台特性、可访问性、国际化等。

## Conventions

### Page Format

Every wiki page should have YAML frontmatter:

```yaml
---
title: Page Title
date: YYYY-MM-DD
tags: [tag1, tag2]
sources: [source-file-1.md]
---
```

### Wikilinks

Use `[[category/page-name]]` for cross-references (include the subdirectory path under `wiki/`).

### Categories

Each category maps to a subdirectory under `wiki/`:

- **concepts** (`wiki/concepts/`): 前端核心概念 — 包管理、依赖安全、可访问性、国际化、模块系统等
- **entities** (`wiki/entities/`): 工具、项目、组织 — npmx、bundlephobia、deps.dev、JSR 等
- **sources** (`wiki/sources/`): 原始文档摘要
- **analysis** (`wiki/analysis/`): 对比分析、综合洞察

### Special Pages

- `wiki/home.md` — 顶层综合与导航，核心洞见和开放问题
- `wiki/index.md` — 内容目录
- `wiki/log.md` — 活动日志
