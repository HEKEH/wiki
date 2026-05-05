---
title: Token Limits
date: 2026-05-05
tags: [tokens, context-window, chunking]
sources: [How-to-Chunk-Code-for-RAG.md]
---

# Token Limits

LLM 的上下文窗口以 token 为单位，而代码分块工具（如 RecursiveCharacterTextSplitter）以字符为单位，两者需要换算。

## 字符与 Token 的换算

代码约 **4 字符/token**（英文代码），因此：
- 500 字符 ≈ 125 token
- 2000 字符 ≈ 500 token

中文和特殊符号的 token 密度不同，需用 `tiktoken` 实测。

## 校验方法

```python
import tiktoken
enc = tiktoken.encoding_for_model('gpt-4o')
token_count = len(enc.encode(text))
```

## 实践建议

- 配置 `chunk_size` 时按字符设置，但始终用 tiktoken 验证实际 token 数
- 预留 token 给 prompt 和输出，不要将整个上下文窗口用于 chunk

## 相关

- [[techniques/code-chunking]] — chunk_size 的字符/token 换算是分块的前提
