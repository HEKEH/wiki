---
title: Effective Chunking Strategies for RAG
date: 2026-05-05
tags: [chunking, overlap, content-dependent, cohere, langchain, retrieval-quality]
sources: [Effective-Chunking-Strategies-for-RAG.md]
---

# Effective Chunking Strategies for RAG

Cohere 官方文档，以电话会议转录稿为例，对比不同分块策略对 RAG 生成质量的影响。

## 核心框架

分块策略设计分三层决策：

1. **文档切分方式** — 按什么条件断开文本
2. **Chunk 大小** — 从切分结果组装多长的 chunk
3. **是否重叠** — 相邻 chunk 之间是否保留重叠区

## 内容无关 vs 内容依赖切分

### 内容无关切分

不假设文本结构，按通用条件切分：
- 按字符数（`RecursiveCharacterTextSplitter`）
- 按句子
- 按固定字符（如 `\n` 分段落）

优势：通用，无需了解文档格式。劣势：可能切断语义单元。

### 内容依赖切分

根据文档结构定制切分规则。例：会议转录稿中，按发言者切换点切分，确保每个人的发言完整保留在一个 chunk 内。

## 关键实验

**问题**：Who mentions Jonathan Nolan？

| 实验 | 策略 | chunk_size | overlap | 结果 |
|------|------|-----------|---------|------|
| 1 | 内容无关（RecursiveCharacterTextSplitter） | 500 | 0 | 错误 — 发言人名字被切到另一个 chunk |
| 2 | 内容无关（同上） | 500 | 100 | 正确 — overlap 捕获了被切断的发言人 |
| 3 | 内容依赖（CharacterTextSplitter + `###` 分隔符） | 1000 | 0 | 正确 — 发言人信息自然完整 |

## 关键洞见

- Overlap 是内容无关切分的有效补偿手段，但引入冗余
- 内容依赖切分从根源上避免语义断裂，但需要文档预处理
- 小 chunk 适合精确检索场景，大 chunk 保留更多上下文
- 转录稿等结构化文档强烈建议内容依赖切分

## 工具链

- `RecursiveCharacterTextSplitter`（LangChain）— 递归分隔，内容无关
- `CharacterTextSplitter`（LangChain）— 按指定分隔符切分，适合内容依赖场景
- Cohere：`embed-v4.0`（embedding）、`co.rerank`（重排序）、`co.chat`（生成+引用）

## 相关

- [[techniques/code-chunking]] — 代码场景的分块实践
- [[techniques/recursive-character-text-splitter]] — 递归分隔工具详解
- [[techniques/chunk-overlap]] — 重叠策略详解
- [[concepts/content-dependent-splitting]] — 内容依赖切分 vs 内容无关切分
- [[concepts/retrieval-quality]] — 分块质量影响检索效果
