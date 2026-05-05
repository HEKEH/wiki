# Claude

tags: [product, llm, anthropic]
date: 2026-04-23
status: stub

## Overview

[Anthropic](Anthropic.md) 开发的大语言模型系列，是 [Managed Agents](Managed-Agents.md) 的推理引擎。

## 模型版本

- **Opus** — 最强智能，适合复杂推理任务
- **Sonnet** — 平衡性能与成本
- **Haiku** — 速度优先

## 行为特征

- **[Context Anxiety](../concepts/context-anxiety.md)** — Sonnet 4.5 在感知 context window 即将耗尽时提前收工；Opus 4.5 已消除此行为
- 随模型升级，harness 中为弥补模型缺陷而添加的逻辑可能变为死代码

## 相关概念

- [Harness](../concepts/harness.md)
- [Context Engineering](../concepts/context-engineering.md)
- [Session](../concepts/session.md)
