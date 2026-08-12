# AI Agent — Index

按类别组织的全部 wiki 页面目录，每次 ingest 更新。共 **81 个内容页**（另有 home/index/log 三个导航页）。序号反映加入顺序。

## Overview

- [[home]] — 顶层综合与导航
- [[analysis/01-interview-roadmap]] — **从这里开始**：面试导向的 14 天 / 7 天学习路线

## Concepts

### 架构与形态

- [[concepts/01-session]] — Agent 运行中的追加写入事件日志，持久化存储
- [[concepts/02-harness]] — Agent 的"大脑"，调用 Claude 并路由工具的循环
- [[concepts/03-sandbox]] — Agent 的"双手"，代码执行和文件编辑环境
- [[concepts/04-brain-hands-session]] — 将 Agent 虚拟化为三个可独立替换的组件
- [[concepts/05-pets-vs-cattle]] — 基础设施运维的经典比喻，Pet 不可丢失 vs Cattle 可替换
- [[concepts/07-meta-harness]] — 不限定特定 harness 实现的系统设计
- [[concepts/13-agentic-systems]] — Anthropic 对 Agentic Systems 的统一分类：Workflows vs Agents
- [[concepts/14-augmented-llm]] — 一切 Agent 系统的基础构件：LLM + 检索 + 工具 + 记忆
- [[concepts/21-agent-loop]] — Agent 主循环：Model + Tools + Instructions 与退出条件

### 编排模式

- [[concepts/15-prompt-chaining]] — 串行分解任务的 Workflow 模式
- [[concepts/16-routing]] — 分类输入并路由到专门处理的 Workflow 模式
- [[concepts/17-parallelization]] — 并行分片/投票的 Workflow 模式
- [[concepts/18-orchestrator-workers]] — 中心 LLM 动态拆解委派的 Workflow 模式
- [[concepts/19-evaluator-optimizer]] — 生成-评估循环迭代的 Workflow 模式
- [[concepts/34-orchestrator-worker-multi-agent]] — 生产级多 agent：lead + subagent，含 90.2% / 15× 数据
- [[concepts/35-manager-vs-decentralized]] — Manager（agents as tools）vs 去中心化（handoffs）
- [[concepts/36-handoff]] — 执行权与会话状态的单向移交

### 推理与规划

- [[concepts/22-planning-and-decomposition]] — CoT / ToT / Plan-and-Solve / ReWOO / LLM Compiler 谱系
- [[concepts/23-react]] — Thought-Action-Observation 循环，当代 agent 循环的原型
- [[concepts/24-reflection]] — Reflexion / Self-Refine：用语言而非梯度改进
- [[concepts/44-generative-agents]] — 记忆流三因子检索与 Voyager 技能库

### 上下文与记忆

- [[concepts/06-context-anxiety]] — Claude 感知 context window 将满时提前收工的行为
- [[concepts/09-context-engineering]] — 管理 LLM context window 内容的工程实践
- [[concepts/10-long-running-agent]] — 跨多个 context window 持续工作的 Agent
- [[concepts/11-initializer-coding-agent]] — 双 Agent 分工：Initializer 搭建环境，Coding Agent 增量推进
- [[concepts/12-feature-list-pattern]] — 用结构化 JSON 拆解需求并追踪完成状态
- [[concepts/26-memory]] — 记忆分层、四种状态策略、向量 vs 文件系统两条路线
- [[concepts/27-rag]] — 检索增强生成：管线、三代分类、GraphRAG、评估与投毒
- [[concepts/28-agentic-search]] — Just-in-time context 与渐进披露
- [[concepts/29-context-rot-and-attention-budget]] — 长上下文退化的机制与注意力预算
- [[concepts/30-compaction-and-note-taking]] — 长程三术：压实 / 结构化笔记 / 子 agent

### 工具与协议

- [[concepts/20-aci]] — 为 Agent 设计工具接口，类比 HCI
- [[concepts/25-tool-use-and-function-calling]] — 工具调用机制、三类工具与设计清单
- [[concepts/31-mcp]] — Model Context Protocol：三角色、两层、三原语
- [[concepts/32-progressive-disclosure-and-code-execution]] — Code Mode：150k → 2k token
- [[concepts/33-a2a]] — Agent2Agent 协议与"何时不需要它"

### 评估与工程化

