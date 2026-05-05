# Context Engineering

tags: [agent-architecture, context-window, prompt-engineering]
date: 2026-04-23
sources: [../../raw/Scaling-Managed-Agents-Decoupling.md]
status: active

## Definition

管理 LLM context window 内容的工程实践——决定保留什么、丢弃什么、如何组织。随着任务超出 context window 长度，context engineering 成为 Agent 系统的核心挑战。

## 主要技术

| 技术 | 描述 | 风险 |
|------|------|------|
| **Compaction** | 让 Claude 保存 context window 的摘要，然后清除原始内容 | 摘要丢失细节，不可逆 |
| **Memory Tool** | 让 Claude 将上下文写入文件，跨 session 学习 | 需要主动使用，可能遗漏 |
| **Context Trimming** | 选择性移除旧工具结果或思考块 | 可能丢弃未来需要的信息 |

## 核心难题

> 不可逆地选择保留或丢弃上下文可能导致失败。很难知道未来的回合需要哪些 token。

## 在 Managed Agents 中的解决

[Managed Agents](../entities/Managed-Agents.md) 通过 [Session](session.md) 与 [Harness](harness.md) 的分离来解决：

- **Session**: 保证所有事件持久化、可查询、不丢失
- **Harness**: 自由决定将哪些事件放入 context window（compaction、trimming、缓存优化等）
- **关注点分离**: 存储的不可逆性与上下文管理的灵活性解耦

## 面向未来

> 我们无法预知未来模型需要什么特定的上下文工程。接口将上下文管理推给 harness，session 只保证持久和可用。

这意味着 context engineering 的具体实现会随模型进化而变化，但 session 作为数据源保持稳定。

## 相关概念

- [Session](session.md) — context engineering 的数据源
- [Harness](harness.md) — context engineering 的执行者
- [Context Anxiety](context-anxiety.md) — context 管理不当引发的模型行为
- [Sandbox](sandbox.md) — 独立于 context 的执行环境
