# Home

Top-level synthesis and orientation for this knowledge base.

## About

This wiki is incrementally built and maintained by an LLM agent. Raw sources are immutable; the wiki is the LLM's layer — summaries, concept pages, technique pages, cross-references, and evolving synthesis.

## Quick Links

- [[index]] — catalog of all wiki pages
- [[log]] — chronological activity record

## Current State

This wiki is focused on **Code RAG** — how LLMs retrieve and use codebase context, code search, embedding strategies, and tooling for code-aware AI.

### 核心洞见

1. **分块是 Code RAG 的基础** — [[techniques/code-chunking]] 的质量直接决定检索和生成效果，语义完整性比切分效率更重要
2. **字符 ≠ Token** — [[concepts/token-limits]] 是最常见的陷阱：`RecursiveCharacterTextSplitter` 的 `chunk_size` 是字符数，代码约 4 字符/token
3. **语义分块优于固定长度切分** — [[techniques/semantic-chunking]] 在逻辑边界处切分，避免破碎 chunk 导致检索不可用
4. **检索质量是系统瓶颈** — [[concepts/retrieval-quality]] 受分块策略、chunk 大小、embedding 质量等多因素影响

## Open Questions

- 代码分块是否存在比 `RecursiveCharacterTextSplitter` 更优的通用方案（如 AST 感知分块）？
- 代码 embedding 模型（如 voyage-code）与通用文本 embedding 在检索质量上的差异有多大？
- Chunk overlap 的最优比例是否存在领域相关的经验值？
