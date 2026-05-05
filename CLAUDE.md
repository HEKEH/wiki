# LLM Wiki Collection

This directory is a **collection of independent knowledge bases**, each maintained by an LLM agent following the pattern described in `llm-wiki.md`.

## Directory Convention

```
wiki/                          ← you are here (collection root)
├── CLAUDE.md                  ← this file (collection-level schema)
├── llm-wiki.md                ← the pattern document
├── _template/                 ← template for creating new knowledge bases
├── code-rag/                  ← an independent knowledge base
│   ├── CLAUDE.md              ← domain-specific schema (overrides this file)
│   ├── raw/                   ← immutable source documents
│   └── wiki/                  ← LLM-generated markdown pages
│       ├── home.md            ← top-level synthesis and navigation
│       ├── index.md           ← content-oriented catalog
│       ├── log.md             ← chronological activity log
│       ├── systems/           ← RAG systems, frameworks, products
│       ├── techniques/        ← retrieval methods, chunking, embeddings
│       ├── concepts/          ← core ideas and mental models
│       ├── sources/           ← summaries of raw source documents
│       └── analysis/          ← comparisons, synthesis, queries
├── <another-kb>/              ← another independent knowledge base
│   └── ...
```

## General Rules

These rules apply to **all** knowledge bases in this collection unless overridden by a knowledge base's own `CLAUDE.md`.

### Raw Sources

- `raw/` is immutable. Never modify, move, or delete source files.
- When ingesting, copy the source into `raw/` first, then read from there.

### Wiki Layer

- `wiki/` is owned by the LLM. Create, update, and delete pages as needed.
- Wiki pages are organized into **category subdirectories** (e.g. `wiki/concepts/`, `wiki/sources/`). Each knowledge base defines its own categories in its `CLAUDE.md`.
- Every page should have a YAML frontmatter block with at least `title`, `date`, `tags`, and `sources`.
- Use `[[category/page-name]]` for cross-references between wiki pages (include the subdirectory path).
- When a single source touches multiple topics, update all relevant pages across categories in one pass.

### Index, Log, and Home

- `wiki/home.md` — top-level synthesis, core insights, and open questions. Updated as the wiki evolves.
- `wiki/index.md` is updated on every ingest. It lists every page with a link and a one-line summary, organized by category.
- `wiki/log.md` is append-only. Each entry follows the format:
  ```
  ## [YYYY-MM-DD] <operation> | <subject>
  ```
  Operations: `ingest`, `query`, `lint`.

### Lint

- When asked to lint, check for: contradictions, stale claims, orphan pages, missing cross-references, concepts without pages, data gaps.
- Propose fixes and new sources to investigate.

### Creating a New Knowledge Base

1. Copy `_template/` to a new directory named after the knowledge base.
2. Replace all `{{KB_NAME}}` placeholders in the new `CLAUDE.md`, `wiki/home.md`, `wiki/index.md`, and `wiki/log.md`.
3. Customize the new `CLAUDE.md` for the domain (define categories, add conventions).
4. Start ingesting sources.

## Operations

### Ingest

1. Place source in `raw/`.
2. Read the source and discuss key takeaways with the user.
3. Write/update the summary page in `wiki/` (place in the appropriate category subdirectory).
4. Update all relevant pages across the wiki (may span multiple categories).
5. Update `wiki/index.md`.
6. Append an entry to `wiki/log.md`.

### Query

1. Read `wiki/index.md` to find relevant pages.
2. Drill into specific pages as needed.
3. Synthesize an answer with citations.
4. If the answer is broadly useful, file it as a new wiki page.

### Lint

1. Scan `wiki/index.md` and all wiki pages.
2. Report: contradictions, stale claims, orphans, missing pages, data gaps.
3. Propose fixes and new sources.
