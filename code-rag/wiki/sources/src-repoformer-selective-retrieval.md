---
title: Repoformer: Selective Retrieval for Repository-Level Code Completion
date: 2026-05-12
tags: [selective-rag, code-completion, self-assessment, retrieval-policy, inference-efficiency]
sources: [Repoformer-Selective-Retrieval-for-Repository-Level-Code-Completion.md]
---

# Repoformer: Selective Retrieval for Repository-Level Code Completion

Wu, Ahmad, Zhang, Ramanathan, Ma — arXiv 2403.10059v2 (2024)

## 核心问题

RAG 系统总是执行检索，但检索并不总是有帮助——甚至经常有害。能否让模型自行判断是否需要检索？

## 关键发现：检索不总是有帮助

论文系统性地验证了 **always-RAG 的悖论**：

- 仅有 **~20%** 的检索实例能提升模型性能
- 超过 **60%** 的检索对结果无影响（输出未改变）
- 约 **20%** 的检索实际**损害**性能（引入噪声）
- 该规律在 CodeGen-Mono (2B/16B) 和 StarCoder (1B/16B) 上均成立
- 覆盖 API 补全和函数补全任务

这是对 [[concepts/retrieval-quality|检索质量]] 的一个重要补充：检索不仅存在质量差异，还存在**是否应该检索**的根本决策问题。

## 方法：Selective RAG 框架

### 自选择性检索 (Self-selective RAG)

模型观察当前文件上下文 $(X_l, X_r)$ 后，通过生成特殊 token `<cc>` 来触发检索，或生成空 token $\phi$ 放弃检索。整个过程在单次 left-to-right pass 中完成：

1. 模型看到 $X_l$ + `<eof>` → 自评估是否需要检索
2. 如需要 → 触发检索器获取跨文件上下文 $CC$ → 结合 $CC$ 生成
3. 如不需要 → 直接生成

### 两种决策策略

| 策略 | 规则 | 特点 |
|------|------|------|
| **Greedy Selection** | `<cc>` 概率最高时触发检索 | 延迟节省最大（~60-70%），精度微降 ~1 ES |
| **Threshold Selection** | `<cc>` 概率 > 阈值 T 时触发 | 同时提升精度和延迟（推荐） |

阈值参考：函数补全 T=0.15，其他任务 T=0.2。

### 自监督多任务训练

**数据构造**（从 The Stack 的 18K Python 仓库）：
1. 采样目标行 $Y$（随机代码块或函数体）
2. 用当前文件检索跨文件上下文 $CC$（50% 数据包含 $Y$ 在查询中）
3. 标注：对比模型有无 $CC$ 时的 Edit Similarity，若提升 > 阈值 $T$ 则 label=true

**训练目标**：$\lambda \mathcal{L}_{eval} + \mathcal{L}_{gen}$
- $\mathcal{L}_{eval}$：`<eof>` 后预测 `<cc>` 的交叉熵（学习何时检索）
- $\mathcal{L}_{gen}$：预测 $Y$ 的交叉熵（学习利用检索结果生成代码）

模型：基于 StarCoderBase 微调 1B/3B/7B/16B 变体。

## 实验结果

### 性能

| 模型 | 策略 | 相比 StarCoderBase Always-RAG |
|------|------|------------------------------|
| Repoformer-3B | Selective_T | 超越 StarCoderBase-7B 大部分任务 |
| Repoformer-3B | Selective_T | API/Chunk ES 超越 5× 大的 StarCoder-16B |
| Repoformer-16B | Selective_T | 所有任务 SOTA，平均 +3 ES |

### 推理效率

| 场景 | 加速比 | 精度影响 |
|------|--------|---------|
| Greedy（仅 ~20% 实例检索） | 60-70% | -1 ES |
| Threshold（~60-80% 实例检索） | 16-28% | **正提升** |
| Dense retrieval + 大仓库 | 最高 70% | 无损 |

### 作为即插即用策略

Repoformer-1B 可作为其他大模型（StarCoderBase-7B/16B, CodeGen25-7B, Code Llama-7B/16B, ChatGPT）的选择性检索策略：
- 平均 ~25% 延迟减少
- **同时**提升精度

## 分析与消融

### 检索弃权的精确度

Repoformer 弃权检索的决策在 >80% 实例上是精确的——要么模型已无检索答对，要么检索确实无法改善结果。

### 对检索结果的鲁棒性

对比 StarCoderBase，Repoformer 在请求检索时获得了更多更大的性能提升，同时性能下降的实例显著减少——训练使其学会了利用而非被噪声干扰。

### 消融实验

| 变体 | 说明 | 结果 |
|------|------|------|
| A1: 合并损失 | $\mathcal{L}_{eval}$ 被长序列淹没 | `<cc>` 概率几乎总为 1，无法选择 |
| A2: 去掉 $\mathcal{L}_{eval}$ | 仅训练生成 | RAG 能力略好，但无选择性能力 |
| A3: 去掉 CC 训练 | 仅 FIM 训练 | RAG 性能差，证明 CC 训练必要 |
| A4: CC 放在生成部分 | 破坏 FIM 语义 | 函数补全 ES 从 ~53 退化到 ~25 |

### 传统启发式的局限

- **Trial Retrieval**（基于相似度分数过滤）：可节省 40-50% 检索，但无延迟节省（仍需执行检索才能决策）
- **Trial Generation**（基于不确定性）：对长序列生成（函数补全）完全失效，且需额外推理开销

## 与已有 RAG 方法的对比

| 方法 | 决策机制 | 额外模块 | 延迟开销 |
|------|---------|---------|---------|
| SKR | 收集无帮助实例 + 分类器 | 需要知识库 + 分类器 | 中 |
| Self-RAG | GPT-4 蒸馏 | 需要 oracle LM 标注 | 高（蒸馏成本） |
| **Repoformer** | 自评估 token | 无 | 极低（单次 forward pass） |

## RAG 执行细节

- **索引**：滑窗分块，chunk size 20（line/API/chunk）/ 50（function），stride = chunk_size/2
- **查询**：$X_l$ 末尾的固定行数
- **检索**：Jaccard 相似度，top-10 + Fragment Alignment（返回匹配 chunk 的下一 chunk）
- **生成**：left context 截断 1024 tokens，right context 512 tokens，CC 512 tokens

## 限制

- 仅覆盖 Python（多语言版在附录）
- 训练标签基于词法相似度 (ES)，非执行验证——函数补全校准性较差
- 未考虑个性化检索策略（不同仓库的 RAG 友好度不同）
- CrossCodeLongEval 是论文新提出的基准

## 与 Wiki 其他页面的关系

- [[concepts/selective-rag]] — 本文核心概念
- [[systems/repoformer]] — 系统页面
- [[concepts/retrieval-quality]] — 检索不总是有益的重要证据
- [[techniques/sparse-vs-dense-retrieval]] — Repoformer 使用 Jaccard 稀疏检索，但也兼容 dense
- [[techniques/bm25-retrieval]] — Jaccard 相似度与 BM25 同属稀疏检索
- [[techniques/code-chunking]] — 论文的索引分块策略（滑窗 + overlap）
- [[concepts/pl-pl-vs-nl-pl-retrieval]] — Repoformer 面向 PL→PL 场景
