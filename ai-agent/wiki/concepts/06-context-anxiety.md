---
title: Context Anxiety
tags: [model-behavior, claude, harness-design]
date: 2026-04-23
sources: [Scaling-Managed-Agents-Decoupling.md, Effective-harnesses-for-long.md]
status: active
---

# Context Anxiety

## Definition

[[entities/03-claude|Claude]] 在感知到 context window 即将耗尽时提前收工的行为模式。该术语由 [[entities/01-anthropic|Anthropic]] 团队在 [Effective Harnesses for Long-Running Agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) 中提出。

## 具体表现

- Claude Sonnet 4.5 会随着 context 限制临近而加速完成任务，可能牺牲质量
- 行为由 model 对自身 context window 大小的感知驱动
- 在长周期 Agent 任务中尤为明显

## Harness 应对

初始应对：harness 添加 context reset（上下文重置）机制，在 context window 耗尽前主动重置。

## 假设过时

在 Claude Opus 4.5 上，context anxiety 行为消失。为 Sonnet 4.5 添加的 context reset 逻辑变成死代码。这是 [[concepts/02-harness|Harness 假设会过时]]的典型案例，也印证了 [[concepts/08-bitter-lesson]]：弥补模型缺陷的特定设计会随模型升级而失效。

## 教训

> Harness 编码了对模型能力的假设，这些假设需要被持续质疑，因为它们会随模型改进而过时。

## 与 Long-Running Agent 的关系

Context anxiety 是 [[concepts/10-long-running-agent]] 面临的深层挑战之一。即使模型没有明显的 anxiety 行为，context window 有限这一根本限制仍然存在——[[concepts/11-initializer-coding-agent]] 通过结构化交接（progress file + feature list + git history）来缓解，而非试图让 Agent 在单个 context window 内完成所有工作。

## 相关概念

- [[concepts/02-harness]] — 为弥补 context anxiety 而添加逻辑的地方
- [[concepts/09-context-engineering]] — 管理 context window 的更广泛实践
- [[concepts/08-bitter-lesson]] — 通用方法胜过特定设计的哲学
- [[concepts/01-session]] — context window 之外的持久化存储
- [[concepts/10-long-running-agent]] — context window 限制的核心受害者
- [[concepts/11-initializer-coding-agent]] — 应对 context 限制的工程策略
