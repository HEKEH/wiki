# Sandbox

tags: [agent-architecture, execution, container, security]
date: 2026-04-23
sources: [../../raw/Scaling-Managed-Agents-Decoupling.md]
status: active

## Definition

Agent 的"双手"——Claude 运行代码和编辑文件的执行环境。Sandbox 是可替换的 cattle，通过统一接口被 [Harness](harness.md) 调用。

## 核心接口

- `execute(name, input) → string` — 统一的工具调用接口
- `provision({resources})` — 按需创建新 sandbox 的标准配方

## 关键特征

1. **Cattle 而非 Pet** — sandbox 失败时 harness 捕获错误，Claude 可决定重试，新 sandbox 通过 `provision()` 初始化
2. **延迟创建** — sandbox 仅在需要时通过工具调用创建，无需 sandbox 的 session 不等待容器启动
3. **类型无关** — harness 不知道 sandbox 是容器、手机还是 Pokémon 模拟器
4. **可传递** — brain 之间可以传递 hands

## 安全模型

### 核心原则：凭证不暴露在 sandbox 中

耦合设计中，Claude 生成的不可信代码运行在与凭证同一容器内——prompt injection 只需说服 Claude 读取自己的环境变量。一旦获得 token，攻击者可以创建不受限的新 session。

### 两种安全模式

1. **资源捆绑 Auth（Git 模式）** — 用 repo 的 access token 在初始化时 clone 并注入 git remote，sandbox 内 push/pull 无需 token
2. **保险库 + 代理（MCP 模式）** — OAuth token 存于安全保险库，Claude 通过专用代理调用 MCP 工具，代理从保险库取凭证

## 性能影响

Sandbox 延迟创建显著降低了 TTFT：
- p50 TTFT 降低 ~60%
- p95 TTFT 降低 >90%

原因：inference 不再等待容器启动，orchestrator 拉取 session 事件后即可开始。

## 多手

一个 brain 可以连接多个 sandbox——Claude 必须推理多个执行环境并决定将工作发送到哪里。这比在单一 shell 中操作更复杂的认知任务，但随着模型智能提升已变得可行。

## 相关概念

- [Harness](harness.md) — 通过工具调用操作 sandbox
- [Session](session.md) — 独立于 sandbox 的持久化日志
- [Pets vs Cattle](pets-vs-cattle.md) — sandbox 的设计哲学
