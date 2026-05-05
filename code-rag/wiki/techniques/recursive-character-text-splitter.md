---
title: RecursiveCharacterTextSplitter
date: 2026-05-05
tags: [chunking, langchain, text-splitting]
sources: [How-to-Chunk-Code-for-RAG.md, Effective-Chunking-Strategies-for-RAG.md]
---

# RecursiveCharacterTextSplitter

LangChain 提供的文本分块工具，按分隔符层级递归切分，尽量在语义边界处断开。

## 工作原理

给定一组分隔符（如 `["\n\n", "\n", " "]`），依次尝试用每个分隔符将文本切成不超过 `chunk_size` 的片段。优先在段落边界切，其次换行，最后空格。这比固定长度切分更能保留语义完整性。

## 代码分块配置

```python
RecursiveCharacterTextSplitter(
    chunk_size=500,        # 字符数（非 token）
    chunk_overlap=50,      # 相邻 chunk 重叠字符数
    separators=["\n\n", "\n", " "]
)
```

## 注意事项

- `chunk_size` 是字符计量，代码约 4 字符/token，需用 [[techniques/code-chunking|tiktoken 校验]]
- `chunk_overlap` 过大会引入冗余，过小会丢失上下文
- 过小的 `chunk_size` 可能将函数/类定义从中间截断

## 局限性

作为[[concepts/content-dependent-splitting|内容无关切分]]工具，它无法识别文档的语义结构。Cohere 实验（[[sources/src-effective-chunking-strategies-for-rag]]）表明，无 overlap 时可能将关键信息（如发言人名字）切到相邻 chunk，导致检索失败。需配合 [[techniques/chunk-overlap|overlap]] 或改用 [[techniques/character-text-splitter|CharacterTextSplitter]] 进行内容依赖切分。

## 相关

- [[techniques/code-chunking]] — 代码分块的整体策略
- [[techniques/character-text-splitter]] — 按指定分隔符切分的替代方案
- [[techniques/chunk-overlap]] — overlap 补偿策略
- [[concepts/content-dependent-splitting]] — 内容依赖切分
