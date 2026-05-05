# Long-Running Agent

tags: [agent-architecture, long-running, context-window]
date: 2026-04-24
sources: [../../raw/Effective-harnesses-for-long.md]
status: active

## Definition

需要跨越多个 context window、持续数小时甚至数天工作的 Agent。由于 context window 有限，而大多数复杂项目无法在单个窗口内完成，Agent 必须找到跨 session 衔接状态的方法。

## 核心挑战

Long-running Agent 的根本困难在于：**每个新 session 都从零记忆开始**。想象一个软件项目由轮班工程师负责，但每位新工程师对上一班的工作毫无记忆。

### 两种经典失败模式

1. **One-shotting** — Agent 试图一次性完成所有工作，context 耗尽时留下半成品代码，下一个 session 无法理解之前的状态。即使有 compaction，传递给下一个 agent 的指令也不总是足够清晰。

2. **Premature Victory** — 后续 session 的 Agent 看到已有进展，直接宣布项目完成。

## 为什么 Compaction 不够

Compaction（上下文压缩）能让 Agent 在单个 context window 内工作更久，但不能解决：
- 跨 context window 的状态丢失
- 压缩后传递的指令不够清晰
- Agent 对"已完成"的判断标准不一致

## 解决方向

问题分解为两部分：

1. **初始环境搭建** — 建立所有后续 session 需要的上下文基础设施
2. **增量推进 + 干净状态** — 每个 session 做一小步，结束时留下可合并的代码

具体模式见 [Initializer/Coding Agent 模式](initializer-coding-agent.md)。

## 类比：轮班工程师

> 每个新工程师到达时对上一班工作毫无记忆——但如果有详细的交接文档、清晰的待办列表、可运行的项目状态，他们就能高效接手。

这正是 Feature List、progress file、git history 和 init.sh 所扮演的角色。

## 开放问题

- 单一通用 Agent vs 多 Agent 架构（测试 Agent、QA Agent、代码清理 Agent）哪个更好？
- 这些实践能否泛化到 Web 开发以外的领域（科研、金融建模等）？

## 相关概念

- [Initializer/Coding Agent 模式](initializer-coding-agent.md) — 解决 long-running 问题的具体设计
- [Feature List Pattern](feature-list-pattern.md) — 拆解需求为可追踪的细粒度项
- [Session](session.md) — 跨 context window 的持久化存储
- [Harness](harness.md) — Agent 的编排循环
- [Context Engineering](context-engineering.md) — 管理 context window 的工程实践
- [Context Anxiety](context-anxiety.md) — context window 限制下模型的行为异常
