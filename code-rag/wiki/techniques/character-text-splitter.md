---
title: CharacterTextSplitter
date: 2026-05-05
tags: [chunking, langchain, text-splitting]
sources: [Effective-Chunking-Strategies-for-RAG.md]
---

# CharacterTextSplitter

LangChain 提供的文本分块工具，按指定分隔符将文本切分为不超过 `chunk_size` 的片段。

## 与 RecursiveCharacterTextSplitter 的区别

| 维度 | CharacterTextSplitter | RecursiveCharacterTextSplitter |
|------|----------------------|-------------------------------|
| 分隔符 | 单一指定分隔符 | 分隔符列表，递归尝试 |
| 灵活性 | 需要预处理插入分隔符 | 自动按层级回退 |
| 适用场景 | 内容依赖切分（已知文档结构） | 内容无关切分（通用文本） |

## 使用场景

当文档经过预处理、已插入自定义分隔符时，`CharacterTextSplitter` 可精确控制切分位置。Cohere 转录稿实验中，先插入 `###` 标记发言切换点，再按此分隔符切分。

## 配置

```python
CharacterTextSplitter(
    separator="###",       # 按此分隔符切分
    chunk_size=1000,       # 最大字符数
    chunk_overlap=0,       # 无需 overlap（语义边界已保证）
    is_separator_regex=False
)
```

## 与内容依赖切分的关系

`CharacterTextSplitter` 是 [[concepts/content-dependent-splitting|内容依赖切分]] 的常用实现工具 — 预处理阶段识别文档结构并插入分隔符，切分阶段按分隔符断开。

## 相关

- [[techniques/recursive-character-text-splitter]] — 递归分隔方案
- [[techniques/code-chunking]] — 代码分块的整体策略
- [[concepts/content-dependent-splitting]] — 内容依赖切分策略
