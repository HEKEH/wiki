---
title: RecursiveCharacterTextSplitter
date: 2026-05-05
tags: [chunking, langchain, text-splitting]
sources: [How-to-Chunk-Code-for-RAG.md]
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

## 相关

- [[techniques/code-chunking]] — 代码分块的整体策略
