---
title: 工具使用与 Function Calling
date: 2026-08-11
tags: [tools, function-calling, aci, mcp, fundamentals]
sources: [OpenAI-A-Practical-Guide-to-Building-Agents.md, Writing-Effective-Tools-for-AI-Agents.md, OpenAI-Agents-SDK-Documentation.md]
---

# 工具使用与 Function Calling

工具是 agent 感知与作用于世界的唯一通道。**面试必答题：一次工具调用在系统里到底发生了什么。**

## 机制：一次工具调用的完整链路

1. 应用把**工具定义**（name + description + JSON Schema 参数）随请求发给模型；Claude 侧会动态注入到 system prompt
2. 模型输出一个**结构化的工具调用请求**（name + arguments），而不是自然语言
3. **应用（不是模型）执行**该函数 —— 这一点是安全边界的关键：凭证与执行权在你的代码里
4. 执行结果作为一条 tool result 消息追加进历史
5. 模型基于结果继续（可能再调工具、可能给最终答案）

要点：模型只会"请求"调用；**并行工具调用**（一轮返回多个调用）能显著降延迟；工具调用是否发生取决于模型判断，可用 `tool_choice` 强制。

## 工具的三种类型（OpenAI 分类）

| 类型 | 用途 | 例子 |
|---|---|---|
| **Data** | 取上下文 | 查数据库/CRM、读 PDF、搜网 |
| **Action** | 改世界 | 发邮件、更新 CRM、转人工 |
| **Orchestration** | agent 作为工具 | refund agent、research agent |

按运行位置还可分：**hosted tools**（跑在模型供应商侧：web search、file search、code interpreter、computer use）、**local function tools**（你的进程）、**远程工具**（MCP server）。

## 设计原则（浓缩版）

来自 [[sources/08-writing-effective-tools-for-agents]] 与 [[concepts/20-aci]]：

1. **别把 API 端点逐个包成工具**。agent 的 affordance 与程序不同：`search_contacts` 优于 `list_contacts`，`search_logs` 优于 `read_logs`，`get_customer_context` 优于三个分散的 get
2. **合并高频链路**：`schedule_event` 内部完成"查空档 + 建日程"
3. **命名空间**：`asana_search` / `asana_projects_search`，前缀 vs 后缀式的效果因模型而异，要实测
4. **返回高信号内容**：语义化名称优于 UUID（显著减少幻觉）；用 `response_format: concise|detailed` 让 agent 自选详略
5. **token 效率**：分页、范围、过滤、截断 + 合理默认；Claude Code 默认单次响应上限 25,000 token
6. **错误信息要可行动**：说清怎么改，而不是抛 traceback
7. **描述当 prompt 写**：像给新同事介绍工具；参数用 `user_id` 而非 `user`
8. **Poka-yoke**：改设计让错误不可能发生（例：强制绝对路径）

**工具数量的判据**（OpenAI）：问题不在数量而在**相似/重叠度** —— 有团队管好 15+ 个界限清晰的工具，也有团队栽在 10 个重叠工具上。先改名/改描述，无效再拆多 agent。

## 学术源流

- **MRKL**（2205.00445）：LLM 作路由分发给神经/符号专家模块；核心结论"**知道何时用、如何用工具比工具本身更难**"
- **TALM / Toolformer**（2302.04761）：微调让模型自学调用 API（以"是否提升输出质量"筛选训练样本）
- **HuggingGPT**：任务规划 → 模型选择 → 执行 → 响应生成的四阶段
- **API-Bank**：三级能力评测 —— 会调用 / 会检索工具 / 会规划多次调用

## 规模化问题与解法

工具从 10 个涨到 1000 个后，定义本身撑爆 context：

- **渐进披露**：文件树 + 按需读取，或 `search_tools`（带详略参数）
- **代码执行**：把 MCP server 当代码 API，让 agent 写代码调用 —— 150k → 2k token
- 详见 [[concepts/32-progressive-disclosure-and-code-execution]]

## 安全（不能漏）

- 工具是 **[[concepts/40-excessive-agency]]** 的主要载体：最小化工具数、最小化每个工具的功能、避免开放式工具（`run_shell`、`fetch_url`）、最小权限、**在用户上下文中执行**、高风险动作需审批、授权判定用代码而非让 LLM 决定
- 工具输出是**不可信输入**（[[concepts/39-prompt-injection]]）；MCP server 与工具包属供应链面，要 pin/签名/审计描述中的隐藏指令
- MCP 的 **tool annotations** 用于声明破坏性/开放世界访问

## 与其他概念的关系

- [[concepts/14-augmented-llm]] 的"工具"增强就是这里
- [[concepts/20-aci]] 是设计哲学，本页是机制与清单
- [[concepts/31-mcp]] 是工具的标准化分发协议

## 来源

- [[sources/08-writing-effective-tools-for-agents]]、[[sources/05-openai-practical-guide-to-building-agents]]、[[sources/11-openai-agents-sdk-docs]]、[[sources/04-llm-powered-autonomous-agents]]
