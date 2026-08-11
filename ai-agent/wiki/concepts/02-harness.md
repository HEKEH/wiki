---
title: Harness
tags: [agent-architecture, orchestration, loop]
date: 2026-04-23
sources: [Scaling-Managed-Agents-Decoupling.md, Effective-harnesses-for-long.md]
status: active
---

# Harness

## Definition

Agent 的"大脑"——调用 Claude 并将其工具调用路由到相关基础设施的循环。Harness 是无状态的、可替换的 cattle，崩溃后可通过 [[concepts/01-session]] 重建。

## 核心职责

1. **Agent 循环** — 调用 Claude API → 接收响应 → 路由工具调用 → 返回结果 → 重复
2. **上下文工程** — 决定将 [[concepts/01-session]] 中的哪些事件放入 Claude 的 context window，包括 compaction、trimming、缓存优化
3. **事件持久化** — 在循环过程中通过 `emitEvent()` 写入 session
4. **故障恢复** — 崩溃后新 harness 通过 `wake(sessionId)` + `getSession()` 恢复

## Harness 假设会过时

Harness 编码了对模型能力的假设（如"[Claude 不能自己处理上下文]"），但随着模型升级，这些假设会过时（go stale）。例如：

- Claude Sonnet 4.5 存在 [[concepts/06-context-anxiety]]，harness 需要添加上下文重置
- Claude Opus 4.5 消除了该行为，重置逻辑变为死代码

这符合 [[concepts/08-bitter-lesson]] 的洞见：利用模型能力的通用方法最终胜过弥补模型缺陷的特定设计。

## Harness 离开容器

原始设计中 harness 在容器内运行，导致：
- 容器 = 宠物（不可丢失）
- 调试困难（容器含用户数据）
- 无法连接外部 VPC

解耦后，harness 通过 `execute(name, input) → string` 调用容器，与调用其他工具无异。

## Harness 的长时运行设计

[[sources/02-effective-harnesses-for-long-running-agents]] 揭示了 harness 在长时运行场景下的具体设计模式：

- **Initializer/Coding Agent 分工** — 在同一 harness 内通过不同 prompt 实现角色切换，第一个 session 搭建环境，后续 session 增量推进
- **标准化 session 启动流程** — pwd → 读 progress → 读 feature list → git log → 冒烟测试 → 开始工作
- **结构化交接** — progress file + git history + feature list 构成跨 session 的三重上下文

这与 [[concepts/07-meta-harness]] 不同——meta-harness 容纳不同 harness 实现，而 Initializer/Coding 模式在同一 harness 内切换 prompt。

## 与 Meta-harness 的关系

[[concepts/07-meta-harness]] 是容纳多种 harness 实现的系统。Managed Agents 不限定特定 harness——Claude Code 是一种 harness，任务特定的窄域 harness 也可以。Meta-harness 对接口有主见，对 harness 实现无主见。

## 相关概念

- [[concepts/01-session]] — harness 的状态来源和事件持久化目标
- [[concepts/03-sandbox]] — harness 通过工具调用操作的执行环境
- [[concepts/09-context-engineering]] — harness 的核心关注点
- [[concepts/07-meta-harness]] — harness 的容器系统
- [[concepts/06-context-anxiety]] — 驱动 harness 设计的模型行为
- [[concepts/10-long-running-agent]] — harness 必须跨 context window 工作的场景
- [[concepts/11-initializer-coding-agent]] — harness 在长时运行中的具体设计
- [[concepts/12-feature-list-pattern]] — harness 内的结构化需求追踪机制

## Harness 与 Agentic Systems 分类

从 [[concepts/13-agentic-systems]] 的角度看，Harness 是 Workflow 和 Agent 的**运行时实现**：

- Harness 的 Agent 循环（调用 LLM → 路由工具 → 返回结果 → 重复）本质上是 [[concepts/14-augmented-llm]] 的循环实例化
- Harness 中预定义的工具调用路径对应 [[concepts/13-agentic-systems|Workflows]]，如 [[concepts/15-prompt-chaining]] 和 [[concepts/16-routing]]
- 当 Harness 允许 LLM 自主决定下一步操作时，即为 [[concepts/13-agentic-systems|Agents]] 模式
- [[concepts/20-aci]] 原则应指导 Harness 中工具的路由和接口设计
