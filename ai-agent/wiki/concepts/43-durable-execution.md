---
title: 持久执行与可观测性（Durable Execution & Observability）
date: 2026-08-11
tags: [production, durable-execution, checkpoint, tracing, deployment]
sources: [How-We-Built-Our-Multi-Agent-Research-System.md, OpenAI-Agents-SDK-Documentation.md, LangGraph-Workflows-and-Agents.md]
---

# 持久执行与可观测性

"从 prototype 到 production 的最后一公里往往是全程的大部分。"这一页是 agent 工程化面试的重头戏。

## 为什么特别难

> 传统软件里一个 bug 可能弄坏一个功能；在 agentic 系统里，**微小改动会级联成巨大的行为变化**。

- **agent 有状态且错误会复合**：跨很多次工具调用维持状态，一步失败会让它走上完全不同的轨迹
- **非确定性**：同样 prompt 两次运行结果不同，传统调试手段失效
- **长期运行**：部署新版本时，agent 可能停在流程的任意位置

## 四组对策

### 1. 可恢复而非重启

- 不能从头重跑（贵且伤体验）→ 建**能从出错处恢复**的系统
- 组合两种手段：**模型的适应性**（"告诉 agent 某工具失败了，让它自己适应，效果出乎意料地好"）+ **确定性保障**（重试逻辑、定期 checkpoint）
- 框架层支持：LangGraph 的 **checkpointer + persistence**（跨失败恢复、长时运行、从中断点继续）；OpenAI Agents SDK 官方集成 **Temporal / Dapr / Restate / DBOS**；`RunState` 支持恢复被暂停的运行
- MCP 侧对应能力：**Tasks 扩展**给长耗时请求返回持久句柄

### 2. 全链路 tracing

- 用户报"agent 找不到明显的信息"时，你需要知道：是查询写得差？来源选得差？还是工具失败了？
- **加上生产级 tracing 才能系统性定位**；除标准可观测性外，还监控 **agent 决策模式与交互结构**
- 隐私实践值得记：Anthropic **不监控单个会话内容**，只看高层模式
- 工具面：LangSmith（tracing/eval/prompt/部署）、Agents SDK 内置 trace 与 `group_id`（把一次逻辑对话里的多次 LLM 调用与 agent 切换聚合）

### 3. 部署协调

- agent 是"长期运行的、由 prompt/工具/执行逻辑组成的有状态网络"，**不能一次性把所有 agent 切到新版本**
- 用 **rainbow deployment**：新旧版本并存，逐步切流，避免打断在跑的 agent

### 4. 状态与人工介入的结合

状态可持久化才能做**任意点中断 → 人工检查/修改 → 继续**（LangGraph `interrupt`），这是 [[concepts/38-human-in-the-loop]] 的技术前提。

## 已知瓶颈：同步 vs 异步

Anthropic 明确列为当前限制：lead agent **同步**等待 subagent 完成 —— 简化协调但造成信息流瓶颈（lead 无法中途操纵 subagent、subagent 之间无法协调、单个慢 subagent 阻塞全局）。异步能带来更多并行，但引入**结果协调、状态一致性、跨 subagent 错误传播**的难题。

## 面试可用的清单式回答

被问"你怎么把 agent 上生产"，按这五层答：

1. **状态**：会话/检查点持久化，可从中断处恢复（不重跑）
2. **可观测**：全链路 trace + 决策模式监控 + 成本/延迟/错误指标
3. **安全**：护栏 + 最小权限 + 高风险动作审批（[[concepts/37-guardrails]]、[[concepts/40-excessive-agency]]）
4. **成本**：token 预算、缓存、模型分级（[[concepts/45-token-economics]]）
5. **发布**：渐进式部署（rainbow）+ 回归 eval 门禁（[[concepts/41-agent-evaluation]]）

## 与其他概念的关系

- [[concepts/01-session]]、[[concepts/04-brain-hands-session]]：状态解耦是持久执行的架构基础
- [[concepts/10-long-running-agent]]：跨 context window 的长时运行与本页互补（一个讲上下文，一个讲执行状态）
- [[concepts/05-pets-vs-cattle]]：可替换的执行体 + 不可丢失的状态

## 来源

- [[sources/07-multi-agent-research-system]]、[[sources/11-openai-agents-sdk-docs]]、[[sources/12-langgraph-workflows-and-agents]]、[[sources/10-mcp-architecture-overview]]
