# Wiki Log

Append-only chronological record of wiki activity.

<!-- Format: ## [YYYY-MM-DD] type | Title -->
<!-- Types: ingest, query, lint -->

## [2026-04-23] ingest | Scaling Managed Agents: Decoupling the Brain, Hands, and Session

- 首次入库，来源：Anthropic 工程博客
- 创建 source 页面：[[sources/01-scaling-managed-agents-decoupling]]
- 创建 entity 页面：Anthropic, Managed Agents, Claude
- 创建 concept 页面：Session, Harness, Sandbox, Brain-Hands-Session 解耦模型, Pets vs Cattle, Context Anxiety, Meta-harness, The Bitter Lesson, Context Engineering
- 更新 index.md, home.md

## [2026-04-24] ingest | Effective Harnesses for Long-Running Agents

- 入库来源：Anthropic 工程博客，作者 Justin Young
- 创建 source 页面：[[sources/02-effective-harnesses-for-long-running-agents]]
- 创建 concept 页面：Long-Running Agent, Initializer/Coding Agent 模式, Feature List Pattern
- 更新 concept 页面：Harness（添加长时运行设计段落）, Session（添加跨 Session 状态衔接段落）, Context Anxiety（添加与 Long-Running Agent 关系段落）
- 更新 index.md, home.md

## [2026-04-25] ingest | Building Effective AI Agents

- 入库来源：Anthropic 工程博客，作者 Erik S. 和 Barry Zhang
- 创建 source 页面：[[sources/03-building-effective-ai-agents]]
- 创建 concept 页面：Agentic Systems 分类框架, The Augmented LLM, Prompt Chaining, Routing, Parallelization, Orchestrator-Workers, Evaluator-Optimizer, Agent-Computer Interface (ACI)
- 更新 concept 页面：Harness（添加与 Agentic Systems 分类框架关系段落）, The Bitter Lesson（添加"Agent 模式选择中的体现"段落）
- 更新 entity 页面：Anthropic（完善工程博客条目，添加 wiki 摘要链接）
- 更新 index.md, home.md

## [2026-06-02] lint | 全库一致性整顿

- 修复断链：创建缺失 entity 页 [[entities/04-erik-schluntz]]、[[entities/05-barry-zhang]]、[[entities/06-justin-young]]，并补入 index.md
- 统一 frontmatter：将 17 个 bare 格式页面转为 `---` 包裹的 YAML，补全 `title:`（含已有 9 个 YAML 页）
- 修复 `sources/Building-Effective-AI-Agents.md` 的 sources 路径（字符串→数组、补 `../../`）
- 为 `sources/Scaling-Managed-Agents-Decoupling.md` 的含冒号标题加引号
- 为 entity 页 Anthropic/Claude 补 `sources` 字段
- 修正 log.md 时间顺序（04-24 / 04-25 互换）

## [2026-08-11] lint | 按 `_template` 最新规范重构全库

- **文件编号**：全部 29 个内容页按加入顺序加两位序号前缀。concepts `01-session` … `20-aci`（04-23 批 01–09、04-24 批 10–12、04-25 批 13–20）；entities `01-anthropic` … `06-justin-young`；sources `01`–`03` 按入库日期。文件名统一为小写 kebab-case（`Erik-S.md` → `04-erik-schluntz.md`）。
- **Wikilinks**：全库 288 处 markdown 相对链接（`[文字](harness.md)`）转为 `[[category/NN-name]]`；链接文字与标题不一致处保留别名形式 `[[concepts/03-sandbox|执行计算]]`。指向 `raw/` 的链接仍用相对路径。
- **Frontmatter**：`sources:` 统一为 `raw/` 下的裸文件名（原先混用 `../../raw/x.md` 与 `sources/x.md` 两种写法，后者本身指错到 wiki 页）；多行列表形式统一为行内数组。
- **补断链**：[[entities/03-claude]] 此前无任何入链，[[entities/06-justin-young]] 仅被 log 引用 —— 在 [[concepts/06-context-anxiety]] 与 [[sources/02-effective-harnesses-for-long-running-agents]] 中补上引用。
- **CLAUDE.md**：改写为 `_template` 结构（Domain / Page Format / Wikilinks / File Numbering / Categories / Raw Sources / Special Pages）；删除从未使用的 `topics` 分类，新建空目录 `wiki/analysis/`。
- **导航页**：`index.md`、`home.md` 改为 wikilink 目录 + 学习动线结构，对齐 react-native 知识库的写法。

## [2026-08-11] ingest | 面试导向的系统性扩库：13 份权威一手资料

**目标**：把知识库从"Agent 架构设计"扩展为可支撑面试的完整体系（用户需求：系统学习 AI Agent，尽快达到应对面试的程度）。

**新增 raw 源（13 份）**

