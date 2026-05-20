# DEX Knowledge Base

## Domain

Decentralized Exchange (Crypto) — 去中心化交易所的技术生态，涵盖链上交易协议、智能合约开发、前端交互库、流动性机制等。

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

- **concepts** (`wiki/concepts/`): DEX 核心概念 — AMM、订单簿、流动性、滑点、无常损失等
- **libraries** (`wiki/libraries/`): 开发库与工具 — ethers.js、viem、wagmi、web3.js 等
- **entities** (`wiki/entities/`): 协议、项目、组织 — Uniswap、Curve、0x 等
- **sources** (`wiki/sources/`): 原始文档摘要
- **analysis** (`wiki/analysis/`): 对比分析、综合洞察

### Special Pages

- `wiki/home.md` — 顶层综合与导航，核心洞见和开放问题
- `wiki/index.md` — 内容目录
- `wiki/log.md` — 活动日志
