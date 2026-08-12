---
title: "Agent2Agent (A2A) Protocol Overview"
date: 2026-08-11
tags: [source, a2a, protocol, multi-agent, interop]
sources: [A2A-Protocol-Overview.md]
status: ingested
---

# Agent2Agent (A2A) Protocol

- 来源：[[entities/12-a2a-project]] 官方 README（仓库 `a2aproject/A2A`，Apache 2.0）
- 原文：[raw/A2A-Protocol-Overview.md](../../raw/A2A-Protocol-Overview.md)

> 归属说明：Google 于 2025-04 发布 A2A，2025-06 捐赠给 Linux Foundation 交由中立治理（技术指导委员会含 AWS、Cisco、Google、IBM、Microsoft、Salesforce、SAP、ServiceNow）。这一点**不在存档的 README 正文内**，来自 Google Developers Blog 与 Linux Foundation 的公告。

**agent 之间**的互操作协议，与 [[concepts/31-mcp]]（agent 与工具/数据之间）互补。面试里"MCP 和 A2A 什么关系"是高频题，这里是一手依据。

## 要解决的问题

> 让不同公司、不同框架、部署在不同服务器上的 agent，**作为 agent 而不仅仅作为工具**互相通信协作。

A2A 让 agent 能够：
- 互相**发现能力**
- **协商交互模态**（文本、表单、媒体）
- 安全地在**长时任务**上协作
- 协作时**不暴露内部状态、记忆或工具**（preserve opacity）

最后一点是它与"把 agent 包成工具"最本质的区别：A2A 假定对端是**不透明**（opaque）的黑盒 —— 保护知识产权与安全边界，也意味着你无法像本地子 agent 那样精细控制它。

## 关键特性

| 特性 | 内容 |
|---|---|
| 标准化通信 | **JSON-RPC 2.0 over HTTP(S)** |
| Agent 发现 | 通过 **Agent Card**（描述能力与连接信息的清单） |
| 灵活交互 | 同步请求/响应、流式（SSE）、异步 push notification |
| 富数据交换 | 文本、文件、结构化 JSON |
| 企业就绪 | 设计时考虑安全、认证、可观测性 |

四条设计目标：打破生态孤岛、支持复杂协作、推动开放标准、保持不透明性。

## 与 MCP 的分工（记这张表）

| | MCP | A2A |
|---|---|---|
| 连接对象 | agent ↔ 工具 / 数据源 | agent ↔ agent |
| 对端形态 | 透明的工具集（schema 已知） | 不透明的对等体（只知能力声明） |
| 发现机制 | `server/discover` + `*/list` | Agent Card |
| 传输 | STDIO / Streamable HTTP | HTTP(S) + SSE + push |
| 典型场景 | 让 agent 会用 GitHub、数据库、Slack | 让公司 A 的排班 agent 与公司 B 的支付 agent 协作 |

两者都用 JSON-RPC 2.0，可以同时使用：一个 A2A 服务端 agent 内部照样用 MCP 连自己的工具。

## 生态与学习资源

- 文档与规范：`a2a-protocol.org`（含完整 spec 与教程）
- SDK：Python（`pip install a2a-sdk`）等多语言
- DeepLearning.AI 与 Google Cloud、IBM Research 合作的短课：把 Google ADK / LangGraph / BeeAI 构建的 agent 暴露为 A2A server、从零写 A2A client、编排顺序与层级工作流、跨框架多 agent 医疗系统、A2A 如何补足 MCP

## 面试注意

- A2A 属于**跨组织/跨框架**互操作层。绝大多数面试涉及的"多 agent"其实是同进程内的 orchestrator-worker（[[concepts/34-orchestrator-worker-multi-agent]]），不需要 A2A —— 能说清"什么时候不需要它"比背特性更能加分
- 安全上，跨 agent 边界会放大 [[concepts/39-prompt-injection]]（对端返回的内容是不可信输入）与 [[concepts/40-excessive-agency]]（多跳后权限扩散：OWASP 明确要求"在委派或多 agent 工作流中保留原始用户上下文与授权范围"）
