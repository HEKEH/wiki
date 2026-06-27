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

Use `[[category/page-name]]` for cross-references (include the subdirectory path under `wiki/` and the sequence number prefix, e.g. `[[concepts/02-xxx]]`).

### File Numbering

Files within each `wiki/` subdirectory are prefixed with a two-digit sequence number reflecting the order in which they were added, starting from `01`. For example, `wiki/concepts/` may contain `01-xxx.md`, `02-xxx.md`, …, `10-xxx.md`.

The number records the order in which the knowledge base grew, making it easy to see how it developed and to review pages in the order they were learned.

- When adding a new page, use the next number after the current highest in that directory.
- When deleting a page, leave its number as a gap and never renumber existing pages, so cross-references stay valid.

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
