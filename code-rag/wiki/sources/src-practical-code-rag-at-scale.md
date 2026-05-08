---
title: Practical Code RAG at Scale
date: 2026-05-07
tags: [code-rag, bm25, chunking, long-code-arena, retrieval, pl-pl, nl-pl, compute-budget]
sources: [practical-code-rag-at-scale.md]
---

# Practical Code RAG at Scale

JetBrains Research (Galimzyanov, Kolomyttseva, Bogomolov), arXiv 2510.20609v1

## 核心问题

在现实计算预算下，代码 RAG 系统的检索配置（分块策略、相似度评分、切分粒度）如何选择？不同任务类型和上下文窗口大小是否需要不同配置？

## 方法

- **基准**：Long Code Arena (LCA)，两个互补任务
  - Code Completion (CC)：PL→PL 检索，指标 Exact Match (EM)
  - Bug Localization (BL)：NL→PL 检索，指标 NDCG
- **三轴实验**：Chunker × Splitter × Scorer
- **上下文预算**：128–16,384 tokens
- **语言**：Java, Kotlin, Python

## 关键发现

### 1. PL→PL 用 BM25，NL→PL 用 Dense

BM25 + 词级切分在代码补全上比 E5-large 高 ~10 p.p. EM，且快 ~14 倍。Bug 定位则完全不同：**voyage-code-3**（Voyage AI；论文表记 Voyager-3-code）NDCG 0.72 vs BM25 0.57，但延迟高 100x+。

### 2. Chunk 大小应与上下文窗口对齐

论文区分了两个独立的分块参数：
- **索引分块大小 (L_i)**：源文件按 L_i 行切分为非重叠窗口进行索引
- **查询窗口大小 (L_q)**：补全文件中，目标行前的最后 L_q 行作为查询

| 上下文预算 | 最优分块大小 |
|-----------|-------------|
| ≤4K tokens | 32–64 行 |
| 4K–8K tokens | 64–128 行 |
| ≥16K tokens | 整文件检索 |

小上下文需精确聚焦的相关 chunk；大上下文可容纳更完整的代码段。L_i 与 L_q 的最优值通常一致，但可以独立调节。

### 3. 行级分块 ≈ 语法感知分块

简单的行级切分与 AST 感知切分性能一致，代码补全更依赖语义相似片段匹配而非层级结构保留。

### 4. 延迟差异高达 200×

| 配置 | 延迟 (s/1M symbols) |
|------|-------------------|
| IoU + 行级 | 0.02 |
| BM25 + 词级 | 0.2 |
| E5-large | 3.3 |
| BPE 切分 | 1.5–2.0（比词级慢 10x，无质量提升） |

### 5. 混合检索（DraCo + BM25）

数据流图引导检索在中等上下文下不如 chunking 检索，大上下文时两者趋同。目录路径距离检索效果最差。

### 6. Bug 定位中的上下文增强

在 BL 任务的 context 中包含文件路径和 import 语句可显著提升性能 — 这虽不改变检索排序，但为生成模型提供了定位线索。

## 论文 5 条实践建议

1. **匹配任务与检索策略**：PL→PL 用 BM25 + 词级切分，NL→PL 用 dense embedding
2. **对齐分块粒度与模型上下文容量**：≤4K tokens 用 32-64 行，4K-8K 用 64-128 行，≥16K 考虑整文件
3. **优先实现效率**：词级切分 ≈ BPE 质量但快 9×；极低延迟可用 IoU + 行级
4. **复杂度不一定提升质量**：代码补全中结构信息未增强性能；Bug 定位中文件路径和 import 有价值
5. **整体考虑检索管道**：最有效的 RAG 系统会组合多种检索策略，针对任务需求、代码库特征和计算约束定制

## 实验设计亮点

- **分阶段搜索**：先固定 chunker 筛 scorer×splitter，再固定最优配置筛 chunk size，最后比 chunker 类型
- **Query 构造**：CC 用目标行前的最后 chunk，BL 用 issue 文本
- **贪心打包**：按 rank 顺序填入 prompt 直到 token 预算用尽

## 局限

- 仅测试单行补全 + Bug 定位，未覆盖多行补全/长文本生成
- 生成模型仅用 DeepSeek-Coder-1.3B
- 未测试去重和上下文压缩
- 仅限仓库内检索

## 与 Wiki 其他页面的关系

- [[techniques/bm25-retrieval]] — 本文的核心稀疏检索方法
- [[techniques/sparse-vs-dense-retrieval]] — 任务模态决定检索范式
- [[concepts/pl-pl-vs-nl-pl-retrieval]] — PL→PL 与 NL→PL 的本质差异
- [[techniques/code-chunking]] — chunk-size 与 context-window 对齐的实验证据
- [[techniques/semantic-chunking]] — 行级分块 ≈ 语法感知分块的实证
- [[concepts/retrieval-quality]] — 检索质量受任务类型和计算预算共同影响
- [[concepts/token-limits]] — 上下文预算决定最优 chunk 大小
