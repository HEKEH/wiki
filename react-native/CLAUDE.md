# React Native Knowledge Base

## Domain

React Native — 使用 JavaScript/React 构建 iOS 与 Android 原生应用的跨平台框架。系统覆盖:核心组件与 API、新架构(Fabric / TurboModules / JSI / Codegen / Yoga)、渲染管线与线程模型、导航、状态管理、性能优化、原生模块开发、调试与测试、生态工具链(Expo、Hermes、FlashList 等)。

目标是搭建一条从入门到进阶的**成体系**学习主线,配以官方文档与权威团队(Callstack、Meta)的一手资料。

## Conventions

以下约定覆盖集合级 `CLAUDE.md`。

### Page Format

每个内容页需带 YAML frontmatter:

```yaml
---
title: Page Title
date: YYYY-MM-DD
tags: [tag1, tag2]
sources: [source-file-1.md]
---
```

导航页 `home.md`、`index.md`、`log.md` 免除 frontmatter。

### Wikilinks

使用 `[[category/page-name]]` 交叉引用,包含子目录路径与序号前缀,例如 `[[concepts/01-new-architecture]]`。

### File Numbering

每个 `wiki/` 子目录内文件以两位序号前缀,反映加入顺序,从 `01` 起。新增取当前最大序号 +1;删除留空号、不重排,以保证交叉引用稳定。

### Categories

- **concepts** (`wiki/concepts/`): 核心概念与机制 — 新架构、Fabric、TurboModules、JSI、渲染管线、线程模型、性能、导航、状态管理等
- **entities** (`wiki/entities/`): 工具、库、组织 — React Native、Hermes、Yoga、Expo、FlashList、Callstack、Meta 等
- **sources** (`wiki/sources/`): 原始文档/文章摘要(对应 `raw/` 中的抓取)
- **analysis** (`wiki/analysis/`): 对比分析、版本演进、综合洞察

### Raw Sources

`raw/` 不可变。抓取的网页/文章以 markdown 形式存档于 `raw/`(文件名含来源与日期),wiki 层从存档二次组织。

### Special Pages

- `wiki/home.md` — 顶层综合与导航,核心洞见和开放问题
- `wiki/index.md` — 内容目录
- `wiki/log.md` — 活动日志
