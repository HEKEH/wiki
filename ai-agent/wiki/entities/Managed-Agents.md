# Managed Agents

tags: [product, anthropic, agent-architecture, hosted-service]
date: 2026-04-23
sources: [../../raw/Scaling-Managed-Agents-Decoupling.md]
status: active

## Overview

[Anthropic](Anthropic.md) 在 Claude Platform 上提供的托管式 Agent 服务，代用户运行长周期 Agent 任务。其核心设计哲学是 [Meta-harness](../concepts/meta-harness.md)：对接口有主见，对实现无主见。

## 架构

Managed Agents 将 Agent 虚拟化为三个可独立替换的组件：

| 组件                                | 比喻  | 接口                                                        |
| --------------------------------- | --- | --------------------------------------------------------- |
| [Session](../concepts/session.md) | 记忆  | `getSession()`, `emitEvent()`, `getEvents()`              |
| [Harness](../concepts/harness.md) | 大脑  | `wake(sessionId)`, Claude 调用 + 工具路由                       |
| [Sandbox](../concepts/sandbox.md) | 双手  | `execute(name, input) → string`, `provision({resources})` |
|                                   |     |                                                           |

### 关键设计原则

1. **解耦** — Harness、Sandbox、Session 独立运行，任一组件失败不影响其他
2. **[Pets vs Cattle](../concepts/pets-vs-cattle.md)** — 容器和 harness 都是 cattle，可替换、可重启
3. **安全边界** — 凭证永远不暴露在 sandbox 中
4. **Session 持久化** — Session 是追加写入的事件日志，独立于 Claude 的 context window

## 性能影响

解耦 brain 和 hands 后：
- TTFT p50 降低 ~60%
- TTFT p95 降低 >90%

原因：无需 sandbox 的 session 不再等待容器启动；inference 可在 orchestrator 拉取 session 事件后立即开始。

## 安全模型

- **Git**: sandbox 初始化时用 access token clone repo 并注入 git remote，sandbox 内 push/pull 无需 token
- **MCP**: OAuth token 存于安全保险库，Claude 通过专用代理调用，代理从保险库取凭证
- **原则**: Harness 始终不接触凭证

## 与 Claude Code 的关系

Claude Code 是一种优秀的 [harness](../concepts/harness.md) 实现；Managed Agents 作为 meta-harness 可以容纳 Claude Code 或其他任务特定的 harness。
