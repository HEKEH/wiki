---
title: Model Context Protocol (MCP) 项目
date: 2026-08-11
tags: [project, protocol, mcp, entity]
sources: [MCP-Architecture-Overview.md, Code-Execution-with-MCP.md]
---

# Model Context Protocol（项目）

由 [[entities/01-anthropic]] 于 **2024 年 11 月**发布的开放标准与开源项目，用于连接 AI agent 与外部系统。

## 项目范围

- **MCP Specification**：规定 client 与 server 的实现要求（当前版本 `2026-07-28`）
- **MCP SDKs**：覆盖各主流语言
- **开发工具**：MCP Inspector 等
- **参考 server 实现**：`modelcontextprotocol/servers`

治理相关文件（AGENTS.md、AI_POLICY.md、GOVERNANCE.md、SEP 提案流程）都在主仓库，说明它已按开放标准的方式运作。

## 采纳情况

发布以来采纳迅速：社区已建成千上万个 server，各主流语言 SDK 齐备，**行业已把 MCP 视为连接 agent 与工具/数据的事实标准**。

## 协议要点

见 [[concepts/31-mcp]] 与 [[sources/10-mcp-architecture-overview]]：三角色（Host/Client/Server）、两层（Data/Transport）、三原语（Tools/Resources/Prompts）+ 客户端 Elicitation、STDIO 与 Streamable HTTP 两种传输、无状态 + `server/discover` 发现、Tasks 等扩展。

## 与其他标准的关系

- **A2A**（[[entities/12-a2a-project]]）：管 agent↔agent，与 MCP 互补
- **function calling**：模型能力层，MCP 是分发与集成层
- 安全上属供应链面（OWASP LLM04 / ASI04），并直接放大[[sources/16-lethal-trifecta|致命三要素]]风险
