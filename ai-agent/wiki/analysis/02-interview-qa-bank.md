---
title: AI Agent 高频面试题与答题要点
date: 2026-08-11
tags: [analysis, interview, qa, drill]
sources: [Building-Effective-AI-Agents.md, OpenAI-A-Practical-Guide-to-Building-Agents.md, Effective-Context-Engineering-for-AI-Agents.md, How-We-Built-Our-Multi-Agent-Research-System.md, Writing-Effective-Tools-for-AI-Agents.md, Code-Execution-with-MCP.md, MCP-Architecture-Overview.md, OWASP-GenAI-LLM-Top-10-2026.md, Foundational-Agent-Papers-Abstracts.md]
---

# AI Agent 高频面试题与答题要点

**用法**：每题限时 90 秒口述，卡住就回到链接页面补。答案是**要点**不是逐字稿 —— 面试要的是判据与取舍，不是背书。

---

## A. 基础概念（★★★★★）

**A1. 什么是 AI Agent？和 workflow、chatbot 的区别？**
- Agent = **LLM 在循环中自主使用工具**（Anthropic 现用定义）
- Workflow：LLM 与工具**沿预定义代码路径**编排；Agent：模型**动态决定自己的过程与工具使用**
- 判据不是用了几次 LLM，而是**谁在控制流程**。不用 LLM 控制工作流执行的（简单 chatbot、单轮调用、分类器）都不是 agent
- → [[concepts/13-agentic-systems]]、[[concepts/21-agent-loop]]

**A2. 一个 agent 的最小构成？**
- **Model + Tools + Instructions**；加上 run 的概念（循环直到退出条件）
- 退出条件：无工具调用的最终输出 / 指定结构化输出 / 出错 / `max_turns` / 外部中断
- → [[concepts/21-agent-loop]]

**A3. 什么时候**不**该用 agent？**
- 确定性方案够用时；任务简单到单次 LLM 调用 + 检索/few-shot 就能解决时
- agent 用延迟与成本换任务性能，要判断这笔交易值不值
- 优先选 agent 的场景：复杂决策（退款审批）、规则难维护（供应商安全审查）、重度非结构化数据（保险理赔）
- → [[sources/05-openai-practical-guide-to-building-agents]]

**A4. Augmented LLM 是什么？**
- LLM + 检索 + 工具 + 记忆，是所有 agent 系统的基础构件；五种 workflow 模式都是对它的逐步编排
- 与 Lilian Weng 的 Planning/Memory/Tool use 三组件是同一事物的两种表述
- → [[concepts/14-augmented-llm]]、[[sources/04-llm-powered-autonomous-agents]]

**A5. 五种 workflow 模式各是什么，举例？**
- **Prompt chaining**：串行分解（先写大纲再写正文，中间可加 gate 检查）
- **Routing**：分类后路由到专门处理（客服问题分流到退款/技术/通用）
- **Parallelization**：分片并行或多次投票（分节写报告；多模型投票判违规）
- **Orchestrator-workers**：中心 LLM **运行时**决定子任务数量与内容（改多个文件的代码修改）
- **Evaluator-optimizer**：生成 ↔ 评估成环迭代（文学翻译润色）
- 关键区别：Parallelization 的分支**编译期已知**，Orchestrator-workers **运行时决定**
- → [[concepts/15-prompt-chaining]]…[[concepts/19-evaluator-optimizer]]

**A6. ReAct 是什么？今天还需要手写吗？**
- 交错生成 reasoning 与 action：Thought → Action → Observation 循环；实验证明优于去掉 Thought 的 Act-only 基线
- **今天通常不需要手写**：function calling 原生支持工具调用（不必解析文本协议），extended/interleaved thinking 把 thought 变成一等能力
- 但结构仍在，且"工具输出重新进入 context"这一机制是后来提示注入问题的结构根源
- → [[concepts/23-react]]

