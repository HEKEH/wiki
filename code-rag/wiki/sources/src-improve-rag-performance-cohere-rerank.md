---
title: Improve RAG Performance Using Cohere Rerank
date: 2026-06-04
tags: [reranking, cross-encoder, cohere, two-stage-retrieval, dense-retrieval, aws-sagemaker, retrieval-quality]
sources: [Improve-RAG-Performance-Using-Cohere-Rerank.md]
---

# Improve RAG Performance Using Cohere Rerank

AWS Machine Learning Blog（Shashi Raina @AWS、Pradeep Prabhakaran @Cohere），2024-09-16。一篇工程实践向文章，论证为什么需要在检索后加一步重排（rerank），并演示在 Amazon SageMaker 上部署 Cohere Rerank 3。

## 核心论点

RAG 编排分两步：**检索（retrieval）** → **接地生成（grounded generation）**。最终答案质量高度依赖检索结果的相关性，但单阶段密集检索常常把最相关的文档排在靠后位置，导致 top-k 截断时漏掉关键信息。**重排是修复这一缺陷的低成本手段**——无需替换或改造现有检索系统，即可为任意关键词/向量检索加上语义增强。

## 为什么密集检索不够

密集检索（dense retrieval）把查询和文档映射到同一向量空间，用余弦相似度/欧氏距离找最近邻。问题在于：

- 文档语义被压缩进单个向量（源文写 **786–1536 维**，其中 786 应为 **768** 笔误——768 是 BERT-base 维度，1536 是 OpenAI ada-002 维度），存在**信息损失**
- 数据/问题越复杂，单向量表示越力不从心
- 结果是：**最相关的信息不一定排在检索结果顶部**

文中实例：查询 "When was the transformer paper coauthored by Aidan Gomez published?"，top-k（k=6）的结果集里确实包含最准确的文档，但它排在**最底部**；若 k=3，正确文档根本不会进入结果。

## 两阶段检索（Two-Stage Retrieval）

搜索工程的经典解法：

1. **第一阶段（retriever / embedding model）**：从大数据集中召回一批候选文档（粗召回，重召回率）
2. **第二阶段（reranker）**：对候选文档逐个相对查询重新打分、重排（精排，重精度）

二者优势互补：第一阶段靠向量空间邻近性快速捕获相关项；重排保证上下文真正相关的结果浮到顶部。

## Cohere Rerank 的工作方式

- 输入：一个 query + 一个文档列表
- 输出：带相关性分数的有序数组（每个文档一个 relevance score）
- **关键区别**：不同于传统 embedding 模型（query 和 document 各自独立编码再算距离），Rerank **把 query 和 document 成对一起送入深度模型**，直接评估两者的对齐程度——这正是 cross-encoder 范式，相关性判断更细致

## Rerank 3 能力

- **4K 上下文长度** — 显著提升长文档的检索质量
- **多维 / 半结构化数据** — 邮件、发票、JSON、代码、表格
- **多语言** — 100+ 种语言
- 更低延迟与更低 TCO

源文结尾另提到 **Rerank 3 Nimble**（低延迟变体），与 Rerank 3 一同在 Amazon SageMaker JumpStart 上可用。

## SageMaker 部署流程（要点）

1. 在 AWS Marketplace 订阅 `Cohere Rerank 3 Model – Multilingual`，记下 Product ARN
2. `pip install --upgrade cohere-aws`，用 `cohere_aws.Client` + 模型包 ARN `create_endpoint`（示例实例 `ml.g5.2xlarge`）
3. 推理调用：

   ```python
   response = co.rerank(
       documents=documents,
       query='What emails have been about returning items?',
       rank_fields=["Title", "Content"],
       top_n=5,
   )
   ```

4. `co.delete_endpoint()` + `co.close()` 清理，避免持续计费

`rank_fields` 支持对半结构化文档指定参与排序的字段；`top_n` 控制返回的精排结果数。示例用了中、英、西、德、阿多语言客服工单，演示多语言重排。

## 关键 takeaway

- 重排是**两阶段检索的第二级**，介于检索与生成之间，过滤噪声、把真正相关的内容顶到前面
- cross-encoder（query-document 联合编码）精度高于 bi-encoder（独立编码），代价是无法预计算、必须在线对每个候选打分
- 重排是**即插即用**的质量提升：不改检索管道即可叠加；同时减少送入 LLM 的 token、降低延迟与幻觉

## 局限 / 注意

- 文章是 Cohere/AWS 的产品向工程教程，缺少独立基准对比（没有给出 nDCG/MRR 等量化提升数字，仅用单个定性例子）
- cross-encoder 重排无法预计算，候选数过大时延迟与成本上升——需控制第一阶段召回的候选规模（通常 50–200）
- 主要面向 NL→PL / NL→NL 等语义检索场景；纯 PL→PL 是否值得重排，本文未涉及（参见 [[concepts/pl-pl-vs-nl-pl-retrieval]]）

## 与 Wiki 其他页面的关系

- [[techniques/reranking]] — 本文贡献的核心技术页
- [[techniques/sparse-vs-dense-retrieval]] — 重排是 Hybrid 检索中 RRF 之外的另一种融合/精排路径
- [[concepts/retrieval-quality]] — 重排作为提升检索质量的独立维度
- [[concepts/pl-pl-vs-nl-pl-retrieval]] — 重排主要受益于 NL→PL 语义检索场景
