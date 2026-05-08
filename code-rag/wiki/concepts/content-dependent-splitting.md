---
title: Content-Dependent Splitting
date: 2026-05-05
tags: [chunking, content-dependent, semantic-boundary]
sources: [Effective-Chunking-Strategies-for-RAG.md]
---

# Content-Dependent Splitting

根据文档自身结构定制切分规则的策略，与内容无关切分相对。

## 核心思想

内容无关切分按通用条件（字符数、句子、段落）切分，不关心文本含义。内容依赖切分则利用文档的结构特征，确保语义单元不被打断。

## 典型场景

| 文档类型 | 结构特征 | 切分策略 |
|----------|----------|----------|
| 会议转录稿 | 发言人切换 | 按发言者分段 |
| 代码文件 | 函数/类定义 | 按 AST 节点切分 |
| 法律文书 | 条款编号 | 按条款切分 |
| HTML 文档 | 标签层级 | 按语义标签切分 |

## 实操案例

Cohere 转录稿实验（[[sources/src-effective-chunking-strategies-for-rag]]）：

1. 预处理：识别 HTML 中 `<p><strong>Name</strong></p>` 标记的发言人
2. 插入自定义分隔符 `###` 标记发言切换点
3. 用 [[techniques/character-text-splitter|CharacterTextSplitter]] 按 `###` 切分，确保每人发言完整

```python
CharacterTextSplitter(
    separator="###",
    chunk_size=1000,
    chunk_overlap=0
)
```

## 与内容无关切分的对比

| 维度 | 内容无关 | 内容依赖 |
|------|---------|---------|
| 通用性 | 高，无需了解文档 | 低，需针对文档定制 |
| 语义完整性 | 不保证，依赖 overlap 补偿 | 从根源保证 |
| 预处理成本 | 无 | 需要文档分析 |
| 适用场景 | 通用文本 | 结构化/半结构化文档 |

## 对代码 RAG 的启示

代码是天然的结构化文档 — 函数、类、模块都有明确的语法边界。AST 感知分块本质上就是代码场景的内容依赖切分。

### 反证：行级切分 ≈ 语法感知切分

[[sources/src-practical-code-rag-at-scale|Galimzyanov et al. (2025)]] 在 Long Code Arena 代码补全任务上发现，简单的行级切分与 LangChain 语法感知递归切分性能一致甚至略优。这意味着在代码补全场景下，内容依赖切分（AST 感知）的额外预处理成本并未带来质量回报。

**原因分析**：代码补全更依赖语义相似片段匹配，而非层级结构中的父子关系保留。行级切分已保留了足够的代码连贯性。

**但需注意**：该实验仅覆盖单行代码补全。对于更复杂的任务（多行补全、代码修复、跨文件重构），语法结构保留的价值可能更高 — 这仍待验证。内容依赖切分在非代码场景（如转录稿）中的优势已被 [[sources/src-effective-chunking-strategies-for-rag|Cohere 实验]] 证实。

## 相关

- [[techniques/semantic-chunking]] — 语义感知分块的通用策略
- [[techniques/chunk-overlap]] — 内容无关切分的补偿手段
- [[techniques/code-chunking]] — 代码场景的实践方法
