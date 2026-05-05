---
tags: [agent, building-block, llm, mcp]
date: 2026-04-25
sources:
  - sources/Building-Effective-AI-Agents.md
---

# The Augmented LLM

一切 Agentic System 的基础构件：LLM + 检索 + 工具 + 记忆。

## 定义

增强型 LLM 不仅能生成文本，还能**主动使用**附加能力：

- **检索（Retrieval）**：生成搜索查询，获取外部信息
- **工具（Tools）**：选择并调用合适的工具
- **记忆（Memory）**：决定保留什么信息供后续使用

关键特征是 LLM 的**主动性**——不是被动接收增强信息，而是自己决定何时、如何使用这些能力。

## 实现建议

- 针对具体用例定制增强能力
- 确保增强能力提供简洁、文档完善的接口
- 推荐通过 [Model Context Protocol (MCP)](https://modelcontextprotocol.io/) 集成第三方工具生态

## 在模式层级中的位置

```
Augmented LLM（基础构件）
  → Workflow 模式（预定义编排）
    → Agent（动态自控）
```

Augmented LLM 是所有 [Agentic Systems](agentic-systems.md) 的起点。每个 Workflow 模式中的节点、Agent 循环中的每一步，本质上都是一个 Augmented LLM 调用。

## 与其他概念的关系

- [Context Engineering](context-engineering.md) 管理 Augmented LLM 中"记忆"增强的具体内容
- [Agent-Computer Interface (ACI)](aci.md) 关注 Augmented LLM 中"工具"增强的接口设计
- [Harness](harness.md) 是 Augmented LLM 的运行时包装：调用 LLM → 路由工具 → 追加结果 → 循环

## 来源

- [Building Effective AI Agents](../sources/Building-Effective-AI-Agents.md) — Anthropic 工程博客
