---
title: 渐进披露与代码执行（Code Mode）
date: 2026-08-11
tags: [mcp, context-efficiency, code-execution, skills]
sources: [Code-Execution-with-MCP.md, Effective-Context-Engineering-for-AI-Agents.md]
---

# 渐进披露与代码执行（Code Mode）

工具规模化后的标准解法。**这是最能体现"读过 2025–2026 年最新工程实践"的话题之一。**

## 问题：两种 token 膨胀

1. **工具定义压满 context**：多数 MCP client 预先加载全部工具定义；接入上千工具时，模型读到请求前已消耗几十万 token
2. **中间结果重复穿过 context**：`gdrive.getDocument` 返回完整转录 → 模型把整段**重新写出来**传给 `salesforce.updateRecord`。同一内容穿两次；2 小时会议多花约 5 万 token；大文档直接超限；复制大段数据时模型更易出错

## 解法：把 MCP server 当代码 API

生成工具的文件树，让 agent **写代码**调用：

```text
servers/
├── google-drive/{getDocument.ts, index.ts, ...}
└── salesforce/{updateRecord.ts, index.ts, ...}
```

```ts
const transcript = (await gdrive.getDocument({ documentId: 'abc123' })).content;
await salesforce.updateRecord({ objectType: 'SalesMeeting', recordId: '...', data: { Notes: transcript } });
```

agent 通过列 `./servers/` 目录发现 server、按需读取具体工具文件了解接口。**150,000 → 2,000 token，节省 98.7%。** Cloudflare 的同类实践称为 **"Code Mode"**。

## 五项收益

| 收益 | 说明 |
|---|---|
| **Progressive disclosure** | 模型擅长在文件系统导航，按需读定义；或提供 `search_tools` 并带**详略级别**（仅名称 / 名称+描述 / 完整 schema） |
| **context 高效的结果** | 1 万行表格在执行环境里 filter 后只打印 5 行；聚合、跨源 join、字段抽取同理 |
| **更强控制流** | 循环/条件/错误处理用代码写，胜过 agent 循环里交替 tool call + sleep；写死条件分支还省"首 token 延迟" |
| **隐私保护** | 中间结果默认留在执行环境；更敏感场景由 harness **自动 tokenize PII**（`[EMAIL_1]`），真实值在 client 侧回填，**数据从不经过模型**；可定义确定性的数据流向规则 |
| **状态与 Skills** | 中间结果落盘可恢复；跑通的代码存成可复用函数（`./skills/*.ts` + `SKILL.md`）→ 随时间积累出更高层能力 |

## 代价（要主动说出来）

跑 agent 生成的代码需要安全的 [[concepts/03-sandbox|沙箱]]、资源限制与监控 —— 这些运维开销与安全考量是直接 tool call 所没有的。**收益（token、延迟、组合性）要与实现成本权衡。**

## 同一原则的其他形态

- **hosted tool search / Programmatic Tool Calling**（OpenAI Agents SDK）：思路相同 —— 工具过多时按需检索或用代码调度
- **glob/grep 而非预建索引**（Claude Code）：数据侧的渐进披露
- **`response_format: concise|detailed`**：单个工具内部的详略分级

## 面试可引用的判断句

> 这些问题（context 管理、工具组合、状态持久化）看似新颖，其实软件工程早有成熟解法；代码执行只是把这些既有模式搬到 agent 上。

## 与其他概念的关系

- [[concepts/31-mcp]]：本页解决 MCP 规模化的痛点
- [[concepts/28-agentic-search]]：数据侧的同一原则
- [[concepts/45-token-economics]]：本页是最有效的降本手段之一
- [[concepts/03-sandbox]]：代码执行的前置条件

## 来源

- [[sources/09-code-execution-with-mcp]]、[[sources/06-effective-context-engineering]]、[[sources/11-openai-agents-sdk-docs]]
