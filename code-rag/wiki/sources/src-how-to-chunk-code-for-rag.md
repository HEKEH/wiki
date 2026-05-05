---
title: How to Chunk Code for RAG
date: 2026-05-05
tags: [source, chunking, code-rag]
sources: [How-to-Chunk-Code-for-RAG.md]
---

# How to Chunk Code for RAG

> **来源**: [The Neural Base](https://theneuralbase.com/chunking/qna/how-to-chunk-code-for-rag/)
> **作者**: The Neural Base
> **验证日期**: 2026-04

## 摘要

使用 `RecursiveCharacterTextSplitter` 对代码做分块，适配 RAG 的 token 限制和语义边界。核心要点：

1. `chunk_size` 是字符数而非 token 数，500 字符 ≈ 125 token（代码约 4 字符/token），需用 tiktoken 校验
2. 用 `separators=["\n\n", "\n", " "]` 保留逻辑块边界
3. 避免按固定字符数随意切分，会破坏代码语义完整性
4. 变体方案：流式输出、异步处理、gpt-4o-mini 降成本

## Team Note 关键纠正

- 文章说 "chunk_size = 500 max tokens approx"，实际是字符数
- 流式代码示例未检查 `event.choices[0].delta.content` 为 null 的情况
- 异步示例缺少速率限制和超时处理

## 关联页面

- [[techniques/code-chunking]] — 代码分块整体策略
- [[techniques/recursive-character-text-splitter]] — 具体工具详解
- [[techniques/semantic-chunking]] — 语义感知分块
- [[concepts/token-limits]] — 字符/token 换算
