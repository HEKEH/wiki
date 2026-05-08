---
title: BM25 Retrieval
date: 2026-05-07
tags: [bm25, sparse-retrieval, code-search, pl-pl]
sources: [practical-code-rag-at-scale.md]
---

# BM25 Retrieval

BM25 (Okapi BM25) 是**检索评分算法**，用于衡量查询与候选代码片段的相似度，然后按分数排序返回最相关的块。它**不是**分块方法 — 分块（Chunker）和切分（Splitter）是上游独立步骤。

## 在检索管道中的位置

```
Chunker（分块器）→ Splitter（切分器）→ Scorer（评分器）
  把文件切成块        把块拆成词项        计算查询与块的相关性
  如：32行/块         如：词级切分         如：BM25
```

BM25 在最后一步：接收 Splitter 产出的词项，计算相关性分数。

## 核心原理

BM25 基于查询与文档的词项重叠度评分，考虑词频 (TF)、逆文档频率 (IDF) 和文档长度归一化。相比简单的 IoU（集合交集），BM25 对高频词降权、对长文档惩罚，更适合代码搜索。

评分公式：

```
score(D, Q) = Σ IDF(qi) × (f(qi, D) × (k1 + 1)) / (f(qi, D) + k1 × (1 - b + b × |D| / avgdl))
```

- `f(qi, D)` — 词项 qi 在文档 D 中的词频
- `|D|` / `avgdl` — 文档长度 / 平均文档长度（长度归一化）
- `k1`、`b` — 调参参数（典型默认 k1=1.2, b=0.75）

实现流程：建立倒排索引（词项 → 文档列表 + 词频）→ 查询时在倒排索引中查找包含查询词的文档 → 评分排序。

## 第三方库

| 库 | 语言 | 特点 | 适用场景 |
|---|------|------|---------|
| **rank-bm25** | Python | 最简单，pip install 即用 | 原型验证，小规模 |
| **Elasticsearch / OpenSearch** | Java | 生产级，分布式，内置 BM25 | 大规模生产环境 |
| **Lucene** | Java | ES 底层引擎 | Java 生态直接使用 |
| **Whoosh** | Python | 纯 Python 全文检索 | 轻量级需求 |
| **Meilisearch / Typesense** | Go/Rust | 轻量级搜索引擎 | 中小规模快速部署 |
| **Tantivy** | Rust | 高性能，类似 Lucene 的 Rust 实现 | Rust 生态 / 高性能需求 |

## 在代码 RAG 中的表现

[[sources/src-practical-code-rag-at-scale|Galimzyanov et al. (2025)]] 在 Long Code Arena 上的实验：

- **代码补全 (PL→PL)**：BM25 + 词级切分比 dense encoder (E5-large) 高 ~10 p.p. EM，延迟低 ~14 倍
- **Bug 定位 (NL→PL)**：BM25 NDCG 0.57 vs **voyage-code-3** 0.72，稀疏检索在跨模态场景下劣势明显
- **Python 仓库**：BM25 在 Bug 定位上与 dense 差距缩小 (0.64 vs 0.61)，因 Python 代码中自然语言注释更丰富

## 切分粒度选择

BM25 + 词级切分与 BM25 + BPE 切分在准确率上统计无差异，但词级快 9 倍。论文中"行级"数据来自 **IoU + 行级**（BM25 未单独报告行级结果），IoU+行级比 BM25+词级低 ~4 p.p. EM 但延迟最低。

| 配置 (Scorer + Splitter) | 延迟 (s/1M symbols) | EM @4K | EM @16K | 备注 |
|---------|------|------|------|------|
| IoU + 行级 | 0.02 | 0.46 | 0.57 | 极低延迟基线，质量有损 |
| BM25 + 词级 | 0.2 | 0.55 | 0.60 | **默认推荐**，最佳质量-延迟平衡 |
| BM25 + BPE/Token | 2.0 | 0.50 | 0.59 | 比 BM25+词级慢 10×，质量无提升 |

BPE 更慢的原因：BPE 产生更大的 posting list 增加计算开销。词级切分是稀疏检索的实用最优解。

## 与其他检索方法的对比

- [[techniques/sparse-vs-dense-retrieval]] — 稀疏 vs 密集检索的任务依赖性
- [[concepts/pl-pl-vs-nl-pl-retrieval]] — 任务模态决定 BM25 的适用性
- [[concepts/retrieval-quality]] — BM25 是代码检索的强基线
