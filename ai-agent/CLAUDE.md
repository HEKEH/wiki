# AI Agent Knowledge Base

## Domain

Agent 架构设计 — 如何构建可扩展、可演进的 AI Agent 系统。当前核心主题围绕 Anthropic 的 Managed Agents 解耦架构与 Agentic Systems 分类框架展开。

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

- **entities** (`wiki/entities/`): 公司、产品、人物等实体
- **concepts** (`wiki/concepts/`): 核心概念、设计模式、架构思想
- **sources** (`wiki/sources/`): 原始文档的摘要页面
- **analysis** (`wiki/analysis/`): 对比分析、综合探索
- **topics** (`wiki/topics/`): 跨分类的主题聚合页面

### Special Pages

- `wiki/home.md` — 顶层综合与导航，核心洞见和开放问题
- `wiki/index.md` — 内容目录
- `wiki/log.md` — 活动日志
