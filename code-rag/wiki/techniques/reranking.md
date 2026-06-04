---
title: Reranking (重排)
date: 2026-06-04
tags: [reranking, cross-encoder, bi-encoder, two-stage-retrieval, cohere, retrieval-quality]
sources: [Improve-RAG-Performance-Using-Cohere-Rerank.md]
---

# Reranking（重排）

重排是 RAG 检索管道的**第二级精排**：第一阶段检索器（BM25 或 dense encoder）粗召回一批候选文档，重排模型对这些候选**相对查询重新打分并重新排序**，把真正相关的内容顶到前面，再送入 LLM 生成。

## 为什么需要重排

第一阶段检索的固有缺陷：

- **Dense 检索的信息压缩** — 文档语义被压进单个向量（768–1536 维），存在信息损失，最相关文档常常**不在 top 位置**（[[techniques/sparse-vs-dense-retrieval]]）
- **BM25 的词汇局限** — 纯词项匹配无法理解语义改写
- **top-k 截断风险** — 若正确文档排在第 6 位而 k=3，它根本进不了上下文

重排修复的是**排序质量**而非召回率：第一阶段保召回（宁可多召回），重排保精度（把对的排前面）。这就是**两阶段检索（two-stage retrieval）**范式。

```
查询
 ├─ 第一阶段 retriever（BM25 / dense encoder）→ 粗召回 50–200 候选（重召回率）
 └─ 第二阶段 reranker（cross-encoder）→ 对候选逐一相对 query 打分重排（重精度）
       ↓
   取 top_n 填入 LLM 上下文 → grounded generation
```

## Cross-Encoder vs Bi-Encoder

重排质量的关键在于 **cross-encoder** 架构，这也是它区别于 embedding 检索的本质：

| | Bi-Encoder（embedding 检索） | Cross-Encoder（重排） |
|---|---|---|
| 编码方式 | query 和 document **各自独立**编码成向量 | query 和 document **成对一起**送入模型 |
| 相关性 | 事后算向量距离（余弦/内积） | 模型直接输出 relevance score |
| 精度 | 中（受单向量压缩限制） | 高（联合注意力，捕获细粒度对齐） |
| 可预计算 | ✅ document 向量可离线建索引 | ❌ 必须在线对每个 (query, doc) 对打分 |
| 延迟/成本 | 低（查询时只编码 query） | 高（候选数 × 一次前向） |
| 适用阶段 | 第一阶段大规模召回 | 第二阶段小规模精排 |

正因 cross-encoder 无法预计算，它**只能用于精排**少量候选——不能拿来扫全库。这也解释了为什么要先用 bi-encoder/BM25 把候选缩到几百个，再交给重排。

## 工具与实现

### Cohere Rerank（托管 API / SageMaker）

最常见的即插即用方案。输入 query + 文档列表，输出带分数的有序数组：

```python
response = co.rerank(
    documents=documents,      # 第一阶段召回的候选
    query="What emails have been about returning items?",
    rank_fields=["Title", "Content"],  # 半结构化文档可指定参与排序的字段
    top_n=5,                  # 返回精排后的前 5 个
)
```

- **Rerank 3** 能力：4K 上下文、100+ 语言、半结构化数据（邮件/JSON/代码/表格）
- 可在 Cohere Hosted API 或 Amazon SageMaker（如 `ml.g5.2xlarge` 实例）部署
- 即插即用：**不改第一阶段检索管道**即可叠加（[[sources/src-improve-rag-performance-cohere-rerank]]）

### 其他重排路径

> 以下为补充知识（非本次 ingest 的 Cohere 源文内容），其中 RankGPT 是待 ingest 的候选源。

- **开源 cross-encoder** — sentence-transformers 的 `cross-encoder/ms-marco-*` 系列，可自部署
- **LLM-as-reranker** — 直接用 GPT/Claude 等生成式模型做相关性排序（如 RankGPT，EMNLP 2023 杰出论文，arXiv 2304.09542）；零样本即可达到甚至超过监督方法，代价是延迟和 token 成本更高
- **FlashRank** 等轻量库 — 面向低延迟场景的小型重排模型

## 重排 vs RRF（两种"精排/融合"路径）

[[techniques/sparse-vs-dense-retrieval]] 中的 Hybrid 检索用 **Reciprocal Rank Fusion (RRF)** 合并多路结果——RRF 是**无模型的排名融合**（只看名次，不重新理解内容）。重排则是**有模型的语义精排**（cross-encoder 重新读 query+doc）。二者可叠加：

```
Dense 召回 ┐
           ├─ RRF 融合多路 → cross-encoder 重排 → top_n → LLM
BM25 召回 ┘
```

- **RRF**：快、无额外模型，适合融合异构检索源
- **Reranking**：慢、需模型，适合在融合后做最终精排冲精度

## 何时值得加重排

- ✅ **NL→PL / NL→NL 语义检索** — 查询与文档存在语义改写、跨模态鸿沟时收益最大（[[concepts/pl-pl-vs-nl-pl-retrieval]]）
- ✅ **第一阶段召回噪声大 / top-k 截断激进** — 重排能把漏排的相关文档救回 top
- ✅ **对生成质量敏感、可接受额外延迟** — 减少送入 LLM 的无关 token、降低幻觉
- ⚠️ **纯 PL→PL（代码补全）** — BM25 已是最优且词汇高度重叠，重排收益存疑、延迟不划算（本源未涉及，属开放问题）
- ⚠️ **候选规模过大** — cross-encoder 延迟随候选数线性增长，需先收窄第一阶段召回（通常 50–200）

## 收益与代价

**收益**：提升 top 位置相关性 → 更准的 grounded generation；减少上下文 token、降延迟与幻觉；即插即用不动现有检索。

**代价**：cross-encoder 无法预计算，在线逐对打分；候选多则延迟/成本上升；托管 API（如 Cohere）按调用计费。

## 相关

- [[sources/src-improve-rag-performance-cohere-rerank]] — Cohere Rerank 工程实践来源
- [[techniques/sparse-vs-dense-retrieval]] — 第一阶段检索器；RRF vs 重排
- [[techniques/bm25-retrieval]] — 稀疏召回作为重排的第一阶段
- [[concepts/retrieval-quality]] — 重排作为提升检索质量的独立维度
- [[concepts/pl-pl-vs-nl-pl-retrieval]] — 重排主要受益于 NL→PL 场景
