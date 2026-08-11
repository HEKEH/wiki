# AI Agent Knowledge Base

## Domain

Agent 架构设计 —— 如何构建可扩展、可演进的 AI Agent 系统。当前主干由 Anthropic 三篇工程博客构成：Managed Agents 的 Brain-Hands-Session 解耦架构、长时运行 harness 的 Initializer/Coding Agent 模式、以及 Agentic Systems 分类框架（Workflows vs Agents 与五种编排模式）。

## Conventions

以下约定覆盖集合级 `CLAUDE.md`。

### Page Format

每个内容页需带 YAML frontmatter：

```yaml
---
title: Page Title
date: YYYY-MM-DD
tags: [tag1, tag2]
sources: [source-file-1.md]
status: stub | active | ingested
---
```

`sources` 填 `raw/` 下的原始文件名（不带路径）。`status` 可选。导航页 `home.md`、`index.md`、`log.md` 免除 frontmatter。

### Wikilinks

使用 `[[category/page-name]]` 交叉引用，包含子目录路径与序号前缀，例如 `[[concepts/02-harness]]`。链接文字与页面标题不一致时用别名形式 `[[concepts/03-sandbox|执行计算]]`。

### File Numbering

每个 `wiki/` 子目录内文件以两位序号前缀，反映加入顺序，从 `01` 起。新增取当前最大序号 +1；删除留空号、不重排，以保证交叉引用稳定。

### Categories

- **concepts** (`wiki/concepts/`): 核心概念、设计模式、架构思想
- **entities** (`wiki/entities/`): 公司、产品、人物等实体
- **sources** (`wiki/sources/`): 原始文档的摘要页面
- **analysis** (`wiki/analysis/`): 对比分析、综合探索

### Raw Sources

`raw/` 不可变。抓取的网页/文章以 markdown 存档于 `raw/`，配图放 `raw/assets/<source-slug>/`；wiki 层从存档二次组织。

### Special Pages

- `wiki/home.md` — 顶层综合与导航，核心洞见和开放问题
- `wiki/index.md` — 内容目录
- `wiki/log.md` — 活动日志