- Anthropic 工程博客 4 篇：`Effective-Context-Engineering-for-AI-Agents.md`、`How-We-Built-Our-Multi-Agent-Research-System.md`、`Writing-Effective-Tools-for-AI-Agents.md`、`Code-Execution-with-MCP.md`
- OpenAI 白皮书：`OpenAI-A-Practical-Guide-to-Building-Agents.pdf`（34 页原件）+ 文本抽取版 `.md`
- Lilian Weng 综述：`LLM-Powered-Autonomous-Agents.md` + 13 张配图（`raw/assets/llm-powered-autonomous-agents/`）
- 协议与框架规范：`MCP-Architecture-Overview.md`（spec 2026-07-28）、`A2A-Protocol-Overview.md`、`OpenAI-Agents-SDK-Documentation.md`（7 章合辑）、`LangGraph-Workflows-and-Agents.md`
- 安全：`OWASP-GenAI-LLM-Top-10-2026.md`（LLM00–LLM10 全文，2026-08-04 发布）、`The-Lethal-Trifecta-for-AI-Agents.md`
- 论文：`Foundational-Agent-Papers-Abstracts.md`（28 篇 arXiv 元数据与摘要原文，按主题分组）

**新增 wiki 页（52 页，内容页 29 → 81）**

- sources `04`–`16`（13 页）：每份新 raw 源一页摘要
- concepts `21`–`45`（25 页）：agent-loop、planning、react、reflection、tool-use、memory、rag、agentic-search、context-rot、compaction、mcp、code-execution、a2a、orchestrator-worker、manager-vs-decentralized、handoff、guardrails、hil、prompt-injection、excessive-agency、evaluation、benchmarks、durable-execution、generative-agents、token-economics
- entities `07`–`14`（8 页）：OpenAI、Lilian Weng、MCP 项目、LangGraph、Agents SDK、A2A Project、OWASP GenAI、Simon Willison
- analysis `01`–`06`（6 页，此前该目录为空）：面试路线图、高频题库、框架对比、单 vs 多 agent 决策、术语速查、资料清单

**更新既有页**：`09-context-engineering`（补 2025–2026 方法论化与新链接）、`13-agentic-systems`、`14-augmented-llm`、`10-long-running-agent`、`20-aci`（补实操版与安全链接）、`entities/01-anthropic`（补 3 篇博客 + MCP）；重写 `index.md`、`home.md`（Core Insights 10 → 15 条，Open Questions 补 4 条）。

**校验**：全库 wikilink 断链扫描通过（84 个 md 文件，唯一告警为 `log.md` 中的格式示例文本）；论文数字（τ-bench `pass^8 <25%`、GAIA 92% vs 15%、MemGPT 虚拟内存分页等）逐条比对 arXiv 摘要原文。

## [2026-08-12] lint | 新增内容的事实与一致性复核

对上一次 ingest 的 52 个新页做独立复核，发现并修正 7 处：

- **归属无源**：A2A "由 Linux Foundation 托管"在存档 README 正文中并无依据（该字样只出现在我自己写的 archive frontmatter 里）。经 Google Developers Blog / Linux Foundation 公告核实为真（2025-04 发布、2025-06 捐赠、TSC 含 AWS/Cisco/Google/IBM/Microsoft/Salesforce/SAP/ServiceNow），已在 [[sources/13-a2a-protocol-overview]] 与 [[entities/12-a2a-project]] 补出处并明确标注"不在存档正文内"
- **计数错误**：`home.md` 写 Anthropic "六篇工程博客"，实为 **7** 篇（括注本身列了 7 项）→ 修正
- **计数不精确**：`index.md` 的题库描述"40+ 道"→ 实测 **47** 道
- **易混数字**：[[analysis/01-interview-roadmap]] 冲刺段的"必背数字"把 GAIA 的 15% 与多 agent 的 15× 并列成裸数字，易误读 → 改为带标签的完整表述
- **来源边界未声明**：[[concepts/27-rag]] 的分块/混合检索/rerank 属行业通行实践、不来自任何 `raw/` 存档 → 页内加说明，区分"有出处的结论"与"常规做法"
- **frontmatter 越界字段**：`sources/02` 存在约定外的 `author:` 键（早于本次 ingest）→ 移入正文并链到 [[entities/06-justin-young]]
- **格式统一**：全库 20 处 ASCII 图/伪代码围栏缺语言标记（其中 13 处为旧页）→ 统一标为 `text`，消除 MD040 告警

**已核实无误的抽查项**：LangGraph `Send` API（raw 行 679）、RAG 原论文"两种 formulation"、2308.11432 的统一框架表述、2309.07864 的 brain-perception-action 与三类场景、Self-Consistency/Plan-and-Solve/AutoGen 的一句话概括、OpenAI 白皮书的"先最大化单 agent 能力"与"15+ 清晰工具 vs <10 重叠工具"、多 agent 的 90.2%/80%/4×/15×、Claude Code 的 25,000 token 上限与最近 5 个文件、tool-testing agent 的 40%。

**结构校验**：wikilink 断链 0、孤儿页 0、`sources:` 指向的 raw 文件全部存在、相对路径链接全部可达、表格列数与代码围栏配对全部正确。
