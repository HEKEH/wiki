---
title: "LangGraph: Workflows and Agents"
date: 2026-08-11
tags: [source, langgraph, framework, workflow-patterns, orchestration]
sources: [LangGraph-Workflows-and-Agents.md]
status: ingested
---

# LangGraph：Workflows and Agents

- 来源：[[entities/10-langgraph]] 官方文档（`docs.langchain.com/oss/python/langgraph`：workflows-agents + overview）
- 原文：[raw/LangGraph-Workflows-and-Agents.md](../../raw/LangGraph-Workflows-and-Agents.md)

这篇文档直接沿用了 [[sources/03-building-effective-ai-agents]] 的五种模式分类，并**逐个给出可运行的图代码**。它是"Anthropic 的模式 → 具体实现"的最佳桥梁，也是面试手写代码题的模板来源。

## 定位：低层编排运行时

> LangGraph is a low-level orchestration framework and runtime for building, managing, and deploying long-running, stateful agents.

- 不抽象 prompt、不规定架构，只提供编排能力
- 核心卖点：**在同一张图里混合确定性步骤与 LLM 驱动步骤** —— 需要可靠可审计的地方写死，需要灵活的地方交给模型
- 六项核心收益：混合确定性/agentic 步骤、**persistence**（跨失败恢复、长时运行）、**human-in-the-loop**（任意点检查修改状态）、**comprehensive memory**（短期工作记忆 + 跨会话长期记忆）、LangSmith 调试、生产部署
- 生态分层（值得记，用来回答"LangChain 和 LangGraph 什么关系"）：
  - **LangChain** = agent 框架（模型/工具/agent loop 的抽象与集成）
  - **LangGraph** = 编排运行时（durable execution、streaming、HIL、persistence）
  - **Deep Agents** = 建在 LangGraph 上的 agent harness（规划、子 agent、文件系统工具、上下文管理）
  - **LangSmith** = tracing / eval / prompt / 部署平台
- 设计灵感来自 Pregel 与 Apache Beam，接口借鉴 NetworkX

## 基础构件

- `StateGraph(State)` + `add_node` + `add_edge` / `add_conditional_edges` + `compile()`
- **增强 LLM** 是一切的起点（对应 [[concepts/14-augmented-llm]]）：`llm.with_structured_output(Schema)` 得到结构化输出；`llm.bind_tools([tool])` 得到工具调用能力

## 五种 workflow 模式的图实现

| 模式 | 图结构要点 | 对应概念页 |
|---|---|---|
| **Prompt chaining** | 线性节点串联 + 可选 gate（条件边做质量检查，不合格回退） | [[concepts/15-prompt-chaining]] |
| **Parallelization** | 从一个节点扇出多条边到并行节点，再汇聚到聚合节点 | [[concepts/17-parallelization]] |
| **Routing** | 用结构化输出让 LLM 做分类，条件边路由到专门节点 | [[concepts/16-routing]] |
| **Orchestrator-worker** | orchestrator 用结构化输出规划出任务列表，用 `Send` API 动态创建 worker（数量运行时决定），worker 结果写回共享 state | [[concepts/18-orchestrator-workers]] |
| **Evaluator-optimizer** | generator ↔ evaluator 两节点成环，evaluator 用结构化输出给 grade + feedback，未通过则带反馈回到 generator | [[concepts/19-evaluator-optimizer]] |

关键区别（面试常考）：**Parallelization 的分支数在编译期已知，Orchestrator-worker 的 worker 数在运行时由 LLM 决定**。

## Agent（非 workflow）的实现

- 一个 `llm_call` 节点 + 一个 `environment`/工具节点 + 条件边形成**循环**：模型请求工具就走工具节点再回来，不请求就 END —— 这就是 [[concepts/21-agent-loop]] 的图形态
- `ToolNode` 是预置的工具执行节点；LangChain 侧还有预置的 agent 架构（常见的 LLM + tool-calling 循环）

## 面试可用的判断句

- "workflow 有预定的代码路径、按既定顺序执行；agent 动态定义自己的过程与工具使用" —— 与 Anthropic 的定义完全一致，说明这套分类已成行业共识
- LangGraph 是**声明式图**的代表，OpenAI Agents SDK 明确批评过这种做法在动态工作流下的笨重；两者的取舍见 [[analysis/03-framework-comparison]]
- 面试若问"你怎么保证 agent 跑几小时不丢状态"：LangGraph 的答案是 checkpointer + persistence + interrupt（HIL），见 [[concepts/43-durable-execution]]