- [[concepts/41-agent-evaluation]] — 小样本起步、LLM-as-judge、end-state 评估、工具评估
- [[concepts/42-agent-benchmarks]] — SWE-bench / τ-bench / GAIA / AgentBench / WebArena
- [[concepts/43-durable-execution]] — 持久执行、tracing、rainbow 部署
- [[concepts/45-token-economics]] — token 即成本即注意力，降本手段表

### 安全

- [[concepts/37-guardrails]] — 七类护栏、分层防御、乐观执行与其局限
- [[concepts/38-human-in-the-loop]] — 两个触发器与分级执行策略
- [[concepts/39-prompt-injection]] — 根因、三轴解剖、lethal trifecta、Rule of Two、11 条缓解
- [[concepts/40-excessive-agency]] — 功能过多 / 权限过大 / 自治过度与九条控制

### 设计哲学

- [[concepts/08-bitter-lesson]] — Sutton：通用方法胜过特定设计

## Entities

- [[entities/01-anthropic]] — AI 安全公司，Claude 系列模型的开发商
- [[entities/02-managed-agents]] — Anthropic 托管式 Agent 服务，解耦架构
- [[entities/03-claude]] — Anthropic 的大语言模型系列
- [[entities/04-erik-schluntz]] — Anthropic 工程师，Building Effective AI Agents 合著者
- [[entities/05-barry-zhang]] — Anthropic 工程师，Building Effective AI Agents 合著者
- [[entities/06-justin-young]] — Anthropic 工程师，Effective Harnesses for Long-Running Agents 作者
- [[entities/07-openai]] — 与 Anthropic 并列的两大 agent 方法论来源
- [[entities/08-lilian-weng]] — Lil'Log 作者，三组件架构综述的提出者
- [[entities/09-model-context-protocol]] — MCP 开放标准与开源项目
- [[entities/10-langgraph]] — LangChain / LangGraph / Deep Agents / LangSmith 技术栈
- [[entities/11-openai-agents-sdk]] — code-first agent 框架
- [[entities/12-a2a-project]] — Agent2Agent 协议项目（Linux Foundation）
- [[entities/13-owasp-genai-security-project]] — LLM Top 10 与 Agentic Top 10 的维护方
- [[entities/14-simon-willison]] — prompt injection 与 lethal trifecta 的提出者

## Sources

- [[sources/01-scaling-managed-agents-decoupling]] — Anthropic：解耦 Brain、Hands、Session 的架构设计
- [[sources/02-effective-harnesses-for-long-running-agents]] — Anthropic：Initializer/Coding Agent 双模式
- [[sources/03-building-effective-ai-agents]] — Anthropic：分类框架与五种 Workflow 模式
- [[sources/04-llm-powered-autonomous-agents]] — Lilian Weng：Planning / Memory / Tool use 三组件综述
- [[sources/05-openai-practical-guide-to-building-agents]] — OpenAI：34 页落地白皮书（判据、编排、护栏）
- [[sources/06-effective-context-engineering]] — Anthropic：上下文工程方法论与长程三术
- [[sources/07-multi-agent-research-system]] — Anthropic：多 agent 生产复盘与关键数字
- [[sources/08-writing-effective-tools-for-agents]] — Anthropic：工具设计与评估驱动改进
- [[sources/09-code-execution-with-mcp]] — Anthropic：Code Mode 与渐进披露
- [[sources/10-mcp-architecture-overview]] — MCP 官方架构规范（2026-07-28 版变化）
- [[sources/11-openai-agents-sdk-docs]] — Agents SDK 七章核心文档
- [[sources/12-langgraph-workflows-and-agents]] — 五种模式的图实现与 LangGraph 生态分层
- [[sources/13-a2a-protocol-overview]] — A2A 协议要点与 MCP 分工
- [[sources/14-owasp-genai-llm-top-10-2026]] — OWASP 2026 榜单与三条 agent 相关条目
- [[sources/15-foundational-agent-papers]] — 28 篇基础论文速查表
- [[sources/16-lethal-trifecta]] — 致命三要素与术语辨析

## Analysis

- [[analysis/01-interview-roadmap]] — 面试考什么 + 14 天 / 7 天计划 + 自测产出
- [[analysis/02-interview-qa-bank]] — 七大板块 47 道高频题与答题要点
- [[analysis/03-framework-comparison]] — 声明式图 vs code-first，选型三判据
- [[analysis/04-single-vs-multi-agent]] — 决策框架与流程图
- [[analysis/05-glossary]] — 术语中英对照速查 + 关键数字表
- [[analysis/06-source-map]] — 已入库资料清单与扩展阅读
