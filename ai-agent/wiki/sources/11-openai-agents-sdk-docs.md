---
title: "OpenAI Agents SDK — Core Documentation"
date: 2026-08-11
tags: [source, openai, framework, sdk, handoffs, guardrails]
sources: [OpenAI-Agents-SDK-Documentation.md]
status: ingested
---

# OpenAI Agents SDK 核心文档

- 来源：[[entities/11-openai-agents-sdk]] 官方文档（agents / running agents / tools / handoffs / multi-agent / guardrails / context 七章合辑）
- 原文：[raw/OpenAI-Agents-SDK-Documentation.md](../../raw/OpenAI-Agents-SDK-Documentation.md)

**code-first（非声明式）框架的代表**。读它的价值在于：它把 [[sources/05-openai-practical-guide-to-building-agents]] 的抽象概念落成了具体 API，面试时能把"概念 → 代码"讲通。

## 四个原语

- **Agent**：LLM + instructions + tools（+ 可选 `output_type` 结构化输出、`handoffs`、`input_guardrails`/`output_guardrails`、lifecycle hooks）
- **Runner**：`run()`（异步）、`run_sync()`、`run_streamed()`（流式事件）
- **Handoff**：一种特殊工具，调用即把控制权与会话状态移交另一个 agent
- **Guardrail**：与主 agent 并发执行的校验，违规抛 tripwire 异常

## Agent loop（能背下来的版本）

`Runner.run()` 的循环：

1. 用当前 agent + 当前输入调用 LLM
2. LLM 产出：
   - 被判定为**最终输出** → 结束返回
   - 请求 **handoff** → 切换 current agent 与输入，重跑循环
   - 产生 **tool calls** → 执行工具、把结果追加进输入，重跑循环
3. 超过 `max_turns` → 抛 `MaxTurnsExceeded`（传 `None` 可关闭上限）

"最终输出"的判定规则：**产生了期望类型的文本输出，且没有工具调用**。输入还可以是 `RunState`，用于恢复被暂停或 `cancel(mode="after_turn")` 停下的运行。→ [[concepts/21-agent-loop]]

## 状态与会话的四种策略（面试常问"多轮怎么存"）

| 策略 | 状态在哪 | 适合 | 下一轮传什么 |
|---|---|---|---|
| `result.to_input_list()` | 你的应用内存 | 小型聊天循环、完全手动控制、任意供应商 | 上一轮 input list + 新用户消息 |
| `session` | 你的存储 + SDK | 需持久化、可恢复运行、自定义存储 | 同一个 session 实例（或指向同一存储的实例） |
| `conversation_id` | OpenAI Conversations API | 想跨 worker/服务共享的命名服务端会话 | 同一 `conversation_id` + 仅新一轮 |
| `previous_response_id` | OpenAI Responses API | 轻量的服务端续接，不建会话资源 | `result.last_response_id` + 仅新一轮 |

前两个是 client-managed，后两个是 OpenAI-managed。**一个会话只选一种**，混用会重复上下文。→ [[concepts/26-memory]]

## 多 agent：两种模式的 API 形态

- **Manager（agents as tools）**：`spanish_agent.as_tool(tool_name=..., tool_description=...)` 塞进 manager 的 `tools`。控制权不转移，manager 负责综合
- **Decentralized（handoffs）**：`triage_agent = Agent(..., handoffs=[support, sales, orders])`。handoff 是单向转移，被移交的 agent 直接接管并可与用户交互；可以给它一个"移交回去"的 handoff

配套能力：`as_tool` 的定制（自定义输出抽取、结构化输入、**approval gates**、流式嵌套运行）、`tool_use_behavior`（强制/停止在某个工具后）、`ModelSettings(tool_choice=...)` 强制用工具。→ [[concepts/35-manager-vs-decentralized]]、[[concepts/36-handoff]]

## 工具的层次

- **Hosted tools**（跑在 OpenAI 服务端）：web search、file search、computer use、code interpreter、hosted tool search、**Programmatic Tool Calling**、hosted container shell + skills
- **Function tools**：任意 Python 函数自动转 schema（自动解析签名与 docstring；用 Pydantic `Field` 约束与描述参数；支持超时、错误处理、返回图片/文件）
- **Agents as tools**：见上

其中 hosted tool search 与 Programmatic Tool Calling 本质是同一个问题的另一种解：工具多到装不进 context 时按需检索/用代码调度，与 [[concepts/32-progressive-disclosure-and-code-execution]] 对应。

## 生产能力（对齐面试里的"工程化"问题）

- **Guardrails**：`@input_guardrail` / output guardrail 函数返回 `GuardrailFunctionOutput(tripwire_triggered=...)`，触发即抛 `InputGuardrailTripwireTriggered`。默认**乐观执行**（主 agent 边跑，护栏并发校验）→ [[concepts/37-guardrails]]
- **Lifecycle hooks**：agent 级/运行级钩子；`call_model_input_filter` 可在每次调模型前改写输入（做上下文裁剪的挂点）
- **错误与恢复**：error handlers、异常体系
- **Durable execution + HIL**：官方集成 **Temporal、Dapr、Restate、DBOS**，把 agent 运行做成可持久、可恢复、可插入人工审批的工作流 → [[concepts/43-durable-execution]]、[[concepts/38-human-in-the-loop]]
- **Tracing**：内置 trace/`group_id`，把一次逻辑对话的多次 LLM 调用聚合

## 与其他框架的对照

见 [[analysis/03-framework-comparison]]。一句话：**LangGraph 让你显式建图（状态机 + checkpoint），Agents SDK 让你用普通代码写循环（agent/handoff/guardrail 是对象）**；前者控制力细、后者上手快。
