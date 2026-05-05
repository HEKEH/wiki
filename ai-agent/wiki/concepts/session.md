# Session

tags: [agent-architecture, persistence, event-log]
date: 2026-04-23
sources: [../../raw/Scaling-Managed-Agents-Decoupling.md, ../../raw/Effective-harnesses-for-long.md]
status: active

## Definition

Agent 运行过程中的追加写入事件日志（append-only log），持久化存储所有发生的事件。Session 是独立于 [Harness](harness.md) 和 [Sandbox](sandbox.md) 的组件。

## 核心接口

- `getSession(id)` / `getEvents()` — 取回事件日志，支持位置切片查询
- `emitEvent(id, event)` — 追加写入事件，保持持久化记录
- `wake(sessionId)` — Harness 崩溃恢复时，用此接口重建状态

## Session ≠ Context Window

关键区分：

| | Session | Context Window |
|---|---------|----------------|
| 容量 | 无限（持久化存储） | 有限（token 上限） |
| 可逆性 | 完全可逆，所有事件保留 | 不可逆（compaction/trimming 丢弃信息） |
| 用途 | 事实来源（source of truth） | Claude 当前工作空间 |
| 管理 | 只保证可用和持久 | 由 [Harness](harness.md) 负责上下文工程 |

### 为什么这种分离重要

1. **不可逆决策的风险** — compaction 和 trimming 丢弃的信息可能在未来需要，但无法预知哪些 token 会被用到
2. **关注点分离** — Session 保证数据不丢；Harness 自由决定向 context window 中放入什么
3. **面向未来** — 无法预知未来模型需要什么上下文工程，接口将管理逻辑推给 harness，session 只保证持久和可查询

## 灵活查询

`getEvents()` 支持：
- 从上次停止的位置继续读取
- 回溯某个动作之前的事件以查看前因
- 重读某个关键时刻之前的上下文

## 跨 Session 状态衔接

[Effective Harnesses for Long-Running Agents](../sources/Effective-Harnesses-for-Long-Running-Agents.md) 展示了 Session 的实际应用——每个新 context window 开始时，Agent 通过三重机制恢复状态：

1. **claude-progress.txt** — 追加写入的交接笔记（自由文本），记录每个 session 的进展、问题、下一步和完成状态；主观的上下文叙述，与 feature list 的客观状态互补
2. **git history** — 代码变更的时间线，支持 `git revert` 回滚
3. **feature_list.json** — 结构化的完成状态追踪

这体现了 Session 设计的核心原则：**让下一个 session 能够快速理解上一个 session 的状态**，而不依赖 context window 的信息传递。

## 相关概念

- [Harness](harness.md) — 从 session 读取事件、写入 context window
- [Sandbox](sandbox.md) — 独立于 session 的执行环境
- [Context Engineering](context-engineering.md) — harness 如何管理 context window
- [Meta-harness](meta-harness.md) — session 是 meta-harness 的核心抽象之一
- [Long-Running Agent](long-running-agent.md) — 跨 session 工作是核心挑战
- [Initializer/Coding Agent 模式](initializer-coding-agent.md) — 跨 session 状态衔接的具体实践
