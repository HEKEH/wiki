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
- 根据 LLM 的上下文窗口大小动态调整 chunk 大小，而非使用固定配置

## 上下文预算决定最优 Chunk 大小

[[sources/src-practical-code-rag-at-scale|Galimzyanov et al. (2025)]] 证明 token 预算不仅约束输入量，还决定检索粒度的最优选择：

| 上下文预算 | 最优分块 | 原因 |
|-----------|---------|------|
| ≤4K tokens | 32–64 行 | 精确 chunk 提高相关性密度 |
| 4K–8K tokens | 64–128 行 | 更多空间容纳结构信息 |
| ≥16K tokens | 整文件 | 大上下文可理解完整代码关系 |

小上下文模型不应贪心塞入大 chunk — 相关性密度下降反而拖累生成质量。大上下文模型则可利用更完整的代码段建立全局理解。

## 相关

- [[techniques/code-chunking]] — chunk_size 的字符/token 换算是分块的前提
