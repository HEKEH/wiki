---
title: Code Chunking for RAG
date: 2026-05-05
tags: [chunking, code-rag, langchain, text-splitting]
sources: [How-to-Chunk-Code-for-RAG.md, Effective-Chunking-Strategies-for-RAG.md, Repoformer-Selective-Retrieval-for-Repository-Level-Code-Completion.md]
---

# Code Chunking for RAG

将代码切分为逻辑块以适配 RAG 的 token 限制和语义边界。

## 核心方法

使用 `RecursiveCharacterTextSplitter`，配合代码友好的分隔符：

```python
splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,       # 字符数，非 token 数
    chunk_overlap=50,
    separators=["\n\n", "\n", " "]
)
```

## 关键陷阱：chunk_size 是字符数

`chunk_size` 计量单位是**字符**，不是 token。代码约 4 字符/token，因此 500 字符 ≈ 125 token。需用 `tiktoken` 校验实际 token 量：

```python
import tiktoken
enc = tiktoken.encoding_for_model('gpt-4o')
tokens = len(enc.encode(chunk))
```

## 最佳实践

- 优先在空行、换行处切分，保留函数/类定义完整性
- `chunk_overlap` 不宜过大，减少冗余 token
- 切分前可先裁剪注释和非核心代码
- 模型上下文允许时，可将多个 chunk 合并在一次请求中
- 用实际代码库测试，避免函数签名被截断

## 常见错误

按固定字符数随意切分，破坏代码语义完整性 → 检索返回破碎片段，生成质量差。

## 变体方案

| 方案 | 延迟 | 成本/调用 | 适用场景 |
|------|------|-----------|----------|
| 标准分块 (gpt-4o) | ~800ms | ~$0.002/500 token | 高精度 RAG |
| 流式输出 | 更快感知 | 同上 | 大代码库，更好 UX |
| 异步分块 | 更高吞吐 | 同上 | 并发处理 |
| gpt-4o-mini | ~400ms | ~$0.0005/500 token | 成本敏感的摘要 |

## 分块策略框架

设计分块策略时需三层决策（[[sources/src-effective-chunking-strategies-for-rag]]）：

1. **文档切分方式** — [[concepts/content-dependent-splitting|内容依赖]]还是内容无关？
2. **Chunk 大小** — 小 chunk 检索精确但缺上下文，大 chunk 反之
3. **是否重叠** — [[techniques/chunk-overlap|Overlap]] 补偿切分丢失的上下文

## Chunk 大小与上下文窗口的关系

[[sources/src-practical-code-rag-at-scale|Galimzyanov et al. (2025)]] 在 Long Code Arena 上的实验证明，最优 chunk 大小取决于模型的上下文窗口容量。论文区分了两个独立参数：

- **索引分块大小 (L_i)**：源文件按 L_i 行切分为非重叠窗口进行索引，L_i=∞ 表示整文件索引
- **查询窗口大小 (L_q)**：补全文件中，目标行前的最后 L_q 行构成查询

两者可独立调节，最优值通常一致：

| 上下文预算 | 最优分块大小 | 原因 |
|-----------|-------------|------|
| ≤4K tokens | 32–64 行 | 小上下文需精确聚焦的相关 chunk，提高相关性密度 |
| 4K–8K tokens | 64–128 行 | 中等上下文可容纳更多结构信息 |
| ≥16K tokens | 整文件检索 | 大上下文可理解完整代码关系，整文件避免信息碎片化 |

小 chunk (8-16 行) 在所有上下文长度下均表现不佳，因包含的信息不足以建立有意义的相关性。

## 切分类型：行级 vs 语法感知

同一实验发现，简单的行级切分与语法感知（AST）切分性能一致。代码补全更依赖语义相似片段的匹配，而非层级结构的保留。这降低了实现复杂度 — 不需要解析器即可获得同等效果。

## Repoformer 的索引分块配置

[[sources/src-repoformer-selective-retrieval|Wu et al. (2024)]] 在仓库级代码补全中使用了滑窗分块 + overlap：

| 任务类型 | Chunk size (行) | Stride (行) | Overlap 比例 |
|---------|----------------|------------|-------------|
| Line / API / Chunk 补全 | 20 | 10 | 50% |
| Function 补全 | 50 | 25 | 50% |

检索器使用 Jaccard 相似度，查询由 $X_l$ 末尾与 chunk 等长的行构成。该配置与 chunk-size ≈ query-size 的对齐原则一致（参见上方 Chunk 大小与上下文窗口的关系）。

## 与其他概念的关系

- [[techniques/recursive-character-text-splitter]] — 内容无关的递归分隔工具
- [[techniques/character-text-splitter]] — 按指定分隔符切分，适合内容依赖场景
- [[techniques/semantic-chunking]] — 更高级的语义感知分块策略
- [[techniques/chunk-overlap]] — 重叠策略详解
- [[concepts/token-limits]] — 理解 chunk_size 与 token 的换算关系
- [[concepts/content-dependent-splitting]] — 内容依赖 vs 内容无关切分
- [[concepts/pl-pl-vs-nl-pl-retrieval]] — 任务模态影响分块策略选择
