# Home — AI Agent Knowledge Base

AI Agent 知识库，由 LLM agent 增量维护。`raw/` 为不可变原始文档（13 份一手材料 + 论文摘要汇编 + 配图），`wiki/` 为 LLM 组织的摘要、概念页、实体页与综合分析。

## Quick Links

- [[index]] — 全部 81 个内容页目录
- [[log]] — 活动日志
- **[[analysis/01-interview-roadmap]] — 面试导向学习路线（新手从这里进）**
- [[analysis/02-interview-qa-bank]] — 高频面试题与答题要点
- [[analysis/05-glossary]] — 术语速查 + 关键数字
- [[analysis/06-source-map]] — 资料清单与扩展阅读

## Current State

覆盖范围已从"Agent 架构设计"扩展为**系统性的 AI Agent 知识体系**，主干由三类一手材料构成：

1. **厂商工程方法论** —— [[entities/01-anthropic]] 七篇工程博客（分类框架、长时 harness、Managed Agents 解耦、上下文工程、多 agent 复盘、工具设计、Code Mode）+ [[entities/07-openai]] 的落地白皮书
2. **协议与框架规范** —— [[entities/09-model-context-protocol]]（spec 2026-07-28）、[[entities/12-a2a-project]]、[[entities/11-openai-agents-sdk]]、[[entities/10-langgraph]]
3. **学术与安全基线** —— [[entities/08-lilian-weng]] 的三组件综述、28 篇基础论文摘要、[[entities/13-owasp-genai-security-project]] 的 2026 榜单、[[entities/14-simon-willison]] 的 lethal trifecta

## 学习动线

1. **分类与骨架**：[[concepts/13-agentic-systems]] → [[concepts/14-augmented-llm]] → [[concepts/21-agent-loop]]
2. **五种 Workflow 模式**：[[concepts/15-prompt-chaining]]、[[concepts/16-routing]]、[[concepts/17-parallelization]]、[[concepts/18-orchestrator-workers]]、[[concepts/19-evaluator-optimizer]]
3. **推理与规划**：[[concepts/22-planning-and-decomposition]] → [[concepts/23-react]] → [[concepts/24-reflection]]
4. **上下文与记忆**：[[concepts/09-context-engineering]] → [[concepts/29-context-rot-and-attention-budget]] → [[concepts/30-compaction-and-note-taking]] → [[concepts/26-memory]] → [[concepts/27-rag]] → [[concepts/28-agentic-search]]
5. **工具与协议**：[[concepts/20-aci]] → [[concepts/25-tool-use-and-function-calling]] → [[concepts/31-mcp]] → [[concepts/32-progressive-disclosure-and-code-execution]] → [[concepts/33-a2a]]
6. **多 Agent**：[[analysis/04-single-vs-multi-agent]] → [[concepts/34-orchestrator-worker-multi-agent]] → [[concepts/35-manager-vs-decentralized]] → [[concepts/36-handoff]]
7. **运行时解耦与长程**：[[concepts/04-brain-hands-session]] → [[concepts/02-harness]]、[[concepts/03-sandbox]]、[[concepts/01-session]] → [[concepts/10-long-running-agent]] → [[concepts/11-initializer-coding-agent]] → [[concepts/12-feature-list-pattern]]
8. **评估与工程化**：[[concepts/41-agent-evaluation]] → [[concepts/42-agent-benchmarks]] → [[concepts/43-durable-execution]] → [[concepts/45-token-economics]]
9. **安全**：[[concepts/39-prompt-injection]] → [[concepts/40-excessive-agency]] → [[concepts/37-guardrails]] → [[concepts/38-human-in-the-loop]]
10. **设计哲学**：[[concepts/08-bitter-lesson]]、[[concepts/07-meta-harness]]、[[concepts/05-pets-vs-cattle]]

## Core Insights

