---
title: "MCP Architecture Overview (spec 2026-07-28)"
date: 2026-08-11
tags: [source, mcp, protocol, spec]
sources: [MCP-Architecture-Overview.md]
status: ingested
---

# MCP Architecture Overview（协议规范 2026-07-28）

- 来源：[[entities/09-model-context-protocol]] 官方文档 `modelcontextprotocol.io/docs/learn/architecture`
- 原文：[raw/MCP-Architecture-Overview.md](../../raw/MCP-Architecture-Overview.md)

MCP 的一手规范说明。**注意版本**：本页依据 `2026-07-28` 版协议，若面试引用的是 2024/2025 年的旧材料，有几处已经变了（下面标了 ⚠️）。概念页见 [[concepts/31-mcp]]。

## 参与者（三角色）

| 角色 | 职责 |
|---|---|
| **MCP Host** | AI 应用本体（Claude Code、Claude Desktop、VS Code），协调管理一个或多个 client |
| **MCP Client** | 与某个 server 维持一条专属连接，为 host 取上下文 |
| **MCP Server** | 提供上下文的程序，可本地也可远程 |

**一个 server 对应一个 client 实例**：VS Code 连 Sentry server 时创建一个 client，再连 filesystem server 时创建第二个。本地 STDIO server 通常只服务一个 client，远程 Streamable HTTP server 通常服务多个。

"local / remote server"的区别只在于运行位置与传输方式，**不是两种不同的东西**。

## 两层结构

- **Data layer（内层）**：基于 **JSON-RPC 2.0** 的协议，定义消息结构与语义 —— 发现、server features、client features、工具/资源/提示等原语、通知、进度
- **Transport layer（外层）**：通信机制与鉴权 —— 连接建立、消息分帧、授权
  - **STDIO**：本地进程间标准输入输出，无网络开销，性能最优
  - **Streamable HTTP**：client→server 用 HTTP POST，可选 SSE 做流式；支持 bearer token / API key / 自定义头，**推荐用 OAuth 获取令牌**

传输层对协议层抽象通信细节，因此同一套 JSON-RPC 消息格式可跑在任意传输上 —— 这就是"实现一次 MCP，解锁整个生态"的技术根据。

## 无状态与发现 ⚠️

- **MCP 是无状态协议**：每个请求在 `_meta` 字段里自带协议版本与相关 capabilities，server 不从历史请求推断任何东西；client 也应在同一字段标识自己
- server 通过**强制的 `server/discover` 请求**公布支持的版本与能力，client 可在任何其他请求之前发送
- （旧版材料常讲 `initialize` 握手 + 有状态会话，这是变化点）

## 原语（primitives）—— 面试必背

**Server 可暴露三种**：

| 原语 | 含义 | 例子 |
|---|---|---|
| **Tools** | 可执行函数，AI 应用调用来**做动作** | 文件操作、API 调用、数据库查询 |
| **Resources** | 提供上下文**数据源** | 文件内容、数据库记录、API 响应 |
| **Prompts** | 可复用的交互**模板** | system prompt、few-shot 示例 |

每类原语都有 `*/list`（发现）、`*/get`（获取）、部分有执行（`tools/call`）。client 先 `tools/list` 再执行，因此**列表是动态的**。典型组合：一个数据库 server 暴露"查询工具 + schema 资源 + few-shot 提示"。

**Client 可暴露**：
- **Elicitation**：server 反过来向用户要信息或要确认（`elicitation/create`），通过 Multi Round-Trip Requests 模式送达
- ⚠️ **Sampling 与 Logging 在 `2026-07-28` 已废弃**。Sampling（server 借用 client 的模型能力，`sampling/createMessage`）建议改为直接对接 LLM 供应商 API；Logging 建议改为写 `stderr`（stdio）或用 OpenTelemetry

**扩展**：如 **Tasks extension** —— server 对长耗时请求返回持久句柄，client 轮询状态、稍后取结果（对长时运行 agent 很关键，参见 [[concepts/43-durable-execution]]）。

## 通知 ⚠️

支持实时通知（如工具列表变化）。JSON-RPC notification 不期待响应。**变更通知是 opt-in 的**：client 开一条长连接 `subscriptions/listen` 并声明想接收的通知类型，server 只在该流上投递匹配的通知。

## 面试要点提炼

1. MCP 只管**上下文交换的协议**，不规定 AI 应用如何使用 LLM、如何管理上下文
2. 与 [[concepts/33-a2a]] 的分工：MCP 解决"agent ↔ 工具/数据"，A2A 解决"agent ↔ agent"
3. 工具太多导致 token 膨胀不是协议缺陷，而是 client 加载策略问题 → [[sources/09-code-execution-with-mcp]] 的渐进披露方案
4. 安全上，MCP server 与工具包属于**供应链面**（OWASP LLM04 / ASI04），要 pin、签名、审计工具描述里的隐藏指令，见 [[concepts/39-prompt-injection]]
