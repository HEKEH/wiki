---
title: MCP（Model Context Protocol）
date: 2026-08-11
tags: [mcp, protocol, tools, integration]
sources: [MCP-Architecture-Overview.md, Code-Execution-with-MCP.md]
---

# MCP（Model Context Protocol）

由 [[entities/01-anthropic]] 于 2024 年 11 月发布的开放标准，用于把 AI agent 连接到外部系统。已成为事实标准（社区已建成千上万个 server，主流语言均有 SDK）。

## 它解决什么问题

传统做法：每对「agent × 工具」都要一个定制集成 → 碎片化、重复劳动。MCP 提供通用协议：**在 agent 里实现一次，解锁整个集成生态**（M×N → M+N）。

## 架构速记

- **三角色**：Host（AI 应用）→ 为每个 server 创建一个 **Client** → 连接 **Server**（提供上下文的程序）
- **两层**：Data layer（JSON-RPC 2.0 协议、发现、原语、通知）+ Transport layer（STDIO 本地 / Streamable HTTP 远程，推荐 OAuth 取令牌）
- **Server 三原语**：**Tools**（做动作）、**Resources**（读数据）、**Prompts**（交互模板）；每类有 `*/list` 发现、`*/get` 获取，工具另有 `tools/call`
- **Client 原语**：**Elicitation**（server 反向向用户索取信息/确认）
- 详细规范与版本变化见 [[sources/10-mcp-architecture-overview]]

## 版本注意（2026-07-28）

面试如引用旧材料容易出错的三点：

1. **协议是无状态的** —— 每个请求在 `_meta` 里自带版本与 capabilities；通过强制的 **`server/discover`** 公布能力（不再是有状态 initialize 握手心智模型）
2. **Sampling 与 Logging 已废弃** —— sampling（server 借用 client 的模型）建议改为直接对接 LLM API；logging 改用 `stderr` 或 OpenTelemetry
3. **变更通知 opt-in** —— client 开 `subscriptions/listen` 流并声明想要的通知类型
4. **Tasks 扩展** —— 长耗时请求返回持久句柄，可轮询与稍后取结果（长时运行 agent 的关键能力）

## 三个高频面试题

**Q：MCP 和 function calling 什么关系？**
function calling 是**模型能力**（模型输出结构化调用请求）；MCP 是**分发与集成协议**（工具从哪来、怎么发现、怎么传输）。MCP server 提供的工具最终仍以 function calling 的形式呈现给模型。

**Q：MCP 和 A2A 什么关系？**
MCP 管 agent↔工具/数据，A2A 管 agent↔agent（对端不透明）。见 [[concepts/33-a2a]]。

**Q：接了几十个 MCP server 后 context 爆了怎么办？**
这是 client 加载策略问题，不是协议缺陷。三档解法：
1. 只连必要 server / 做工具白名单
2. **渐进披露**：文件树按需读取，或 `search_tools`（带详略参数）
3. **代码执行**：把 server 当代码 API，让 agent 写代码调用 —— 150k → 2k token（-98.7%）
详见 [[concepts/32-progressive-disclosure-and-code-execution]]

## 安全（必须能说）

- MCP 直接放大 [[sources/16-lethal-trifecta|致命三要素]]：它**鼓励用户混搭**来自不同来源的工具 —— 一些能访问私有数据，一些会引入不可信内容，而任何能发 HTTP 请求的工具都是外传通道。GitHub 官方 MCP server 的漏洞就是三要素集于一体
- MCP server 与工具包属**供应链面**（OWASP LLM04 / ASI04）：pin 版本、签名验证、**审计工具描述里的隐藏指令**、监控工具组成变化
- 工具输出是**不可信输入**：会重新进入 context 并可能触发链式动作
- 用 **tool annotations** 声明破坏性操作与开放世界访问；高风险动作走审批（[[concepts/38-human-in-the-loop]]）

## 与其他概念的关系

- [[concepts/25-tool-use-and-function-calling]]：MCP 是它的标准化分发形态
- [[concepts/20-aci]] / [[sources/08-writing-effective-tools-for-agents]]：写 MCP server 的设计原则
- [[concepts/39-prompt-injection]]、[[concepts/40-excessive-agency]]：MCP 场景下的两大风险

## 来源

- [[sources/10-mcp-architecture-overview]]、[[sources/09-code-execution-with-mcp]]、[[sources/16-lethal-trifecta]]
