---
title: Code Chunking for RAG
date: 2026-05-05
tags: [chunking, code-rag, langchain, text-splitting]
sources: [How-to-Chunk-Code-for-RAG.md]
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

## 与其他概念的关系

- [[techniques/recursive-character-text-splitter]] — 本篇使用的具体工具
- [[techniques/semantic-chunking]] — 更高级的语义感知分块策略
- [[concepts/token-limits]] — 理解 chunk_size 与 token 的换算关系