**A7. Reflexion / Self-Refine 的原理与前提？**
- Reflexion：二元奖励 + 启发式判定失败（轨迹太长=低效规划；连续相同动作导致相同观察=幻觉），把语言化反思写入工作记忆（最多 3 条）再重试 —— **用语言而非梯度做 RL**
- Self-Refine：同一模型 生成→自评→修订 迭代
- **前提是有可靠反馈信号**：编译错误、测试失败、类型检查有效；自由文本质量与深度专业判断易自我欺骗（ChemCrow 反例）
- → [[concepts/24-reflection]]

**A8. CoT / ToT / Self-Consistency 的关系与代价？**
- CoT 单路径；Self-Consistency 采样多路径投票（N 倍成本，需唯一答案）；ToT 每步多候选构成树 + 评估器 + BFS/DFS 可回溯（成本更高，需设计状态评估器）
- 本质都是**用测试时计算换准确率**；推理模型已把这部分内建 → [[concepts/08-bitter-lesson]] 的例证
- → [[concepts/22-planning-and-decomposition]]

---

## B. 编排与多 Agent（★★★★★）

**B1. 单 agent 还是多 agent，怎么决定？**
- 默认先把单 agent 做满（两家厂商一致）
- 触发拆分：prompt 分支爆炸、工具**重叠**混乱、context 装不下、需要并行加速
- 拆之前先修 ACI（改名/改描述/改参数、拆模板）
- 代价：多 agent ≈ 15× token；协调复杂度、涌现行为、同步瓶颈、信息损耗
- 中间选项：**用 subagent 做上下文隔离**（不是分工），成本低得多
- → [[analysis/04-single-vs-multi-agent]]

**B2. Manager 模式与去中心化模式的区别？**
- Manager（agents as tools）：通过 tool call 调专家，**控制权不转移**，manager 负责综合，只有它面对用户
- Decentralized（handoffs）：**单向移交**执行权与会话状态，接管者直接与用户交互
- 图论上：manager 的边是 tool call，去中心化的边是 handoff
- 判据："这个任务需要有人综合吗？"
- → [[concepts/35-manager-vs-decentralized]]、[[concepts/36-handoff]]

**B3. 多 agent 真的更好吗？给数据。**
- Anthropic 内部 research eval：Opus 4 lead + Sonnet 4 subagent **比单 agent Opus 4 高 90.2%**
- 归因：BrowseComp 上 **token 用量单独解释 80% 性能方差**，加工具调用次数与模型选择共 95% → **多 agent 有效主要因为它能花掉足够多 token**
- 但成本：agent ≈ chat 4×，多 agent ≈ 15×
- 不适合：需共享同一 context、agent 间依赖多的领域（**大多数编码任务可并行部分远少于研究任务**）
- → [[concepts/34-orchestrator-worker-multi-agent]]

**B4. orchestrator 怎么委派才有效？**
- 每个 subagent 需要：目标、输出格式、工具与来源指引、清晰任务边界
- 反例："research the semiconductor shortage" → 一个查 2021 汽车芯片危机，两个重复查 2025 供应链
- 把**努力缩放规则**写进 prompt：简单事实 1 agent/3–10 次调用；比较 2–4 agent 各 10–15 次；复杂研究 10+ agent 且职责划分
- 两层并行（lead 起 3–5 个 subagent；每个用 3+ 工具）→ 研究时间最多降 90%
- → [[concepts/34-orchestrator-worker-multi-agent]]

**B5. 子 agent 的结果怎么回传？**
- 只回传 **1k–2k token 蒸馏摘要**，不回传原始探索内容
- 更好的做法：subagent **直接写文件系统**，只回传轻量引用 —— 避免"传话游戏"与 token 复制，适合代码、报告、可视化
- → [[concepts/30-compaction-and-note-taking]]

**B6. 多 agent 系统怎么调试？**
- 加**全链路 tracing**，监控**决策模式与交互结构**（Anthropic 不看单个会话内容以保护隐私）
- 注意涌现行为：改 lead 的 prompt 会不可预测地改变 subagent 行为 → 评估交互模式而非单体
- → [[concepts/43-durable-execution]]

