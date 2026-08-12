---
title: OpenAI Agents SDK
date: 2026-08-11
tags: [framework, sdk, openai, entity]
sources: [OpenAI-Agents-SDK-Documentation.md]
---

# OpenAI Agents SDK

[[entities/07-openai]] 的开源 agent 框架（Python：`openai-agents-python`）。**code-first / 非声明式**阵营的代表。

## 四个原语

- **Agent** = LLM + instructions + tools（+ `output_type`、`handoffs`、guardrails、hooks）
- **Runner** = `run()` / `run_sync()` / `run_streamed()`，实现 [[concepts/21-agent-loop]]
- **Handoff** = 一种工具，调用即移交控制权与会话状态（[[concepts/36-handoff]]）
- **Guardrail** = 并发校验 + tripwire 异常（[[concepts/37-guardrails]]）

## 特色能力

- **工具的三个层次**：hosted tools（web search、file search、computer use、code interpreter、container shell + skills、**hosted tool search**、**Programmatic Tool Calling**）、function tools（自动 schema/docstring 解析、Pydantic Field 约束、超时、错误处理、返回图片/文件）、**agents as tools**（含 approval gates、结构化输入、自定义输出抽取）
- **四种状态策略**：`to_input_list()` / `session` / `conversation_id` / `previous_response_id`（[[concepts/26-memory]]）
- **持久执行集成**：Temporal、Dapr、Restate、DBOS（[[concepts/43-durable-execution]]）
- **可观测性**：内置 tracing 与 `group_id`
- **恢复能力**：`RunState` 支持恢复被暂停/取消的运行
- 可选 Responses WebSocket 传输以复用连接

## 设计立场

官方文档与白皮书都明确表达：声明式框架要求预先定义每个分支/循环、还要学 DSL，工作流一动态化就笨重；Agents SDK 让开发者**用熟悉的编程结构直接表达逻辑**，无需预定义整张图。

对照 [[entities/10-langgraph]] 与 [[analysis/03-framework-comparison]]。

## 来源

- [[sources/11-openai-agents-sdk-docs]]、[[sources/05-openai-practical-guide-to-building-agents]]
