---
title: "Writing Effective Tools for AI Agents (Anthropic)"
date: 2026-08-11
tags: [source, anthropic, tools, aci, evaluation]
sources: [Writing-Effective-Tools-for-AI-Agents.md]
status: ingested
---

# Writing Effective Tools for AI Agents—Using AI Agents

- 作者：Ken Aizawa 等（[[entities/01-anthropic]]）
- 原文：[raw/Writing-Effective-Tools-for-AI-Agents.md](../../raw/Writing-Effective-Tools-for-AI-Agents.md)

[[concepts/20-aci]] 的实操版。如果面试问"你怎么给 agent 设计工具/MCP server"，这篇是标准答案的来源。

## 工具是一种新的软件契约

传统函数是**确定性系统之间**的契约：`getWeather("NYC")` 每次行为一致。工具是**确定性系统与非确定性 agent 之间**的契约：用户问"今天要带伞吗"，agent 可能调天气工具、可能凭常识回答、可能先反问位置，偶尔还会幻觉或用错工具。

> 不能像给开发者写 API 那样写工具，要**为 agent 而设计**。

好消息：对 agent 最"人体工学"的工具，通常人类也觉得直观。

## 工作流：原型 → 评估 → 与 agent 协作改进

1. **原型**：把工具包进本地 MCP server 或 DXT，在 Claude Code / Desktop 里连上手测；给 Claude 提供依赖库的 `llms.txt` 文档能一次成型
2. **评估**（[[concepts/41-agent-evaluation]]）
   - 生成大量**贴近真实使用**的任务，基于真实数据源与服务，避免过于简单的沙盒；强任务往往需要**多次甚至数十次**工具调用
   - 好任务 vs 坏任务的对比很值得记：
     - 好：「客户 9182 报告一次购买被扣款三次，找出所有相关日志并判断是否影响其他客户」
     - 坏：「在支付日志里搜索 `purchase_complete` 且 `customer_id=9182`」——已经把解法写在题目里了
   - 每个 prompt 配可验证的响应/结果；验证器可以是精确匹配也可以是 LLM 裁判；**别写过严的验证器**（格式、标点、等价表述差异不该判错）
   - 用**直接 API 调用 + 简单 while 循环**跑 eval，一个循环一个任务；在 system prompt 里要求输出 reasoning 与 feedback 块（放在工具调用之前可触发 CoT，提升有效智能），或直接开 interleaved thinking
   - 除准确率外收集：单次工具调用与任务的总耗时、工具调用次数、token 消耗、工具错误
3. **让 agent 改工具**：把 eval 轨迹拼接后交给 Claude Code，它擅长分析 transcript 并批量重构工具（保持实现与描述自洽）。注意**agent 反馈中"没说的"往往比"说了的"更重要**；要读原始 transcript，别只读 CoT。他们用 held-out 测试集防过拟合，结果显示即使是"专家手写"的工具仍有提升空间

## 五条写工具的原则

### 1. 选对要实现的工具（不是越多越好）

常见错误：**把已有 API 端点逐个包成工具**。agent 的"affordance"与传统软件不同：计算机内存廉价，agent 的 context 有限。要在通讯录里找人，程序可以逐条遍历，agent 逐 token 读完所有联系人就是在浪费 context（相当于从第一页开始翻找）。

- 做 `search_contacts` / `message_contact`，而不是 `list_contacts`
- 工具可以**合并**多个操作：与其 `list_users` + `list_events` + `create_event`，不如 `schedule_event`（内部找空档并建日程）
- 与其 `read_logs`，不如 `search_logs`（只返回相关行 + 少量上下文）
- 与其 `get_customer_by_id` + `list_transactions` + `list_notes`，不如 `get_customer_context`
- 目的：让 agent 像人一样切分任务，同时**削减中间输出消耗的 context**

### 2. 命名空间（namespacing）

agent 可能接入几十个 MCP server、几百个工具。按服务前缀（`asana_search`、`jira_search`）和资源（`asana_projects_search`、`asana_users_search`）分组，划清边界。前缀 vs 后缀式命名对评测有非平凡影响，且**因模型而异，要自己测**。

### 3. 返回有意义的 context

- 只回高信号信息，优先**上下文相关性**而非灵活性；避免低层技术标识符（`uuid`、`256px_image_url`、`mime_type`），偏向 `name`、`image_url`、`file_type`
- **把任意 UUID 解析成语义化名称（甚至 0 起索引的 ID）能显著提升检索精度、减少幻觉**
- 需要两者兼顾时，暴露 `response_format` 枚举（`"concise"` / `"detailed"`）让 agent 自己选详略 —— 例子里 concise 只用约 ⅓ token
- 响应结构（XML / JSON / Markdown）会影响性能，**没有万能解**，要按任务实测（LLM 更擅长与训练数据相似的格式）

### 4. 为 token 效率优化

- 组合使用分页、范围选择、过滤、截断，并给出合理默认值。**Claude Code 默认把工具响应限制在 25,000 token**
- 截断时要给 agent 有用的指引（例如鼓励"多次小范围精确搜索"而不是"一次大搜索"）
- **错误响应要 prompt-engineer**：说明具体、可行动的改进方式，而不是不透明的错误码或 traceback

### 5. 为工具描述做 prompt 工程

> 想象你在向团队新人介绍这个工具。

把你隐含的上下文（专用查询格式、小众术语定义、资源之间的关系）显式写出来；用严格数据模型约束输入输出；参数命名无歧义（不要 `user`，要 `user_id`）。**Claude Sonnet 3.5 正是在精修工具描述后拿下 SWE-bench Verified 的 SOTA**，错误率大幅下降。

MCP 场景下别忘了 **tool annotations**：声明哪些工具需要开放世界访问、哪些会做破坏性变更（与 [[concepts/40-excessive-agency]] 直接相关）。

## 一句话总结

> 有效的工具：定义清晰而有意图、审慎使用 agent 的 context、能组合进多样工作流、让 agent 直觉地解决真实任务。