---

## C. 上下文与记忆（★★★★★）

**C1. Context engineering 与 prompt engineering 的区别？**
- prompt engineering：写好指令（尤其 system prompt），面向一次性任务
- context engineering：**在推理过程中策展与维护整个 token 状态**（指令、工具、MCP、外部数据、历史）；写 prompt 是离散动作，**策展是每轮都要重做的循环动作**
- 目标一句话：**找到能最大化目标达成概率的、最小的高信号 token 集合**
- → [[concepts/09-context-engineering]]、[[sources/06-effective-context-engineering]]

**C2. 为什么不能把资料全塞进长上下文？**
- **context rot**：token 增加 → 准确召回能力下降（所有模型都有，衰减陡缓不同）
- 机制：n² 对关系被摊薄；训练数据里长序列稀少；位置编码插值带来退化 → 形成**性能梯度而非硬悬崖**
- 结论：模型有**注意力预算**，每个 token 都在消耗它
- 且原文明确：可预见未来里各种大小窗口都会受 context 污染与相关性问题影响，**别指望窗口变大解决问题**
- → [[concepts/29-context-rot-and-attention-budget]]

**C3. 任务超出 context window 怎么办？**
三种技术 + 适用边界：
- **Compaction**：总结后用摘要重启（保留架构决策、未解决 bug、实现细节，丢弃冗余工具输出 + 带最近 5 个文件）→ 适合大量来回的对话流；调优先最大化 recall 再提 precision；最轻量安全的形式是**清理历史工具结果**
- **结构化笔记**：写 `NOTES.md` / to-do 再读回 → 适合有清晰里程碑的迭代开发（Claude 玩 Pokémon 跨数千步维持计数）
- **子 agent**：干净 context 深挖 + 蒸馏回传 → 适合并行探索有回报的研究
- → [[concepts/30-compaction-and-note-taking]]

**C4. Agent 的记忆怎么设计？**
- 先分层：会话内（短期，消息历史）/ 跨会话（长期，用户偏好与项目状态）/ 程序性（技能、可复用代码）
- 再看状态放哪：应用内存手动传 / SDK session / 服务端 conversation_id / previous_response_id —— **一个会话只选一种**
- 长期记忆两条路线：**向量检索**（MIPS + ANN：LSH/ANNOY/HNSW/FAISS/ScaNN）与 **文件系统/结构化笔记**（可读、可审计、易更新，元数据本身是信号）
- 检索排序可用 Generative Agents 的三因子：**相关性 + 新近性 + 重要性（直接问 LLM 打分）**
- MemGPT 心智模型：context window 是内存不是硬盘，记忆管理本质是**分页与置换**
- → [[concepts/26-memory]]

**C5. 讲讲 RAG 的完整管线与优化点。**
- 离线：加载 → 清洗 → **分块** → embedding → 向量库（+元数据）
- 在线：query 改写 → 检索（**混合：向量 + BM25**，RRF 融合）→ **rerank（cross-encoder）** → 组装 → 生成 → 引用校验
- 优化点：小块检索 + 返回父块、句窗检索、元数据过滤、多查询/HyDE/子问题分解、上下文压缩
- 三代：Naive → Advanced → Modular；全局性问题用 **GraphRAG**（实体图 + 社区摘要）
- 评估要**分开评检索与生成**（recall@k/nDCG vs 忠实度/引用准确性）
- → [[concepts/27-rag]]

**C6. 静态 RAG 与 agentic search 怎么选？**
- 静态 RAG：一次取相似 chunk 就生成；快、可预测；但相似≠相关、索引会过期、无法多跳
- **Just-in-time**：只存轻量标识符（路径/查询/链接），运行时用工具取；能多跳、判断来源质量、不受索引过期影响；代价是慢且依赖工具质量
- **混合最实用**：稳定核心资料预置（如 `CLAUDE.md`），其余即时检索（Claude Code 的做法）；法律/金融这类静态内容更适合混合
- → [[concepts/28-agentic-search]]

