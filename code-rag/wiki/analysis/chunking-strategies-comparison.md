---
title: 分块策略对比：The Neural Base vs Cohere
date: 2026-05-05
tags: [chunking, comparison, overlap, content-dependent]
sources: [How-to-Chunk-Code-for-RAG.md, Effective-Chunking-Strategies-for-RAG.md]
---

# 分块策略对比：The Neural Base vs Cohere

对比两篇来源文章的核心观点，分析共识与分歧。

## 共识

### 1. 语义边界优先于固定长度

两篇都明确反对按固定字符数随意切分。The Neural Base 称其为 "common mistake"，Cohere 用实验证明固定切分会导致发言人名字被切断、检索失败。两者都推荐在逻辑边界处切分。

### 2. RecursiveCharacterTextSplitter 是首选工具

两者都以 `RecursiveCharacterTextSplitter` 作为默认方案，且都使用 `["\n\n", "\n", " "]` 分隔符层级。这是当前最通用的分块工具共识。

### 3. chunk_size 是字符数

两篇都清楚 `chunk_size` 计量单位是字符。The Neural Base 更进一步，给出换算公式（代码约 4 字符/token）并推荐用 tiktoken 校验。

### 4. Overlap 有价值

两篇都认可 chunk overlap 作为切分边界的信息补偿手段。

## 分歧

### 1. Overlap 的权重

| 维度 | The Neural Base | Cohere |
|------|----------------|--------|
| 态度 | "use sparingly to reduce redundant tokens" | 视为内容无关切分的关键补偿手段 |
| 实验 | 无 | 有：无 overlap → 检索失败，加 overlap → 成功 |

The Neural Base 从成本优化角度建议少用 overlap；Cohere 从检索完整性角度证明 overlap 是兜底手段。**差异原因**：代码文本的语义边界比转录稿更清晰（函数/类有明确语法标记），代码场景对 overlap 的依赖度本身就更低。

### 2. 是否需要内容依赖切分

| 维度 | The Neural Base | Cohere |
|------|----------------|--------|
| 提及 | 未讨论 | 核心论点：三层框架 + 内容依赖 vs 内容无关 |

The Neural Base 只讨论内容无关切分（RecursiveCharacterTextSplitter），不涉及按文档结构定制切分规则。Cohere 则认为内容依赖切分是更优解 — 它从根源避免语义断裂，无需依赖 overlap 兜底。

**但**：The Neural Base 关注的是代码，代码的 `["\n\n", "\n", " "]` 分隔符层级实际上已经隐含了代码的结构（空行 = 函数/类边界），所以 `RecursiveCharacterTextSplitter` 在代码场景下本身就有一定的"内容感知"能力。而 Cohere 面对的转录稿没有这样明确的通用分隔符，因此必须显式预处理。

### 3. 讨论范围

| 维度 | The Neural Base | Cohere |
|------|----------------|--------|
| 范围 | 分块步骤本身 | 完整 RAG pipeline（embed → rerank → generate） |
| 验证方式 | tiktoken 校验 token 数 | 引用归因验证检索结果 |
| 实验深度 | 代码示例 + 性能数据 | 对照实验（3 组策略对比） |

The Neural Base 是操作手册风格（怎么做）；Cohere 是实验分析风格（为什么这样做）。

### 4. Chunk 大小指导

| 维度 | The Neural Base | Cohere |
|------|----------------|--------|
| 建议 | 具体数值：500 字符 / chunk_overlap=50 | 原则性：小 chunk 精确但缺上下文，大 chunk 反之 |

The Neural Base 给出可直接使用的配置值；Cohere 只给出权衡框架，需要根据场景实验。

## 综合：代码分块的最优实践

两篇结合，对代码 RAG 的启示：

1. **默认用 RecursiveCharacterTextSplitter** — 代码的结构性使内容无关切分已有不错效果
2. **但内容依赖切分是更优方向** — AST 感知分块就是代码场景的内容依赖切分，可从根源保证函数/类完整性
3. **Overlap 权重因场景而异** — 代码场景可少用（语法边界已提供足够保护），但部署前务必用实际代码测试边界情况
4. **用 tiktoken 校验 + 引用归因验证** — 前者确保 chunk 不超 token 限制，后者确保检索结果能支撑答案

## 第三视角：Galimzyanov et al. (2025) 的修正

[[sources/src-practical-code-rag-at-scale|Galimzyanov et al. (2025)]] 提供了系统性实验证据，对上述两篇的观点形成重要补充和修正：

### 对共识 2 的补充：行级切分已足够

两篇都推荐 `RecursiveCharacterTextSplitter` 作为首选，隐含假设是递归分隔比简单行级切分更好。但 Galimzyanov 等人的实验发现 **行级切分 ≈ 语法感知递归切分**，代码补全场景下简单行级切分性能一致甚至略优。这意味着递归分隔的额外复杂度在代码补全中未必有回报。

### 对分歧 2 的深化：内容依赖切分的 ROI 质疑

Cohere 认为内容依赖切分是更优方向，The Neural Base 未涉及。Galimzyanov 等人的实验则对代码补全场景提出了反证 — AST 感知切分无稳定收益，与 [[concepts/content-dependent-splitting|内容依赖切分]] 的理论优势形成张力。但需注意该实验仅覆盖单行补全，更复杂任务仍待验证。

### 新增维度：Chunk 大小与上下文窗口的交互

前两篇均未讨论 chunk 大小与模型上下文容量的关系。Galimzyanov 等人发现这是关键因素 — ≤4K tokens 用 32-64 行，≥16K tokens 可用整文件。这补充了 Cohere "小 chunk 精确但缺上下文，大 chunk 反之" 的原则性描述，给出了可操作的量化指南。
