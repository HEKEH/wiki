---
title: "OWASP GenAI LLM Top 10 (2026)"
date: 2026-08-11
tags: [source, security, owasp, prompt-injection, excessive-agency]
sources: [OWASP-GenAI-LLM-Top-10-2026.md]
status: ingested
---

# OWASP GenAI LLM Top 10（2026 版）

- 来源：[[entities/13-owasp-genai-security-project]]，2026-08-04 发布（项目负责人 Steve Wilson、Rock Lambros）
- 原文：[raw/OWASP-GenAI-LLM-Top-10-2026.md](../../raw/OWASP-GenAI-LLM-Top-10-2026.md)（LLM00 前言 + LLM01–LLM10 全文）

安全是 agent 面试的**必考短板项**。这份榜单是业界公认的检查清单，2026 版的排序变化本身就说明了行业风向。

## 2026 榜单与排名变化

| 编号 | 条目 | 变化 |
|---|---|---|
| LLM01 | **Prompt Injection** | 保持第一 |
| LLM02 | Sensitive Information Disclosure | 保持第二（信念与证据最一致的一条） |
| LLM03 | **Excessive Agency** | **↑ 升到第三（本次最重要的移动）** |
| LLM04 | Supply Chain | 新增"被推广的模型工件名不副实"的信任失效 |
| LLM05 | Data and Model Poisoning | 吸收了 fine-tuning 破坏 |
| LLM06 | Unbounded Consumption | **↑ 上升 4 位** |
| LLM07 | Misinformation | — |
| LLM08 | **Hidden Context Exposure** | 原 System Prompt Leakage **改名扩围** |
| LLM09 | Vector and Embedding Weaknesses | — |
| LLM10 | Improper Output Handling | **↓ 从第 5 跌到第 10**（含 AI 生成不安全代码） |

> Excessive Agency 升到第三，因为投票与事故记录一致表明：**损害正落在 agentic 部署上**。

**边界声明（很重要）**：这份榜单管的是"模型作为应用中的一个组件"时的风险；一旦模型变成**行动者**（能调工具、跨会话带记忆、在下游引发后果），风险归 **OWASP Agentic Top 10（ASI 系列）**。两份要配合使用。文中出现的 ASI 条目：ASI02 Tool Misuse、ASI03 Identity & Privilege Abuse、ASI04 Agentic Supply Chain、ASI08 Cascading Failures。

## LLM01 Prompt Injection：为什么无法根治

- 根因：**LLM 在架构上不区分"指令"与"数据"**（都是同一 token 流），因此没有"参数化查询"那样的干净解法（NCSC 2025）
- 三个部署期性质让问题恶化：
  1. **context-window pooling**：system prompt、用户输入、检索文档、工具输出、历史、记忆共享一条 token 流，没有强制的信任边界
  2. **memory persistence**：一次注入写进长期记忆 / RAG 语料 / 向量库，会污染此后**每一个**读取它的会话
  3. **agentic execution**：模型输出驱动工具调用（文件系统、shell、邮件、云 API、MCP server、子 agent），爆炸半径从聊天窗扩大到工具可达之处，且工具输出会再次进入 context，形成链式甚至自我复制效应
- **注入解剖三轴**（威胁建模时先分解）：**投递面**（直接输入 / 检索内容 / 工具输出 / 工具连接通道 / 持久记忆）、**传播行为**（单发 / 多步 kill-chain / 经记忆或 RAG 跨会话 / 跨 agent 自我复制）、**编码**（明文 / base64 等混淆 / 不可见 Unicode / 多模态隐写 / 低资源语言）
- 间接注入按可信度分三层：**不可信面**（公开网页、陌生邮件、搜索结果）、**半可信面**（issue 标题、包 README、第三方 API 响应）、**可信面**（自己的仓库、数据库、内部文档 —— 攻击者可能通过公开反馈表单把内容送进来）
- 当代最危险的模式：攻击者把文本放进"用户信任的位置"，等用户的 MCP 连接 agent **以用户的高权限凭证**去读它 —— 执行特权动作的是 agent，不是攻击者

**缓解（防御是架构性的，不是拦截式的）**：核心 11 条见 [[concepts/39-prompt-injection]]，其中对 agentic 部署"承重"的两条是 **least privilege（#4：凭证与状态变更能力放在应用代码里，经确定性策略引擎在执行时复核意图与参数）** 和 **capability budgeting（#8：Meta 的 Rule of Two 作为下限）**。还要注意 **#9 把 agent 的记忆写入当作特权操作**、**#11 用"读过你防御方案的自适应攻击者"来测试**（Nasr 等 2025：12 种近期防御在静态攻击下成功率近 0，自适应攻击下多数 >90%）。

## LLM03 Excessive Agency：agent 时代的核心条目

三大根因：**excessive functionality（功能过多）**、**excessive permissions（权限过大）**、**excessive autonomy（自治过度）**。

七条根本控制 + 两条限损控制，见 [[concepts/40-excessive-agency]]。特别值得记的两条：
- **Execute tools in user's context**：在委派/多 agent 链路中**保留原始用户上下文与授权范围**，不能只依赖调用方 agent 或服务身份的权限
- **Complete mediation**：授权用代码判定，不能让 LLM 决定动作是否允许；采用**分级执行策略（audit → warn → block → escalate）**，可逆低风险自动放行、不可逆高风险转人工（例：退款走店铺积分可自动，外部打款需人工）

## LLM08 Hidden Context Exposure

从"system prompt 泄露"扩展为更一般的框架：**不该被触及的信息被信任并暴露**。对 agent 而言包括推理链内容、工具描述、隐藏在 context 中的凭证与内部约定。

## 怎么用这份榜单准备面试

1. 记住 LLM01/LLM03/LLM08 三条与 agent 最相关的
2. 记住两个可引用的结构化判据：**lethal trifecta**（[[sources/16-lethal-trifecta]]）与 **Rule of Two**
3. 记住一句立场：**"假设指令边界终将被突破，去约束模型被允许做什么、以及它的输出被允许触达什么"**
