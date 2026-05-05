# {{KB_NAME}} Knowledge Base

## Domain

<!-- Describe the domain/topic of this knowledge base. -->

## Conventions

<!-- Domain-specific conventions go here. These override the collection-level CLAUDE.md. -->

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

Each category maps to a subdirectory under `wiki/`. Define your categories here:

<!-- - **entities** (`wiki/entities/`): People, organizations, products -->
<!-- - **concepts** (`wiki/concepts/`): Key ideas, frameworks, theories -->
<!-- - **sources** (`wiki/sources/`): Summaries of raw source documents -->
<!-- - **analysis** (`wiki/analysis/`): Comparisons, synthesis, queries -->

### Special Pages

- `wiki/home.md` — 顶层综合与导航，核心洞见和开放问题
- `wiki/index.md` — 内容目录
- `wiki/log.md` — 活动日志
