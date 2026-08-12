---
title: LangGraph / LangChain
date: 2026-08-11
tags: [framework, langgraph, langchain, entity]
sources: [LangGraph-Workflows-and-Agents.md]
---

# LangGraph / LangChain

LangChain Inc. 的开源 agent 技术栈。面试常问"LangChain 和 LangGraph 什么关系"，官方分层如下：

| 组件 | 定位 |
|---|---|
| **LangChain** | agent **框架**：模型、工具、agent loop 的抽象与集成 |
| **LangGraph** | 编排**运行时**：durable execution、streaming、human-in-the-loop、persistence |
| **Deep Agents** | 建在 LangGraph 上的 agent **harness**：规划、子 agent、文件系统工具、上下文管理 |
| **LangSmith** | 平台：tracing、评估、prompt、部署（Engine 能检测 trace 中的问题并提 PR，Fleet 是无代码 agent 构建器） |

LangGraph 可独立使用，不依赖 LangChain。

## 技术特征

- **声明式图**：`StateGraph` + 节点 + 边（含条件边），`Send` API 支持运行时动态创建 worker
- 核心卖点：**在同一张图里混合确定性步骤与 LLM 驱动步骤** —— 需要可审计就写死，需要灵活就交给模型
- 六项收益：混合确定性/agentic、persistence、HIL、完整记忆（短期 + 跨会话）、LangSmith 调试、生产部署
- 设计灵感：Pregel、Apache Beam；接口借鉴 NetworkX
- 采用方：文档列举 Klarna、Uber、J.P. Morgan 等

## 在本库中的位置

- 官方文档把 Anthropic 的五种 workflow 模式逐个实现了一遍 → [[sources/12-langgraph-workflows-and-agents]]
- 它是"声明式图"阵营的代表，与 code-first 的 [[entities/11-openai-agents-sdk]] 形成对照 → [[analysis/03-framework-comparison]]
- 持久化与中断能力对应 [[concepts/43-durable-execution]]、[[concepts/38-human-in-the-loop]]
