---
title: "Code Execution with MCP (Anthropic)"
date: 2026-08-11
tags: [source, anthropic, mcp, context-efficiency, code-execution]
sources: [Code-Execution-with-MCP.md]
status: ingested
---

# Code Execution with MCP: Building More Efficient AI Agents

- 作者：Adam Jones、Conor Kelly（[[entities/01-anthropic]]）
- 原文：[raw/Code-Execution-with-MCP.md](../../raw/Code-Execution-with-MCP.md)

MCP 生态成熟后暴露的**规模化问题**及其解法。面试里问到"MCP 工具太多怎么办"、"如何降低 agent 成本"，这篇给出可量化的答案。

## 问题：两种 token 膨胀

1. **工具定义压满 context**：多数 MCP client 把所有工具定义预先加载进 context。接入上千工具时，模型在读到请求之前就要处理几十万 token
2. **中间结果重复穿过 context**：例如"把 Google Drive 的会议记录附到 Salesforce lead"——`gdrive.getDocument` 返回完整转录进 context，再由模型把整段转录**重新写出来**传给 `salesforce.updateRecord`。同一份内容穿过两次；两小时会议可能多花 5 万 token，超大文档直接超限。而且模型在复制大段数据时更容易出错

## 解法：把 MCP server 当作代码 API

不再直接暴露 tool call，而是让 agent **写代码**调用 MCP。一种实现是把所有工具生成成文件树：

```text
servers/
├── google-drive/{getDocument.ts, ..., index.ts}
└── salesforce/{updateRecord.ts, ..., index.ts}
```

上例变成三行代码：读文档 → 取 `content` → 写 Salesforce。agent 通过**列目录发现 server、按需读取具体工具文件**来了解接口。

> token 用量从 150,000 降到 2,000 —— **节省 98.7%**。

Cloudflare 的同类做法称为 **"Code Mode"**，核心洞见相同：LLM 擅长写代码，就该利用这一点。

## 五项收益

1. **Progressive disclosure（渐进披露）** —— 模型擅长在文件系统里导航，按需读取工具定义而非一次读完。替代方案：提供 `search_tools` 工具，并带**详略参数**（仅名称 / 名称+描述 / 完整 schema），进一步省 context。见 [[concepts/32-progressive-disclosure-and-code-execution]]
2. **context 高效的工具结果** —— 1 万行表格在执行环境里 `filter` 后只打印 5 行给模型看；聚合、跨源 join、字段抽取同理
3. **更强的控制流** —— 循环、条件、错误处理用熟悉的代码写，而不是靠 agent 循环里交替 tool call + sleep；写死条件分支还能省"首 token 延迟"（不必等模型评估 if）
4. **隐私保护** —— 中间结果默认留在执行环境，模型只看到你显式 log/return 的部分。更敏感的场景可由 harness **自动 tokenize PII**（`[EMAIL_1]`、`[PHONE_1]`），真实值在 MCP client 侧回填，数据从 Google Sheets 流到 Salesforce 而**从不经过模型**；这也允许定义确定性的数据流向安全规则
5. **状态持久化与 Skills** —— 中间结果写文件，可恢复、可追踪进度；agent 还能把跑通的代码存成可复用函数（`./skills/save-sheet-as-csv.ts`），配上 `SKILL.md` 就是结构化 Skill。**随时间积累出更高层能力的工具箱，自己演化脚手架**

## 代价（原文明确提示）

跑 agent 生成的代码需要安全的[[concepts/03-sandbox|执行环境]]、资源限制与监控，这些基础设施开销和安全考量是直接 tool call 所不需要的。收益（token、延迟、组合性）要与实现成本权衡。

## 一句话总结

> 这些问题（context 管理、工具组合、状态持久化）看似新颖，其实软件工程早有成熟解法；代码执行只是把这些既有模式搬到 agent 上。

关联：[[concepts/31-mcp]]、[[sources/08-writing-effective-tools-for-agents]]、[[concepts/45-token-economics]]
