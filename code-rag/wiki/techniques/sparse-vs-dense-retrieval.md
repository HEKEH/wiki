---
title: Sparse vs Dense Retrieval
date: 2026-05-07
tags: [sparse-retrieval, dense-retrieval, bm25, embedding, code-search]
sources: [practical-code-rag-at-scale.md, Repoformer-Selective-Retrieval-for-Repository-Level-Code-Completion.md]
---

# Sparse vs Dense Retrieval

检索方法的选择取决于任务模态（PL→PL 还是 NL→PL），不存在万能最优方案。

## 稀疏检索 (Sparse)

基于词项匹配，代表算法：BM25、IoU。

**优势**：速度快（BM25+词级 0.2s/1M symbols）、无需模型推理、实现简单
**劣势**：依赖词汇重叠，跨模态查询（自然语言→代码）效果差

## 密集检索 (Dense)

基于神经网络编码的语义相似度，代表模型：E5、**voyage-code-3**（Voyage AI）等。

**优势**：捕获语义对应，跨模态查询效果强
**劣势**：延迟高（E5-large 在 PL→PL 设定下约 3.3s/1M symbols；**voyage-code-3** 在论文复现设定下远高于此，见下文实验表）、需要 GPU、成本高

### Dense Encoder 实现流程

分**离线索引**和**在线查询**两阶段：

**离线索引**（仓库入库时做一次）：
1. 代码 chunk → Encoder 模型编码 → 向量
2. 向量写入向量数据库（FAISS / ChromaDB / Qdrant / Milvus）
3. chunk 元数据（file_path, imports, raw content）同步存储

**在线查询**（用户提问时）：
1. 用户问题 → 同一个 Encoder 编码 → 查询向量
2. 在向量数据库中做相似度搜索（余弦相似度 / 内积）
3. 返回 top-k 最相关 chunk

### 模型选择

论文在 NL→PL 上的最强 dense 基准对应 **Voyage AI** 的 **voyage-code-3**（论文 PDF 表格中常写作 `Voyager-3-code`，与商用模型 ID **voyage-code-3** 指同一产品线；勿与误写的「Voyager-3-Code」混淆）。工程上还可选用 **E5**、**CodeBERT** 等自部署模型或其它代码 embedding；**不同模型/供应商的向量空间不可混用**。

| 模型 | 部署方式 | 代码检索效果 | 延迟 | 成本 | 输入长度 |
|------|---------|------------|------|------|---------|
| **E5-large** (intfloat/e5-large) | 自部署 | 中 | 慢（需 GPU） | 免费 | 512 tokens |
| **E5-base/small** | 自部署 | 较低 | 较快 | 免费 | 512 tokens |
| **voyage-code-3** | Voyage AI API 等 | NL→PL 最优（论文） | 极高（全库编码） | 按调用计费 | 512 / 32K 等变体 |
| **CodeBERT** | 自部署 | 较低 | 中 | 免费 | 512 tokens |

长上下文变体（如论文中的 **voyage-code-3** 32K 配置）可缓解 E5 类模型的 512 token 截断问题；自研系统可选同类长上下文 code encoder。

### 向量数据库

| 数据库 | 特点 | 适用场景 |
|--------|------|---------|
| **FAISS** | 轻量，纯向量搜索 | 原型验证，单机 |
| **ChromaDB** | Python 原生，自带元数据存储 | 小项目快速起步 |
| **Qdrant** | 支持 filtering + 向量搜索 | 生产环境，需按路径过滤 |
| **Milvus** | 分布式，大规模 | 多仓库、大规模部署 |

### 关键注意事项

- **查询和文档必须用同一个模型** — 例如 E5 编码的向量不能和 **voyage-code-3**（或其它模型）的向量混用
- **input type / 前缀不能搞错** — E5 用 `query:` / `passage:` 前缀；**Voyage** API 用 `input_type`（或文档规定字段）区分 query/document，搞反了质量会大幅下降
- **长文档截断** — E5 最大 512 tokens，超长 chunk 会被截断；**voyage-code-3** 等长上下文模型可缓解
- **预计算 embedding 省延迟** — 论文的延迟数据包含编码全仓库的时间；实际部署中索引阶段已预计算，查询时只需编码用户问题（毫秒级）

