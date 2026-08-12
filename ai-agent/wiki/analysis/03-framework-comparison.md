---
title: Agent 框架对比与选型
date: 2026-08-11
tags: [analysis, frameworks, langgraph, agents-sdk, comparison]
sources: [LangGraph-Workflows-and-Agents.md, OpenAI-Agents-SDK-Documentation.md, OpenAI-A-Practical-Guide-to-Building-Agents.md, Building-Effective-AI-Agents.md]
---

# Agent 框架对比与选型

面试高频："你用过哪些框架？为什么选它？"**只报名字会被追问到底**，要能讲清抽象层次与取舍。

## 先记住 Anthropic 的立场

[[sources/03-building-effective-ai-agents]] 的核心建议是**先不要用框架**：

> 最成功的实现并没有使用复杂框架或专用库，而是用简单可组合的模式构建。

框架的风险：它在 prompt 与响应之上加了一层抽象，让你难以看清底层实际发生了什么，从而更难调试；也容易在简单方案够用时引入不必要的复杂度。

推荐路径：**先直接用 LLM API 手写循环**（几行代码就够），理解清楚后再决定是否引入框架；用框架时要理解其底层代码。

## 两大阵营

| | **声明式图**（LangGraph 代表） | **code-first**（OpenAI Agents SDK 代表） |
|---|---|---|
| 表达方式 | 预先定义节点与边（含条件边），编译成图 | 用普通编程结构写逻辑，agent/handoff/guardrail 是对象 |
| 优点 | 可视化清晰、可审计；**状态机 + checkpoint 天然支持持久化与中断** | 上手快；动态工作流表达自然；不必预定义整张图 |
| 缺点 | 工作流越动态越笨重，常需学专用 DSL 心智模型 | 图结构不显式，复杂流程的可视化与审计要自己补 |
| 官方互评 | — | OpenAI 白皮书明确批评声明式图在动态场景下"笨重"且需学 DSL |

**注意这不是"谁更好"的问题**：LangGraph 的图是为了拿到 durable execution / interrupt / 精细状态控制；Agents SDK 的 code-first 是为了减少概念负担。选择取决于你更需要哪一样。

## 主流选项速览

| 框架 | 抽象层次 | 特色 | 适合 |
|---|---|---|---|
| **直接用 API**（Anthropic 建议起点） | 无 | 完全透明，`while` 循环 + 工具分发 | 学习、简单 agent、需要极致可控 |
| **[[entities/11-openai-agents-sdk]]** | 中 | agent/handoff/guardrail/tracing 四件套；hosted tools；Temporal/Dapr/Restate/DBOS 持久执行集成 | OpenAI 生态、快速落地、多 agent handoff |
| **[[entities/10-langgraph]]** | 低（编排原语） | StateGraph、checkpointer、interrupt、`Send` 动态 worker；LangSmith 生态 | 长时运行、需要 HIL 与状态可控、混合确定性+agentic |
| **LangChain** | 高 | 模型/工具集成与预置 agent 架构 | 快速原型、集成多种模型与向量库 |
| **Deep Agents**（LangGraph 之上） | 高（harness） | 规划、子 agent、文件系统工具、上下文管理 | 直接要一个长程 harness |
| **Claude Agent SDK** | 中高（harness） | Anthropic 的长时运行 harness 实践（initializer + coding agent） | 编码类长程任务，见 [[sources/02-effective-harnesses-for-long-running-agents]] |
| **AutoGen**（2308.08155） | 中 | 可对话 agent 的多 agent 编排，可插人类 | 研究、多 agent 协作实验 |
| **CrewAI / smolagents 等** | 高 | 角色化多 agent / 极简代码 agent | 快速演示；生产需谨慎评估 |

（AutoGen 有论文依据；CrewAI、smolagents 在本库中未入库一手资料，属常识性补充，面试引用时注意区分。）

## 选型的三条判据

1. **控制权在谁手上**：需要在任意点检查/修改状态、按 checkpoint 恢复 → 选有持久化原语的（LangGraph / Temporal 类）；只需要"跑通一个工具循环" → 直接写代码
2. **工作流有多动态**：分支在编译期已知 → 图很合适；分支数量与内容由模型运行时决定 → code-first 更顺（或用 `Send` 这类动态 API）
3. **可观测与评估怎么接**：无论选谁，都要有 trace + eval 门禁（[[concepts/41-agent-evaluation]]、[[concepts/43-durable-execution]]）。这一条常被忽略，却是生产成败的关键

## 一段可直接用于面试的回答

> 我倾向先不用框架：用 API 手写循环把工具、退出条件、上下文策略搞清楚，因为框架会遮住"实际发送给模型的到底是什么"。当需求进入长时运行、人工审批、状态恢复这些方向时，我会引入编排运行时 —— LangGraph 这类的价值是 checkpoint 与 interrupt，不是画图本身。如果团队在 OpenAI 生态且流程比较动态，Agents SDK 的 code-first 心智负担更低。无论选哪个，我都会先把 tracing 和 20 条左右的回归 eval 接上，否则改 prompt 就是在赌。

## 相关

- [[analysis/04-single-vs-multi-agent]]：架构层面的取舍
- [[concepts/13-agentic-systems]]：模式分类
- [[concepts/07-meta-harness]]：为"尚未设想的 harness"设计接口的思路