**C7. system prompt 应该写多细？**
- 在"**正确的海拔**"：一端是硬编码脆弱 if-else（脆且难维护），一端是空泛高层指导（缺具体信号）
- 做法：分节组织（`<background_information>`、`<instructions>`、`## Tool guidance`），追求**最小但完整**（minimal ≠ short）；先用最强模型测最小 prompt，再按失败模式补
- few-shot：**不要堆边界情况清单**，要策展多样、典型的范例
- → [[sources/06-effective-context-engineering]]

---

## D. 工具与协议（★★★★☆）

**D1. 一次 function call 在系统里发生了什么？**
1. 应用把工具定义（name + description + JSON Schema）随请求发给模型
2. 模型输出**结构化调用请求**（不是自然语言）
3. **应用执行**该函数 —— 凭证与执行权在你的代码里（安全边界）
4. 结果作为 tool result 追加进历史
5. 模型基于结果继续
- 补充：并行工具调用降延迟；`tool_choice` 可强制；模型只"请求"不"执行"
- → [[concepts/25-tool-use-and-function-calling]]

**D2. 怎么给 agent 设计工具？**
- **别把 API 端点逐个包成工具**：`search_contacts` > `list_contacts`；`search_logs` > `read_logs`；`get_customer_context` > 三个分散 get；`schedule_event` 合并"查空档+建日程"
- 命名空间：`asana_search` / `asana_projects_search`
- 返回高信号：语义化名称优于 UUID（显著减少幻觉）；`response_format: concise|detailed`
- token 效率：分页/过滤/截断 + 合理默认（Claude Code 默认 25,000 token 上限）
- **错误信息要可行动**；描述当 prompt 写（像给新同事讲）；参数 `user_id` 而非 `user`
- **Poka-yoke**：改设计让错误不可能发生（强制绝对路径）
- 证据：精修工具描述让 Claude Sonnet 3.5 拿下 SWE-bench Verified SOTA
- → [[sources/08-writing-effective-tools-for-agents]]、[[concepts/20-aci]]

**D3. 工具太多怎么办？**
- 先判断：问题通常不是数量而是**重叠度**（有系统管好 15+ 清晰工具，也有栽在 10 个重叠工具上）
- 三档解法：白名单/精选 → **渐进披露**（文件树按需读 / `search_tools` 带详略级别）→ **代码执行**（把 MCP server 当代码 API，**150k → 2k token，-98.7%**）
- → [[concepts/32-progressive-disclosure-and-code-execution]]

**D4. MCP 是什么？三原语？**
- 开放标准（2024-11 由 Anthropic 发布），把"每对 agent×工具都要定制集成"变成 M+N
- 三角色：Host → 每个 server 一个 Client → Server
- 两层：Data（JSON-RPC 2.0）+ Transport（STDIO 本地 / Streamable HTTP 远程，推荐 OAuth）
- **Server 三原语：Tools（做动作）/ Resources（读数据）/ Prompts（模板）**；Client 侧有 **Elicitation**
- 版本注意（`2026-07-28`）：协议**无状态**（`_meta` 带版本与能力）+ 强制 `server/discover`；**Sampling 与 Logging 已废弃**；变更通知 opt-in（`subscriptions/listen`）；Tasks 扩展给长任务持久句柄
- → [[concepts/31-mcp]]

**D5. MCP 与 function calling、与 A2A 的区别？**
- function calling 是**模型能力**；MCP 是**分发与集成协议**（工具从哪来、怎么发现、怎么传输）
- MCP：agent↔工具/数据（对端透明）；A2A：agent↔agent（对端**不透明**，用 Agent Card 发现，JSON-RPC over HTTP + SSE + push）
- 可同时用：A2A server agent 内部用 MCP 连工具
- 加分：**说清大多数"多 agent"不需要 A2A**（同进程 orchestrator-worker 即可）
- → [[concepts/33-a2a]]

