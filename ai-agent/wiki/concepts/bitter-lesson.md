# The Bitter Lesson

tags: [philosophy, ml, scaling, richard-sutton]
date: 2026-04-23
sources: [../../raw/Scaling-Managed-Agents-Decoupling.md]
status: active

## Definition

Richard Sutton 在 2019 年提出的观点：在 AI 研究史上，利用算力的通用方法最终胜过人类设计的特定方法。[原文](http://www.incompleteideas.net/IncIdeas/BitterLesson.html)

## 核心论点

> 70 年来，AI 研究最大的遗憾是：我们倾向于构建利用人类知识的特定方法，但每当最终用算力搜索和学习取代这些方法时，进步总是最大的。

## 在 Agent 架构中的映射

Anthropic 在 [Managed Agents](../entities/Managed-Agents.md) 的设计中引用了 The Bitter Lesson 作为哲学基础：

1. **Harness 假设会过时** — 为弥补模型缺陷（如 [Context Anxiety](context-anxiety.md)）而设计的特定逻辑，随模型升级变为死代码
2. **通用接口 > 特定实现** — [Meta-harness](meta-harness.md) 提供通用抽象，不限定特定 harness 实现
3. **不要对模型能力做假设** — 模型会持续变强，基于"模型做不到 X"的设计注定过时

## 与操作系统设计的类比

操作系统的抽象（`process`、`file`）能存活数十年，正是因为它们是通用的而非特定的。`read()` 不预设存储介质的具体类型——这种通用性正是 The Bitter Lesson 在系统设计中的体现。

## 在 Agent 模式选择中的体现

[Building Effective AI Agents](../sources/Building-Effective-AI-Agents.md) 的核心建议——"最成功的实现使用简单可组合的模式而非复杂框架"——正是 The Bitter Lesson 在 Agent 工程中的直接体现：

1. **简单模式 > 复杂框架** — 框架引入的抽象层往往掩盖底层 prompt 和响应，增加调试难度
2. **只在验证有效时添加复杂度** — 从单次 LLM 调用开始，逐步增加 Workflow 模式，而非一步到位构建完整 Agent
3. **通用可组合模式 > 专用框架** — [Prompt Chaining](prompt-chaining.md)、[Routing](routing.md) 等模式用几行代码即可实现，不需要框架

## 相关概念

- [Meta-harness](meta-harness.md) — 直接应用 The Bitter Lesson 的系统设计
- [Context Anxiety](context-anxiety.md) — 特定设计随模型升级而过时的案例
- [Harness](harness.md) — 需要持续质疑假设的组件
- [Agentic Systems 分类框架](agentic-systems.md) — 简单优先的模式选择策略
