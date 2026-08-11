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