**D6. 代码执行（Code Mode）的收益与代价？**
- 收益：渐进披露、结果在执行环境过滤（1 万行 → 打印 5 行）、用代码写控制流（省"首 token 延迟"）、隐私（中间结果不进模型，可自动 tokenize PII）、状态与 **Skills** 积累
- 代价：需要安全**沙箱** + 资源限制 + 监控，运维与安全开销是直接 tool call 没有的
- → [[sources/09-code-execution-with-mcp]]

---

## E. 评估与工程化（★★★★☆）

**E1. Agent 怎么评估？为什么比传统测试难？**
- 难点：非确定性 —— 同样起点会走**不同的有效路径**，不能断言"输入X→路径Y→输出Z"
- 方法：**约 20 条真实查询立刻开始**（早期效应量大，30%→80% 的改动几条样本就能看出）
- **LLM-as-judge**：单次调用、单 prompt、输出 0.0–1.0 + pass/fail（比多裁判分工更一致）；rubric 五维：事实准确性、引用准确性、完整性、来源质量、工具效率
- **人工评估**捕捉自动化遗漏（如系统性偏好 SEO 内容农场而非权威学术 PDF）
- **end-state evaluation**：会改状态的 agent 评最终状态而非逐轮，复杂流程拆 checkpoint
- 指标：成功率、`pass^k`、工具调用次数/错误率、token、延迟、护栏触发率
- → [[concepts/41-agent-evaluation]]

**E2. LLM-as-judge 的边界在哪？**
- ChemCrow 反例：LLM 评估认为 GPT-4 与 ChemCrow 相当，化学专家认为 ChemCrow 大幅胜出 —— **在需要深度专业知识的领域，模型不知道自己不知道**
- 对策：有明确答案时用它验证；主观质量维度配人工抽检；rubric 具体化；避免让同一模型既生成又评判关键结论

**E3. 知道哪些 agent benchmark？**
- **SWE-bench**（真实 GitHub issue → 补丁）、**τ-bench**（工具-agent-用户三方，比对最终数据库状态，提出 **`pass^k`**；gpt-4o 成功率 <50%，零售域 pass^8 <25%）、**GAIA**（人类 92% vs 带插件 GPT-4 15%）、**AgentBench**（8 环境）、**WebArena**（可复现真实网站）、**BrowseComp**、**AgentDojo/JailbreakBench**（注入与越狱）
- 读法：区分能力上限与**可靠性**；警惕污染与 scaffold 差异；基准用于选模型，业务靠自建 eval
- → [[concepts/42-agent-benchmarks]]

**E4. 怎么把 agent 上生产？（五层清单）**
1. **状态**：checkpoint + 可从中断处恢复（不重跑）；模型适应性 + 确定性重试
2. **可观测**：全链路 trace、决策模式监控、成本/延迟/错误指标
3. **安全**：护栏 + 最小权限 + 高风险动作审批
4. **成本**：token 预算、缓存、模型分级
5. **发布**：**rainbow deployment** 渐进切流（agent 可能停在流程任意位置）+ 回归 eval 门禁
- → [[concepts/43-durable-execution]]

**E5. 怎么控制 agent 成本与延迟？**
- 先度量：每任务 token/成本/延迟、缓存命中率、工具调用分布
- 架构级：渐进披露/代码执行、子 agent 蒸馏、混合检索、compaction
- 参数级：模型分级（**先用最强模型建基线再换小模型**）、prompt caching（稳定前缀在前）、并行工具调用/并行 subagent、截断与分页、**努力缩放规则**
- 判据：任务价值 vs 15× 成本；延迟敏感→预检索，质量优先→agentic search
- 注意 `max_turns` 与限流（对应 OWASP **LLM06 Unbounded Consumption**）
- → [[concepts/45-token-economics]]