1. **简单优先，复杂度须验证** — [[concepts/13-agentic-systems]] 的首要原则：能用单次 LLM 调用解决就不要用 Agent；两家厂商都主张**先把单 agent 做满**再拆
2. **Workflows 与 Agents 的分野在"谁控制流程"** — 不是用了几次 LLM，而是路径由代码预定还是由模型动态决定
3. **[[concepts/14-augmented-llm]] 是一切的基础** — 检索、工具、记忆三大增强；五种 Workflow 模式是对它的逐步编排
4. **Agent 的当代定义极简** — LLM 在循环中自主使用工具（[[concepts/21-agent-loop]]）；工程复杂度都在循环之外
5. **Context 是有限资源，且边际收益递减** — [[concepts/29-context-rot-and-attention-budget]]：注意力预算的存在使"最小高信号 token 集合"成为总纲，而非"塞得越多越好"
6. **取上下文的范式正在从预检索转向 just-in-time** — [[concepts/28-agentic-search]]：存标识符、运行时取、逐层披露；生产上常落在混合策略
7. **多 Agent 有效主要因为它能花掉足够多 token** — [[concepts/34-orchestrator-worker-multi-agent]]：+90.2%、token 解释 80% 方差、但 15× 成本，只在高价值重并行任务上成立
8. **工具质量决定 agent 能力上限** — [[concepts/20-aci]] + [[sources/08-writing-effective-tools-for-agents]]：按 agent 的 affordance 设计工具、返回高信号内容、错误信息可行动；评估驱动迭代
9. **工具规模化靠渐进披露与代码执行** — [[concepts/32-progressive-disclosure-and-code-execution]]：150k → 2k token，且中间结果不必经过模型
10. **解耦是 Agent 系统扩展的基础** — [[concepts/04-brain-hands-session]] 与 [[concepts/43-durable-execution]]：状态可持久、执行可恢复、发布可渐进
11. **长时运行需要结构化交接** — [[concepts/10-long-running-agent]] 无法依赖 context window 传递状态，靠 progress file + feature list + git history 与压实/笔记/子 agent
12. **评估必须容忍多条有效路径** — [[concepts/41-agent-evaluation]]：约 20 条起步、LLM-as-judge 单调用单 prompt、会改状态的评 end-state；LLM 裁判在深度专业领域会失真（ChemCrow 反例）
13. **提示注入无法根治，只能限制爆炸半径** — [[concepts/39-prompt-injection]]：LLM 架构上不分指令与数据；防御是架构性的（最小权限 + 确定性中介 + 能力预算），护栏只降低发生率
14. **过度授权是 agent 时代的头号放大器** — [[concepts/40-excessive-agency]] 在 OWASP 2026 升至第三：功能过多 / 权限过大 / 自治过度
15. **假设会随模型进化而过时** — [[concepts/08-bitter-lesson]]：补偿模型缺陷的 scaffold 终将消融（手写 ReAct 模板已是例证），但管理稀缺资源（注意力、token、权限）的原则会留下

## Open Questions

- Workflow 模式中硬编码的编排逻辑，随模型能力增长是否会逐步消融为纯 Agent？
- 五种 Workflow 模式之外，是否存在尚未被识别的常见编排模式？
- Meta-harness 的接口设计能否真正容纳未来未知的 harness 类型？
- Context engineering 的最佳实践会收敛，还是随模型进化持续变化？
- 异步多 agent（lead 可中途操纵 subagent、subagent 之间可协调）需要什么样的状态一致性与错误传播模型？
- 记忆的"遗忘"与"更新"缺少成熟方案：文件系统路线可读可改，但何时该删、谁来判定？
- 提示注入是否存在**结构性**解法（CaMeL 类能力控制、来源标注通道），还是只能永久停留在"限制后果"？
- Agent 评估如何度量"可靠性"而非"能力上限"（`pass^k` 之外还需要什么）？
- 单一通用 coding agent vs 多 agent 架构（测试/QA/清理 agent）哪个更优？
- Feature List Pattern 能否泛化到 Web 开发以外的领域（科研、金融建模）？