## 实验证据

[[sources/src-practical-code-rag-at-scale|Galimzyanov et al. (2025)]] 的关键数据：

**代码补全 (PL→PL)**：

| 配置 | EM @4K / @16K | 延迟 (s/1M symbols) |
|------|-------------|-------------------|
| BM25 + 词级 | 0.55 / 0.60 | 0.2 |
| E5-large (512) | 0.39 / 0.52 | 3.3 |

**Bug 定位 (NL→PL)**：

| 配置 | NDCG (mean) | 延迟 (s/1M symbols) |
|------|-----------|-------------------|
| BM25 + 词级 | 0.574 | 0.07 |
| E5-large (512) | 0.590 | 2.8 |
| voyage-code-3 (512) | **0.717** | 512* |
| voyage-code-3 (32K) | 0.696 | 110* |

*\*论文表中记为 Voyager-3-code；延迟极高因需批量编码全仓库，且 32K 配置受 token 预算约束需小 batch size*

## 核心洞察

**模态驱动分界**：PL→PL 查询与目标有大量词汇重叠，稀疏评分天然适配；NL→PL 查询需要语义桥接，密集编码更有效。

**Python 的例外**：BM25 在 Python 仓库 Bug 定位上与 E5-large 差距缩小，因代码中自然语言痕迹（注释、docstring）更丰富，降低了跨模态鸿沟。

## 实践建议

- 代码到代码检索 → BM25 + 词级切分（快且准）
- 自然语言到代码检索 → Dense encoder（准但慢）
- 速度敏感 + NL→PL → BM25 仍是可用基线
- 追求极致 NL→PL 精度 → 论文基准为 **voyage-code-3**；落地时可用同等能力的 code dense encoder

## Hybrid 混合检索

对 NL→PL 主导的项目（如仓库代码问答），推荐 Dense 主检索 + BM25 补充的混合架构。先判别查询类型，再选主排序路径（避免误读为「单次查询并行跑两套完整排序」而不做融合语义）：

```
判别查询类型
  ├─ 自然语言 → Dense Encoder（如 E5-large、**voyage-code-3** 或等价 code embedding）主排序
  └─ 代码片段 → BM25 + 词级切分 主排序
      ↓
  （可选）另一路作补充召回 → Reciprocal Rank Fusion 合并 →（可选）[[techniques/reranking|cross-encoder 重排]] → 填入 LLM 上下文
```

混合检索的优势：Dense 捕获语义对应（"登录失败" → `AuthException`），BM25 捕获精确标识符匹配（`validateToken` → `validateToken`），两者互补。

**RRF vs 重排**：RRF 是无模型的排名融合（只看名次），适合合并异构检索源；[[techniques/reranking|重排]] 是有模型的语义精排（cross-encoder 重新联合编码 query+doc），可叠加在 RRF 之后做最终精排。NL→PL 场景下重排收益最大。

Python 仓库可简化：BM25 在 Python NL→PL 场景下与 E5-large 差距极小 (NDCG 0.64 vs 0.61)，成本敏感场景下 BM25 单路可能够用。

## 选择性检索与稀疏检索

[[sources/src-repoformer-selective-retrieval|Wu et al. (2024)]] 发现，在使用 Jaccard 稀疏检索的仓库级代码补全中，仅 ~20% 的检索实例能提升模型性能。这为稀疏检索增加了一个效率维度——通过 [[concepts/selective-rag|选择性检索]]，可在保持稀疏检索速度优势的同时，进一步减少不必要的检索调用。Repoformer 也验证了其框架与 dense retrieval 的兼容性。

## 相关

- [[techniques/bm25-retrieval]] — 稀疏检索的详细分析
- [[techniques/reranking]] — 第一阶段检索后的 cross-encoder 精排
- [[concepts/pl-pl-vs-nl-pl-retrieval]] — 任务模态的本质差异
- [[concepts/retrieval-quality]] — 检索质量的多维评估
