# Home

Top-level synthesis and orientation for this knowledge base.

## About

This wiki is incrementally built and maintained by an LLM agent. Raw sources are immutable; the wiki is the LLM's layer — summaries, entity pages, concept pages, cross-references, and evolving synthesis.

## Quick Links

- [Index](index.md) — catalog of all wiki pages
- [Log](log.md) — chronological activity record

## Current State

This wiki is focused on **Agent 架构设计**，特别是如何构建可扩展、可演进的 Agent 系统。当前核心主题围绕 [Anthropic](entities/Anthropic.md) 的 [Managed Agents](entities/Managed-Agents.md) 解耦架构与 [Agentic Systems 分类框架](concepts/agentic-systems.md) 展开。

### 核心洞见

1. **简单优先，复杂度须验证** — [Agentic Systems 分类框架](concepts/agentic-systems.md) 的首要原则：能用单次 LLM 调用解决就不要用 Agent，只在简单方案不足时增加复杂度
2. **Workflows 与 Agents 的分野** — [Workflows](concepts/agentic-systems.md) 沿预定义路径编排，[Agents](concepts/agentic-systems.md) 动态自控；前者可预测，后者灵活，选择取决于任务性质
3. **[The Augmented LLM](concepts/augmented-llm.md) 是一切的基础** — 检索、工具、记忆三大增强构成所有 Agent 系统的起点，五种 Workflow 模式是对 Augmented LLM 的逐步编排
4. **解耦是 Agent 系统扩展的基础** — [Brain-Hands-Session 解耦模型](concepts/brain-hands-session.md) 将 Agent 的推理、执行和记忆分离为可独立替换的组件
5. **接口应比实现更长寿** — [Meta-harness](concepts/meta-harness.md) 借鉴操作系统的抽象思路，为"尚未设想的程序"设计系统
6. **假设会随模型进化而过时** — [The Bitter Lesson](concepts/bitter-lesson.md) 和 [Context Anxiety](concepts/context-anxiety.md) 提醒我们：弥补模型缺陷的特定设计终将失效
7. **[ACI](concepts/aci.md) 与 HCI 同等重要** — 工具定义应获得与 prompt 工程同等的投入，Poka-yoke 防呆设计让错误不可能发生
8. **安全边界不可妥协** — 凭证永远不暴露在 [Sandbox](concepts/sandbox.md) 中
9. **长时运行需要结构化交接** — [Long-Running Agent](concepts/long-running-agent.md) 无法依赖 context window 传递状态，[Initializer/Coding Agent 模式](concepts/initializer-coding-agent.md) 通过 progress file + feature list + git history 三重机制桥接 session
10. **增量推进胜过一次性完成** — [Feature List Pattern](concepts/feature-list-pattern.md) 将需求拆解为细粒度可验证项，一次做一个 feature，端到端测试后才标记完成

## Open Questions

- Workflow 模式中硬编码的编排逻辑，随着模型能力增长，是否会逐步消融为纯 Agent？
- 五种 Workflow 模式之外，是否存在尚未被识别的常见编排模式？
- Meta-harness 的接口设计能否真正容纳未来未知的 harness 类型？
- 随着模型能力持续增长，harness 中的哪些逻辑可以完全消除？
- Context engineering 的最佳实践是否会收敛，还是会随模型进化持续变化？
- 单一通用 coding agent vs 多 agent 架构（测试/QA/清理 agent）哪个更优？
- Feature List Pattern 能否泛化到 Web 开发以外的领域（科研、金融建模）？
- ACI 设计原则（如 Poka-yoke）能否系统化为一套可复用的工具设计 checklist？
