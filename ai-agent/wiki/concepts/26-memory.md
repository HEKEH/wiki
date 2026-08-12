---
title: Agent 记忆系统（Memory）
date: 2026-08-11
tags: [memory, context, memgpt, sessions, rag]
sources: [LLM-Powered-Autonomous-Agents.md, Effective-Context-Engineering-for-AI-Agents.md, OpenAI-Agents-SDK-Documentation.md, Foundational-Agent-Papers-Abstracts.md]
---

# Agent 记忆系统

"记忆"在面试里最容易答得含糊。稳妥的答法是**先分层，再分实现**。

## 分层（Lilian Weng 的人脑类比）

| 人类记忆 | Agent 对应 |
|---|---|
| 感觉记忆 | 原始输入的 embedding 表示 |
| 短期 / 工作记忆 | **in-context learning** —— 受 context window 限制 |
| 长期记忆（显式/隐式） | 外部存储 + 检索 |

工程上更实用的三分：

1. **会话内记忆（短期）**：消息历史本身。策略是裁剪、压实、清理工具结果
2. **跨会话记忆（长期）**：用户偏好、项目状态、既往结论。载体是数据库/文件/向量库
3. **程序性记忆（技能）**：把跑通的做法存成可复用脚本或 Skill（Voyager 的技能库、MCP code execution 里的 `./skills/*.ts`）

## 四类实现（按"状态放在哪"）

以 OpenAI Agents SDK 的分类为骨架，这张表可直接用于回答"多轮会话怎么存"：

| 策略 | 状态位置 | 适合 |
|---|---|---|
| 手动传历史（`to_input_list()`） | 应用内存 | 小型循环、完全手控、任意供应商 |
| Session 对象 | 你的存储 + SDK | 需持久化、可恢复运行 |
| 服务端会话 ID（`conversation_id`） | 供应商侧 | 跨 worker/服务共享 |
| 上一响应 ID（`previous_response_id`） | 供应商侧 | 轻量续接 |

**一个会话只选一种**，client-managed 与 server-managed 混用会导致上下文重复。

## 长期记忆的两条技术路线

### 路线 A：向量检索（经典）

写入 embedding 到向量库，用 **MIPS（最大内积搜索）** 取回，实践中用 ANN 近似：
- **LSH**（局部敏感哈希）、**ANNOY**（随机投影树）、**HNSW**（分层小世界图）、**FAISS**（向量量化 + 聚类）、**ScaNN**（各向异性量化）
- 这是 [[concepts/27-rag]] 与"agent 长期记忆"共用的底座

局限：检索到的是**片段**，表达力不如 full attention；相似度 ≠ 相关性；写入即固化，更新与遗忘难做。

### 路线 B：文件系统 / 结构化笔记（现代 agent 更常用）

- **结构化笔记（agentic memory）**：agent 主动把要点写到 context 之外（`NOTES.md`、to-do、progress file），需要时读回。Claude 玩 Pokémon 跨数千步维持计数、自画地图就是这个机制
- **Anthropic memory tool**（Claude 平台 public beta）：基于文件的记忆存取
- 优势：可读可审计、天然支持更新与删除、路径与命名本身携带元数据信号（[[concepts/28-agentic-search]]）

### MemGPT（2310.08560）：把 OS 思路搬过来

- 类比操作系统的**分层内存与虚拟内存分页**：主上下文（快）↔ 外部存储（慢）之间自主换页
- 用 **interrupt** 管理自身与用户之间的控制流
- 面试价值：一句话说明"context window 是内存不是硬盘"，记忆管理本质是**分页与置换策略**

## Generative Agents 的检索三因子

取回记忆时按 **relevance（相关性）+ recency（新近性）+ importance（重要性，直接问 LLM 打分）** 综合排序 —— 比纯向量相似度更接近"人的回忆"。见 [[concepts/44-generative-agents]]。

## 安全：记忆是被低估的攻击面

- 一次注入写进长期记忆 / RAG 语料，会**污染此后每一个读取它的会话**（OWASP LLM01 的 memory persistence）
- 缓解：**把 agent 的记忆写入当作特权操作** —— 记录导致写入的 prompt、对写入内容做"是否含指令/角色修改"分类、跨会话持久化前需审批
- 参见 [[concepts/39-prompt-injection]]

## 与其他概念的关系

- [[concepts/01-session]] 是记忆的事件日志载体
- [[concepts/30-compaction-and-note-taking]] 是短期记忆溢出时的三种技术
- [[concepts/06-context-anxiety]] 是记忆不足时的模型行为副作用

## 来源

- [[sources/04-llm-powered-autonomous-agents]]、[[sources/06-effective-context-engineering]]、[[sources/11-openai-agents-sdk-docs]]、[[sources/15-foundational-agent-papers]]
