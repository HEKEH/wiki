# SRE Knowledge Base

## Domain

站点可靠性工程（Site Reliability Engineering）相关知识库，覆盖容器与编排（Kubernetes、Docker）、分布式系统、运维自动化、监控告警、可观测性、故障处理等主题。内容以中文为主。

## Conventions

<!-- Domain-specific conventions go here. These override the collection-level CLAUDE.md. -->

- 页面语言以中文为主，保留技术术语英文原文（如 Pod、apiserver）。

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

- **entities** (`wiki/entities/`): 系统、产品、工具、组件（如 Kubernetes、etcd、kube-proxy）
- **concepts** (`wiki/concepts/`): 核心理念与机制（如 容器编排、声明式 API、服务发现）
- **sources** (`wiki/sources/`): 原始文档摘要
- **analysis** (`wiki/analysis/`): 对比、综合与查询结果

### Special Pages

- `wiki/home.md` — 顶层综合与导航，核心洞见和开放问题
- `wiki/index.md` — 内容目录
- `wiki/log.md` — 活动日志
