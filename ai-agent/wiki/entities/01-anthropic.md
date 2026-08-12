---
title: Anthropic
tags: [organization, ai-safety, llm]
date: 2026-04-23
sources: [Scaling-Managed-Agents-Decoupling.md]
status: stub
---

# Anthropic

## Overview

AI 安全公司，Claude 系列模型的开发商。在工程博客上持续发布 Agent 架构设计方面的深度文章。

## 产品

- **Claude** — 大语言模型系列，包括 Opus、Sonnet、Haiku 等版本
- **[[entities/02-managed-agents]]** — 托管式 Agent 服务，在 Claude Platform 上代用户运行长周期 Agent

## 工程博客贡献

- [Building Effective AI Agents](https://www.anthropic.com/engineering/building-effective-agents) — Agent 系统的分类框架与五种 Workflow 模式（→ [[sources/03-building-effective-ai-agents|Wiki 摘要]]）
- [Effective Harnesses for Long-Running Agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) — 长运行 Agent 的 harness 设计（→ [[sources/02-effective-harnesses-for-long-running-agents|Wiki 摘要]]）
- [Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — 上下文工程方法论（→ [[sources/06-effective-context-engineering|Wiki 摘要]]）
- [Scaling Managed Agents: Decoupling](../../raw/Scaling-Managed-Agents-Decoupling.md) — Managed Agents 的解耦架构（→ [[sources/01-scaling-managed-agents-decoupling|Wiki 摘要]]）
- [How we built our multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system) — 多 agent 生产复盘（→ [[sources/07-multi-agent-research-system|Wiki 摘要]]）
- [Writing effective tools for AI agents](https://www.anthropic.com/engineering/writing-tools-for-agents) — 工具设计与评估驱动改进（→ [[sources/08-writing-effective-tools-for-agents|Wiki 摘要]]）
- [Code execution with MCP](https://www.anthropic.com/engineering/code-execution-with-mcp) — 用代码执行降低 MCP token 开销（→ [[sources/09-code-execution-with-mcp|Wiki 摘要]]）

## 标准与开源

- **[[entities/09-model-context-protocol]]** — 2024 年 11 月发布的 MCP 开放标准，已成为连接 agent 与工具/数据的事实标准

## 相关概念

- [[concepts/02-harness]]
- [[concepts/07-meta-harness]]
- [[concepts/09-context-engineering]]
- [[concepts/31-mcp]]
- [[concepts/34-orchestrator-worker-multi-agent]]
