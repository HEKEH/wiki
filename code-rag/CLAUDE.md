# Code RAG Knowledge Base

## Domain

RAG (Retrieval-Augmented Generation) systems in the context of coding — how LLMs retrieve and use codebase context, code search, embedding strategies, and tooling for code-aware AI.

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

### Categories

Each category maps to a subdirectory under `wiki/`:

- **systems** (`wiki/systems/`): RAG systems, frameworks, and products (e.g. Sourcegraph Cody, Cursor, GitHub Copilot)
- **techniques** (`wiki/techniques/`): Retrieval methods, embedding strategies, chunking approaches, re-ranking
- **concepts** (`wiki/concepts/`): Core ideas — context windows, retrieval quality, hallucination, grounding
- **sources** (`wiki/sources/`): Summaries of raw source documents (papers, articles, docs)
- **analysis** (`wiki/analysis/`): Comparisons, synthesis, and exploratory queries

### Special Pages

- `wiki/home.md` — 顶层综合与导航，核心洞见和开放问题
- `wiki/index.md` — 内容目录
- `wiki/log.md` — 活动日志
