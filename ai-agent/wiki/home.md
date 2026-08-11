# Home — AI Agent Knowledge Base

Agent 架构设计知识库，由 LLM agent 增量维护。`raw/` 为不可变原始文档，`wiki/` 为 LLM 组织的摘要、概念页、实体页与综合。

## Quick Links

- [[index]] — 全部页面目录
- [[log]] — 活动日志

## Current State

聚焦 **Agent 架构设计** —— 如何构建可扩展、可演进的 Agent 系统。当前主干由 [[entities/01-anthropic]] 的三篇工程博客构成：[[entities/02-managed-agents]] 的解耦架构、长时运行 harness 设计、以及 [[concepts/13-agentic-systems]] 分类框架。

学习动线：

1. 分类框架入门：[[concepts/13-agentic-systems]] → [[concepts/14-augmented-llm]]
2. 五种 Workflow 模式：[[concepts/15-prompt-chaining]]、[[concepts/16-routing]]、[[concepts/17-parallelization]]、[[concepts/18-orchestrator-workers]]、[[concepts/19-evaluator-optimizer]]
3. 运行时解耦：[[concepts/04-brain-hands-session]] → [[concepts/02-harness]]、[[concepts/03-sandbox]]、[[concepts/01-session]]
4. 长时运行：[[concepts/10-long-running-agent]] → [[concepts/11-initializer-coding-agent]] → [[concepts/12-feature-list-pattern]]
5. 设计哲学与工具接口：[[concepts/08-bitter-lesson]]、[[concepts/07-meta-harness]]、[[concepts/20-aci]]

## Core Insights

1. **简单优先，复杂度须验证** — [[concepts/13-agentic-systems]] 的首要原则：能用单次 LLM 调用解决就不要用 Agent，只在简单方案不足时增加复杂度
2. **Workflows 与 Agents 的分野** — [[concepts/13-agentic-systems|Workflows]] 沿预定义路径编排，[[concepts/13-agentic-systems|Agents]] 动态自控；前者可预测，后者灵活，选择取决于任务性质
3. **[[concepts/14-augmented-llm]] 是一切的基础** — 检索、工具、记忆三大增强构成所有 Agent 系统的起点，五种 Workflow 模式是对 Augmented LLM 的逐步编排
4. **解耦是 Agent 系统扩展的基础** — [[concepts/04-brain-hands-session]] 将 Agent 的推理、执行和记忆分离为可独立替换的组件
5. **接口应比实现更长寿** — [[concepts/07-meta-harness]] 借鉴操作系统的抽象思路，为"尚未设想的程序"设计系统
6. **假设会随模型进化而过时** — [[concepts/08-bitter-lesson]] 和 [[concepts/06-context-anxiety]] 提醒我们：弥补模型缺陷的特定设计终将失效
7. **[[concepts/20-aci]] 与 HCI 同等重要** — 工具定义应获得与 prompt 工程同等的投入，Poka-yoke 防呆设计让错误不可能发生
8. **安全边界不可妥协** — 凭证永远不暴露在 [[concepts/03-sandbox]] 中
9. **长时运行需要结构化交接** — [[concepts/10-long-running-agent]] 无法依赖 context window 传递状态，[[concepts/11-initializer-coding-agent]] 通过 progress file + feature list + git history 三重机制桥接 session
10. **增量推进胜过一次性完成** — [[concepts/12-feature-list-pattern]] 将需求拆解为细粒度可验证项，一次做一个 feature，端到端测试后才标记完成

## Open Questions

- Workflow 模式中硬编码的编排逻辑，随着模型能力增长，是否会逐步消融为纯 Agent？
- 五种 Workflow 模式之外，是否存在尚未被识别的常见编排模式？
- Meta-harness 的接口设计能否真正容纳未来未知的 harness 类型？
- 随着模型能力持续增长，harness 中的哪些逻辑可以完全消除？
- Context engineering 的最佳实践是否会收敛，还是会随模型进化持续变化？
- 单一通用 coding agent vs 多 agent 架构（测试/QA/清理 agent）哪个更优？
- Feature List Pattern 能否泛化到 Web 开发以外的领域（科研、金融建模）？
- ACI 设计原则（如 Poka-yoke）能否系统化为一套可复用的工具设计 checklist？
- `analysis/` 目录尚空：缺少跨源对比（如五种 Workflow 模式与 Brain-Hands-Session 解耦的适用边界）。
