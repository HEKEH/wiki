---
title: A2A Project（Agent2Agent）
date: 2026-08-11
tags: [project, protocol, a2a, entity]
sources: [A2A-Protocol-Overview.md]
---

# A2A Project

**Agent2Agent 协议**的开源项目，仓库 `a2aproject/A2A`，Apache 2.0。

治理沿革：Google 于 **2025-04** 发布 A2A，**2025-06** 捐赠给 **Linux Foundation** 以获得中立治理，由技术指导委员会维护（成员含 AWS、Cisco、Google、IBM、Microsoft、Salesforce、SAP、ServiceNow）。此沿革不在存档的 README 正文内，依据 Google Developers Blog 与 Linux Foundation 公告。

## 提供什么

- 协议规范与文档站：`a2a-protocol.org`
- 多语言 SDK（Python：`pip install a2a-sdk` 等）
- 与各框架的集成示例：Google ADK、LangGraph、BeeAI 等均可暴露为 A2A server
- 与 DeepLearning.AI、Google Cloud、IBM Research 合作的公开短课

## 协议要点

JSON-RPC 2.0 over HTTP(S)、**Agent Card** 做能力发现、同步/SSE 流式/异步 push 三种交互、支持文本+文件+结构化 JSON、面向企业的安全与可观测性设计。核心特色是**保持对端不透明**（不暴露内部状态、记忆、工具）。

详见 [[concepts/33-a2a]] 与 [[sources/13-a2a-protocol-overview]]。

## 与 MCP 的关系

互补而非竞争：[[entities/09-model-context-protocol]] 解决 agent↔工具，A2A 解决 agent↔agent。一个 A2A server agent 内部完全可以用 MCP 连自己的工具。