**E6. 长时间运行（数小时/跨天）的 agent 怎么做？**
- 上下文侧：compaction + 结构化笔记 + 子 agent
- 状态侧：progress file + feature list + git history 三重机制交接（Initializer agent 建环境，Coding agent 每次增量推进并留清晰产物）
- 执行侧：checkpoint、可恢复、审批中断、rainbow 部署
- 注意 **context anxiety**：模型感知窗口将满会提前收工，要在 harness 里处理
- → [[concepts/10-long-running-agent]]、[[concepts/11-initializer-coding-agent]]、[[concepts/12-feature-list-pattern]]、[[concepts/06-context-anxiety]]

**E7. 用什么框架？为什么？**
- 先讲 Anthropic 立场：**先不用框架**，用 API 手写循环，因为框架会遮住"实际发给模型的是什么"，更难调试
- 两阵营：声明式图（LangGraph：checkpoint + interrupt + 精细状态）vs code-first（Agents SDK：概念负担低、动态流程自然）
- 三条判据：控制权（是否需要状态检查/恢复）、工作流动态程度、可观测与 eval 怎么接
- → [[analysis/03-framework-comparison]]

---

## F. 安全（★★★☆☆ 但极易挂）

**F1. 什么是 prompt injection？为什么无法根治？**
- 根因：**LLM 在架构上不区分指令与数据**（同一 token 流），没有参数化查询式的干净解法；且无法可靠按来源区分指令重要性
- agent 场景三个恶化因素：**context pooling**（无信任边界）、**memory persistence**（污染此后每个会话）、**agentic execution**（爆炸半径扩到工具可达之处，工具输出又回到 context 形成链式效应）
- **区分 jailbreaking**：注入是应用架构问题，越狱是模型安全问题
- → [[concepts/39-prompt-injection]]

**F2. Lethal Trifecta 是什么？**
- **访问私有数据 + 暴露于不可信内容 + 对外通信能力**，三者同时具备即高危；移除任一条即破除条件
- "对外通信"极易被忽略：**任何能发 HTTP 请求的工具**（调 API、加载图片、甚至生成一个可点链接）都是外传通道
- MCP 直接放大它（鼓励混搭工具）；GitHub 官方 MCP server 漏洞就是三要素集于一体
- → [[sources/16-lethal-trifecta]]

**F3. 间接注入的典型攻击路径？**
- 攻击者把文本放进**用户信任的位置**（issue、反馈表单、支持工单），等用户的 MCP 连接 agent **以用户的高权限凭证**去读 → **执行特权动作的是 agent**
- 其他面：RAG 语料投毒（**约 5 篇文档 → 约 90% 成功率**）、多模态隐写、不可见 Unicode（走私指令或外传字节）、记忆投毒
- → [[concepts/39-prompt-injection]]

**F4. 怎么缓解？**
- 立场：**假设指令边界终将被突破**，转而约束模型能做什么、输出能触达什么（架构性 > 拦截性）
- 承重控制：**凭证与状态变更能力放在应用代码，逐操作最小权限，特权调用走确定性策略引擎复核**；**Rule of Two** 作为能力预算下限（A 不可信输入 / B 敏感数据 / C 状态变更或对外通信，三者齐备必须逐动作人工审批）
- 其他：schema 校验、跨模态过滤、剥离不可见字符、来源标注通道、**把记忆写入当作特权操作**、pin/签名 MCP server 并审计工具描述、**用自适应攻击者测试**（静态成功率近 0 而自适应 >90% 的防御很常见）
- 护栏的定位：降低发生率，不能作为唯一依赖（"95% 在 Web 安全里是不及格分"）
- → [[concepts/39-prompt-injection]]、[[concepts/37-guardrails]]

**F5. Excessive Agency 的三大根因与控制？**
- 根因：**功能过多 / 权限过大 / 自治过度**（OWASP 2026 升至第三位）
- 七条根本控制：最小化工具、最小化工具功能、避免开放式工具（`run_shell`/`fetch_url`）、最小化权限（数据库层只读限表）、**在用户上下文中执行并在多 agent 链路保留原始授权范围**、高影响动作人工批准、**完全中介**（授权用代码判定 + audit→warn→block→escalate 分级）
- 两条限损：监控工具使用、限流与熔断（可上下文感知，如累计金额阈值）
- 经典例子：邮件助手被劫持 —— 三种独立修法正对应三大根因
- → [[concepts/40-excessive-agency]]

