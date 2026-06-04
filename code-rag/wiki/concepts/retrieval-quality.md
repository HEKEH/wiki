---
title: Retrieval Quality
date: 2026-05-05
tags: [retrieval, quality, evaluation]
sources:
  [
    How-to-Chunk-Code-for-RAG.md,
    Effective-Chunking-Strategies-for-RAG.md,
    practical-code-rag-at-scale.md,
    Repoformer-Selective-Retrieval-for-Repository-Level-Code-Completion.md,
    Improve-RAG-Performance-Using-Cohere-Rerank.md,
  ]
---

# Retrieval Quality

RAG 系统中检索结果的相关性和完整性，直接决定生成质量。

## 验证手段

Cohere 的 `co.chat` 提供引用归因功能 — 生成答案时标注每个断言来自哪个 chunk（如 `[1]`），可据此反向验证检索结果是否正确支撑了答案。这对评估分块策略效果尤其有用（[[sources/src-effective-chunking-strategies-for-rag]]）。

## 影响因素

- **分块策略**：[[techniques/semantic-chunking|语义分块]] 优于固定长度切分；[[concepts/content-dependent-splitting|内容依赖切分]] 从根源保证语义完整性
- **Chunk 大小**：过大则信息密度低、相关性稀释；过小则缺乏上下文。最优大小与上下文窗口对齐（[[techniques/code-chunking#Chunk 大小与上下文窗口的关系|实验证据]]）
- **Embedding 质量**：代码 embedding 需要理解语法结构，通用文本模型未必合适
- **Overlap**：[[techniques/chunk-overlap|Chunk overlap]] 补偿切分丢失的上下文，但引入冗余
- **切分方式与文档的匹配度**：内容无关切分对结构化文档（如转录稿）效果差，Cohere 实验证实无 overlap 时发言人信息丢失导致检索失败
- **任务模态**：PL→PL 检索中 BM25 已是最优；NL→PL 需要语义编码（[[concepts/pl-pl-vs-nl-pl-retrieval]]）
- **上下文增强**：NL→PL 场景中，在检索结果的 context 里包含文件路径和 import 语句可显著提升性能 — 为模型提供定位线索（[[sources/src-practical-code-rag-at-scale]]）
- **切分粒度**：词级切分是稀疏检索的最佳粒度，BPE 相对词级约慢 10× 且无质量提升（[[techniques/bm25-retrieval]]）
- **检索决策**：并非所有查询都需要检索——代码补全中仅 ~20% 检索实例提升性能，~20% 反而损害性能（[[concepts/selective-rag]]）
- **重排（两阶段检索）**：第一阶段检索常把最相关文档排在靠后位置（dense 单向量压缩导致信息损失），top-k 截断时漏掉关键信息。第二阶段用 cross-encoder [[techniques/reranking|重排]] 把真正相关的内容顶到前面——提升排序质量而非召回率，是即插即用的检索质量增强（[[sources/src-improve-rag-performance-cohere-rerank]]）

## 延迟-质量权衡

[[sources/src-practical-code-rag-at-scale|Galimzyanov et al. (2025)]] 量化了不同检索配置的延迟差异（最高约 200×；细分数据见 [[techniques/sparse-vs-dense-retrieval]]）：

- IoU + 行级：0.02s/1M symbols（极快但质量略低）
- BM25 + 词级：0.2s（**最佳质量-延迟平衡**，PL→PL 场景推荐）
- E5-large：代码补全 (PL→PL) 约 3.3s/1M symbols；Bug 定位 (NL→PL) 约 2.8s/1M symbols，NDCG 仅略优于 BM25
- **voyage-code-3**（Voyage AI）：NL→PL 质量最高；论文中延迟远高于 BM25 / E5（含全库编码，512 与 32K 变体见 [[techniques/sparse-vs-dense-retrieval]] 实验表）

检索质量不仅关乎准确率，还必须考虑延迟预算 — 实际部署中极大的延迟跨度意味着从毫秒级到秒级甚至分钟级的用户体验差异。

## 相关

- [[techniques/code-chunking]] — 分块是检索质量的基础
- [[techniques/semantic-chunking]] — 语义感知的分块提升检索质量
- [[techniques/chunk-overlap]] — overlap 补偿切分边界的信息丢失
- [[concepts/content-dependent-splitting]] — 内容依赖切分从根源提升检索质量
- [[concepts/selective-rag]] — 跳过不必要的检索是提升检索质量的新维度
- [[techniques/reranking]] — 两阶段检索的第二级精排，修复第一阶段的排序质量
