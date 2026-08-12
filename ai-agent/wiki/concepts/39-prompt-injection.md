---
title: 提示注入（Prompt Injection）
date: 2026-08-11
tags: [security, prompt-injection, owasp, lethal-trifecta]
sources: [OWASP-GenAI-LLM-Top-10-2026.md, The-Lethal-Trifecta-for-AI-Agents.md, Foundational-Agent-Papers-Abstracts.md]
---

# 提示注入（Prompt Injection）

OWASP GenAI LLM Top 10 的**第一名**（2026 年仍保持）。Agent 面试里安全部分的核心，答不好会被认为"没做过生产系统"。

## 根因：架构层面无法区分指令与数据

> LLM 在架构上不区分"指令"与"数据"（两者都是同一 token 流上的 token），因此不存在类似参数化查询的干净解法。

Simon Willison 的补充：LLM **无法可靠地按来源区分指令的重要性** —— 一切最终被拼成一串 token。这也是他仿照 SQL injection 命名该问题的原因。

**术语辨析（易错）**：prompt injection ≠ jailbreaking。前者是"可信与不可信内容混在同一 context"的**应用架构问题**；后者是骗模型违反安全策略的**模型安全问题**。把两者混淆的开发者常误以为这与自己无关。

## 三个让 agent 场景恶化的性质

1. **context-window pooling**：system prompt、用户输入、检索文档、工具输出、历史、记忆共享一条 token 流，**没有强制信任边界**
2. **memory persistence**：一次注入写进长期记忆 / RAG 语料 / 向量库，会污染此后**每一个**读取它的会话
3. **agentic execution**：模型输出驱动工具调用（文件系统、shell、邮件、云 API、MCP server、子 agent），爆炸半径从聊天窗扩到工具可达之处；工具输出又重新进入 context，形成**链式甚至自我复制**效应

## 注入解剖三轴（威胁建模先做这一步）

- **投递面**：直接输入 / 检索内容 / 工具输出 / 工具连接通道 / 持久记忆
- **传播行为**：单发 / 多步 kill-chain / 经记忆或 RAG 跨会话 / 跨 agent 自我复制
- **编码**：明文 / base64 等混淆 / 不可见 Unicode / 多模态隐写 / 低资源语言

## 直接 vs 间接

- **直接注入**：用户（或拿到用户访问路径的攻击者）自己提供的输入改变行为。可以是有意（越狱）也可以是**无意**（粘贴的内容里恰好含冲突指令）
- **间接注入**（2302.12173 开山）：模型摄入外部内容（网页、文档、邮件、工具响应、RAG 段落、图片、MCP server 输出、数据库行、issue 标题），其中含指令，**用户从未提供也未看到**。按可信度分三层：
  - **不可信面**：公开网页、陌生邮件、搜索结果
  - **半可信面**：公开 issue 标题、包 README/changelog、第三方 API 响应
  - **可信面**：自己的仓库、数据库、内部文档 —— 攻击者可能通过公开反馈表单把内容送进来

**当代最危险的模式**：攻击者把文本放进"用户信任的位置"，等用户的 MCP 连接 agent **以用户的高权限凭证**去读它 —— 执行特权动作的是 agent，不是攻击者。

## 两个判据（面试直接引用）

### Lethal Trifecta（Simon Willison）

同时具备三者即高危：**① 访问私有数据 ② 暴露于不可信内容 ③ 对外通信能力**。**移除任意一条即破除条件**。注意"对外通信"极易被忽略：任何能发 HTTP 请求的工具（调 API、加载图片、甚至生成一个用户会点的链接）都是外传通道。

### Rule of Two（Meta，OWASP 采纳为下限）

把 (A) 不可信输入、(B) 敏感数据、(C) 状态变更或对外通信 三者同时具备视为高风险：**[A,B,C] 必须逐动作人工审批**；[A,B] 或 [A,C] 需明确的残余风险评估。局限：它对**自治深度**没有约束。

## 缓解（架构性 > 拦截性）

OWASP 的立场：**没有可靠的预防机制**（与 NIST 2025、NCSC 2025 一致）。要**假设指令边界终将被突破**，去约束模型被允许做什么、以及它的输出被允许触达什么。11 条控制里对 agent 承重的是 #4 与 #8：

1. 在 system prompt 里用**声明式 allow/deny** 限定角色与能力（部分控制，能被推断绕过）
2. 定义严格输出 schema 并**在可信应用代码里结构化校验**（抓格式，抓不住语义）
3. **在每个模态边界过滤**（文本/图像/音频/结构化数据），OCR、转写后再过文本过滤器
4. **凭证与状态变更能力放在应用代码而非模型里，逐操作最小权限**；特权调用走**确定性策略引擎**在执行时复核意图与参数 ⭐
5. 在每个 ingest 与 render 边界**剥离不可见字符**（tag block U+E0000–E007F、variation selector U+FE00–FE0F、零宽 U+200B/C/D、U+2060）
6. 外部内容走**结构上独立、带来源标注的通道**（StruQ 类方案；自适应攻击下已被绕过）
7. **特权/不可逆/对外可见动作需人工确认**，且要展示**真实将执行的动作**而非摘要
8. 用 **Rule of Two 做能力预算下限** ⭐
9. **把记忆写入当作特权操作**：记录导致写入的 prompt、分类是否含指令/角色修改、跨会话持久化前审批
10. **pin/签名/校验每个 MCP server 与第三方工具包**，审计工具描述里的隐藏指令，监控工具组成
11. **用读过你防御方案的自适应攻击者做测试**，拒绝只报静态攻击成功率（先用 AgentDojo、JailbreakBench 打基线）

## 与其他概念的关系

- [[concepts/40-excessive-agency]]：注入是输入侧妥协，过度授权决定它有多大后果
- [[concepts/37-guardrails]]：降低发生率的一层，不能作为唯一依赖
- [[concepts/27-rag]]：RAG 语料投毒（约 5 篇文档 → 约 90% 成功率）
- [[concepts/31-mcp]]：MCP 鼓励混搭工具，直接放大三要素

## 来源

- [[sources/14-owasp-genai-llm-top-10-2026]]、[[sources/16-lethal-trifecta]]、[[sources/15-foundational-agent-papers]]