**F6. 护栏有哪些类型？**
- relevance / safety（越狱注入）/ PII / moderation / **tool safeguards**（按只读vs写入、可逆性、权限、资金影响打风险分）/ rules-based（黑名单、长度、正则）/ output validation
- **分层防御**：单一护栏不够；乐观执行 + tripwire 不牺牲延迟
- 建设顺序：先隐私与内容安全 → 按真实失败逐步加 → 同时优化安全与体验
- → [[concepts/37-guardrails]]

**F7. 什么时候必须人工介入？**
- 两个触发器：**超过失败阈值**（重试/动作次数超限）、**高风险动作**（敏感、不可逆、高影响：取消订单、大额退款、发起支付）
- 分级执行避免审批疲劳；给人看的必须是**真实将执行的动作**而非摘要（不可见字符走私会让两者不一致）
- → [[concepts/38-human-in-the-loop]]

---

## G. 排障与开放题（面试后半程）

**G1. Agent 陷入死循环怎么办？**
- 检测：连续相同动作导致相同观察（Reflexion 的幻觉判据）；轮次/时长阈值
- 处理：`max_turns` 上限、限流熔断、把失败信息喂回让它调整、必要时升级到人
- 预防：工具错误信息要可行动；工具边界清晰减少反复试错

**G2. Agent 选错工具 / 用错参数，怎么系统性解决？**
- 先看指标：大量参数无效错误 → 描述与示例不足；大量冗余调用 → 分页/token 上限默认值不合理
- 改 ACI：命名空间、参数改名、加示例、Poka-yoke、合并重叠工具
- 用 agent 帮你改：把 eval transcript 交给 Claude Code 批量重构工具描述（**Anthropic 的 tool-testing agent 让任务完成时间降 40%**）
- 仍不行再考虑拆多 agent

**G3. Agent "找不到明显的信息"，用户投诉，怎么定位？**
- 没有 trace 就查不了：需要全链路 tracing 看是查询写得差、来源选得差、还是工具失败
- 常见原因：查询过长过具体（应**先宽后窄**）、工具选择错误（Slack 里的信息去搜网）、来源质量偏差
- 修法：prompt 里加显式启发式（先看全部工具、按意图匹配工具、优先专用工具、来源质量规则）

**G4. 上线后成本翻了 10 倍，怎么查？**
- 分解：工具定义占用 / 中间结果穿透 / 历史增长 / 单次工具响应过大 / 模型选择 / 重复无缓存前缀 / 无界循环
- 对应手段见 [[concepts/45-token-economics]] 的表；优先做渐进披露与结果过滤（往往同时提升准确率，因为减少的是低信号 token）

**G5. 你怎么看 agent 领域未来 1–2 年的变化？**（价值观题）
- 模型越强，**规定性的工程越少**：补偿模型缺陷的 scaffold 会消融（[[concepts/08-bitter-lesson]]），但管理稀缺资源（注意力/token/权限）的原则会留下
- 具体预期：更长的自主时长与更强的错误恢复；工具与上下文的按需加载成为默认；异步多 agent 协调成熟；安全从"拦截"转向"能力预算与确定性中介"
- 加分：明确说出**哪些当前做法你预期会过时**（如手写 ReAct 模板、精细的 prompt 分支、静态预检索），体现判断力而非罗列

**G6. 讲一个你做过的 agent 项目。**（准备一个覆盖以下要素的故事）
- 为什么需要 agent（而不是 workflow 或单次调用）→ 架构选择与代价
- 遇到的**具体失败模式** → 用什么**指标**发现 → 改了什么 → **结果数字**
- 安全与成本上做了什么权衡
- 如果重做会怎么改（体现反思）

---

## 相关

- 学习顺序：[[analysis/01-interview-roadmap]]
- 术语速查：[[analysis/05-glossary]]
- 资料清单：[[analysis/06-source-map]]
