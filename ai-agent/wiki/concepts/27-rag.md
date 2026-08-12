---
title: RAG（检索增强生成）
date: 2026-08-11
tags: [rag, retrieval, embeddings, graphrag, papers]
sources: [Foundational-Agent-Papers-Abstracts.md, LLM-Powered-Autonomous-Agents.md, How-We-Built-Our-Multi-Agent-Research-System.md]
---

# RAG（检索增强生成）

Agent 岗面试几乎必问 RAG，因为它是绝大多数生产系统的取上下文手段。要点：**先讲清标准管线，再讲清它在 agent 时代的位置变化**。

## 起点：RAG 原论文（2005.11401）

Lewis et al., 2020：把**参数化记忆**（预训练 seq2seq）与**非参数化记忆**（DPR 稠密向量索引 + 检索器）联合训练，用于知识密集任务。两种形式：整个输出共用同一段检索passage，或每个 token 用不同 passage。

## 标准管线（要能背）

> 说明：本节的分块/混合检索/重排等工程细节属**行业通行实践**，不来自本库 `raw/` 的任何一篇存档；`sources:` 中的论文与博客对应的是 RAG 定义、三代分类、GraphRAG、agentic search 对比与投毒数据。面试引用时注意区分"有出处的结论"与"常规做法"。

**离线（索引）**：加载 → 清洗 → **分块（chunking）** → 向量化（embedding）→ 写入向量库（+ 元数据）

**在线（查询）**：query 改写 → 检索（向量 / 关键词 / 混合）→ **重排（rerank）** → 组装 prompt → 生成 → （可选）引用与校验

每个环节的常见旋钮：
- **分块**：固定长度 + overlap、按结构（标题/段落/代码函数）、语义分块；小块检索准、大块上下文全 → 常用"小块检索 + 返回父块"
- **检索**：稠密向量（语义）+ BM25/关键词（精确术语、专名）→ **混合检索**几乎总更好；用 RRF 融合
- **重排**：cross-encoder / rerank 模型对 top-k 精排，是性价比最高的单点改进
- **query 侧**：多查询改写、HyDE（先生成假想答案再检索）、子问题分解
- **索引侧**：加元数据过滤、加摘要索引、句窗检索

## 三代分类（RAG Survey 2312.10997）

| 代 | 特征 |
|---|---|
| **Naive RAG** | 索引 → 检索 → 生成，一次性 |
| **Advanced RAG** | 检索前后加优化（query 改写、重排、上下文压缩） |
| **Modular RAG** | 可插拔模块与流程编排（含路由、记忆、迭代检索） |

**GraphRAG**（2404.16130）：用 LLM 抽实体关系建知识图 + 层次化社区摘要，解决"整个语料的主题是什么"这类**全局性查询**（本质是 query-focused summarization，纯向量检索天然做不到）。

## Agent 时代：RAG 的位置变了

Anthropic 的明确表述（[[sources/07-multi-agent-research-system]]）：

> 传统 RAG 用**静态检索**：取一批与 query 最相似的 chunk 就生成。我们的架构用**多步搜索**，动态发现相关信息、随发现调整、分析结果后再形成答案。

以及 [[sources/06-effective-context-engineering]]：从"推理前 embedding 预检索"转向 **just-in-time**（只存轻量标识符，运行时用工具取）。

因此现代答法是**三分**：
1. **静态 RAG**：低延迟、语料稳定、答案在少数片段里 → 客服 FAQ、文档问答
2. **Agentic search**（[[concepts/28-agentic-search]]）：需要多跳、需要判断来源质量、语料动态 → 研究、代码库
3. **混合**：稳定核心资料预置进 context（如 `CLAUDE.md`），其余即时检索 —— Claude Code 的做法，也适合法律/金融这类内容不那么动态的领域

## 评估

- 检索侧：recall@k、precision@k、MRR、nDCG
- 生成侧：忠实度（faithfulness，有无幻觉）、答案相关性、**引用准确性**
- 端到端：用 LLM-as-judge 打 rubric（事实准确性、引用准确性、完整性、来源质量、工具效率），见 [[concepts/41-agent-evaluation]]
- 常见坑：只看生成分数，检索坏了却不知道 —— **先单独评检索**

## 安全（OWASP LLM09 / LLM01）

- **向量与 embedding 弱点**（LLM09）：跨租户泄漏、embedding 反演、相似度投毒
- **RAG 语料投毒**：向百万级知识库注入**约 5 篇**投毒文档即可达到约 90% 攻击成功率（W. Zou et al., 2025）；检索内容是**间接注入**的头号投递面
- 缓解：语料写入的来源校验与审计、按用户/租户做检索隔离、检索内容作为**数据通道**标注传入、输出侧 schema 校验

## 与其他概念的关系

- [[concepts/26-memory]] 与 RAG 共用向量检索底座，但目的不同（记忆是自己写的，RAG 是外部知识）
- [[concepts/14-augmented-llm]] 的"检索"增强就是 RAG
- [[concepts/28-agentic-search]] 是 RAG 的 agent 化替代/补充

## 来源

- [[sources/15-foundational-agent-papers]]、[[sources/04-llm-powered-autonomous-agents]]、[[sources/07-multi-agent-research-system]]、[[sources/14-owasp-genai-llm-top-10-2026]]
