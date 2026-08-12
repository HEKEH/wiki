---
title: 术语速查（中英对照）
date: 2026-08-11
tags: [analysis, glossary, cheatsheet, interview]
sources: [Building-Effective-AI-Agents.md, Effective-Context-Engineering-for-AI-Agents.md, MCP-Architecture-Overview.md, OWASP-GenAI-LLM-Top-10-2026.md]
---

# 术语速查（中英对照）

面试前 30 分钟扫一遍用。每条给"一句话定义 + 去哪深挖"。

## 架构与形态

| 术语 | 一句话 | 详见 |
|---|---|---|
| **Agentic system** | Workflows 与 Agents 的统称 | [[concepts/13-agentic-systems]] |
| **Workflow** | LLM 与工具沿**预定义代码路径**编排 | [[concepts/13-agentic-systems]] |
| **Agent** | LLM 在循环中**自主使用工具**，自己控制流程 | [[concepts/21-agent-loop]] |
| **Augmented LLM** | LLM + 检索 + 工具 + 记忆，一切 agent 的基础构件 | [[concepts/14-augmented-llm]] |
| **Harness** | 承载 agent 循环的工程实现（调模型、路由工具、管状态） | [[concepts/02-harness]] |
| **Meta-harness** | 不绑定具体 harness 实现的系统设计 | [[concepts/07-meta-harness]] |
| **Sandbox** | agent 的"双手"：代码执行与文件编辑环境；凭证永不进入 | [[concepts/03-sandbox]] |
| **Session** | agent 运行的追加写事件日志，可持久化 | [[concepts/01-session]] |
| **Run / agent loop** | 一次运行：循环到退出条件（最终输出/结构化输出/错误/max_turns） | [[concepts/21-agent-loop]] |
| **Skills** | 可复用的指令+脚本+资源文件夹（配 `SKILL.md`） | [[concepts/32-progressive-disclosure-and-code-execution]] |

## 编排模式

| 术语 | 一句话 | 详见 |
|---|---|---|
| **Prompt chaining** | 串行分解，可加 gate 检查 | [[concepts/15-prompt-chaining]] |
| **Routing** | 分类后路由到专门处理 | [[concepts/16-routing]] |
| **Parallelization** | 分片并行（sectioning）或多次投票（voting）；分支编译期已知 | [[concepts/17-parallelization]] |
| **Orchestrator-workers** | 中心 LLM **运行时**决定子任务数量与内容 | [[concepts/18-orchestrator-workers]] |
| **Evaluator-optimizer** | 生成 ↔ 评估成环迭代 | [[concepts/19-evaluator-optimizer]] |
| **Manager pattern / agents as tools** | 通过 tool call 调专家，控制权不转移 | [[concepts/35-manager-vs-decentralized]] |
| **Decentralized / handoff** | 单向移交执行权与会话状态 | [[concepts/36-handoff]] |
| **Lead agent / subagent** | 主 agent 持计划，子 agent 用干净 context 深挖后蒸馏回传 | [[concepts/34-orchestrator-worker-multi-agent]] |

## 推理与规划

| 术语 | 一句话 | 详见 |
|---|---|---|
| **CoT** | "think step by step"，用测试时计算换准确率 | [[concepts/22-planning-and-decomposition]] |
| **Self-Consistency** | 采样多条路径取多数答案 | 同上 |
| **ToT** | 每步多候选构成树 + 评估器 + BFS/DFS 可回溯 | 同上 |
| **PAL** | 推理写成程序，计算交给解释器 | 同上 |
| **ReAct** | Thought → Action → Observation 交错循环 | [[concepts/23-react]] |
| **ReWOO** | 推理与观察解耦：先规划全部调用再执行，省 token | 同上 |
| **Reflexion** | 语言化反思写入工作记忆再重试（用语言做 RL） | [[concepts/24-reflection]] |
| **Self-Refine** | 同一模型 生成→自评→修订 | 同上 |
| **Extended / interleaved thinking** | 模型可见的思考过程；后者在工具结果之后继续思考 | [[concepts/23-react]] |

## 上下文与记忆

| 术语 | 一句话 | 详见 |
|---|---|---|
| **Context engineering** | 每轮推理都要重做的 token 状态策展 | [[concepts/09-context-engineering]] |
| **Context rot** | token 越多，准确召回能力越差 | [[concepts/29-context-rot-and-attention-budget]] |
| **Attention budget** | 模型的有限注意力预算，每个 token 都在消耗 | 同上 |
| **Right altitude** | system prompt 在"硬编码脆弱"与"空泛无信号"之间的最优海拔 | [[concepts/09-context-engineering]] |
| **Compaction** | 总结历史后用摘要重启新 context | [[concepts/30-compaction-and-note-taking]] |
| **Structured note-taking / agentic memory** | agent 把笔记写到 context 之外再读回 | 同上 |
| **Just-in-time context** | 只存轻量标识符，运行时用工具取 | [[concepts/28-agentic-search]] |
| **Progressive disclosure** | 逐层探索发现上下文/工具，而非一次全加载 | [[concepts/32-progressive-disclosure-and-code-execution]] |
| **Context anxiety** | 模型感知窗口将满而提前收工 | [[concepts/06-context-anxiety]] |
| **MIPS / ANN** | 最大内积搜索 / 近似最近邻（LSH、ANNOY、HNSW、FAISS、ScaNN） | [[concepts/26-memory]] |
| **GraphRAG** | 实体图 + 社区摘要，回答全局性问题 | [[concepts/27-rag]] |
| **Rerank** | 用 cross-encoder 对 top-k 精排，性价比最高的单点改进 | 同上 |
| **Hybrid retrieval** | 向量 + 关键词（BM25）融合（如 RRF） | 同上 |

## 工具与协议

| 术语 | 一句话 | 详见 |
|---|---|---|
| **ACI** | Agent-Computer Interface：为 agent 设计工具接口 | [[concepts/20-aci]] |
| **Poka-yoke** | 防呆设计：改设计让错误不可能发生 | 同上 |
| **Function calling** | 模型输出结构化工具调用请求（应用负责执行） | [[concepts/25-tool-use-and-function-calling]] |
| **MCP** | 连接 agent 与外部系统的开放协议 | [[concepts/31-mcp]] |
| **MCP 三原语** | Tools（做动作）/ Resources（读数据）/ Prompts（模板） | 同上 |
| **Elicitation** | client 侧原语：server 向用户索取信息或确认 | 同上 |
| **STDIO / Streamable HTTP** | MCP 两种传输：本地进程 / 远程 HTTP(+SSE) | 同上 |
| **Code Mode / code execution with MCP** | 把 server 当代码 API，agent 写代码调用（-98.7% token） | [[concepts/32-progressive-disclosure-and-code-execution]] |
| **A2A** | agent↔agent 互操作协议，用 Agent Card 发现，对端不透明 | [[concepts/33-a2a]] |
| **Tool annotations** | MCP 中声明破坏性操作/开放世界访问 | [[concepts/31-mcp]] |

## 评估与工程化

| 术语 | 一句话 | 详见 |
|---|---|---|
| **LLM-as-judge** | 用 LLM 按 rubric 打分（单调用单 prompt 最一致） | [[concepts/41-agent-evaluation]] |
| **End-state evaluation** | 评最终状态而非逐轮流程 | 同上 |
| **`pass^k`** | 同一任务多次运行都成功的比例（可靠性指标，τ-bench 提出） | [[concepts/42-agent-benchmarks]] |
| **Held-out test set** | 防止把工具/prompt 过拟合到训练 eval | [[concepts/41-agent-evaluation]] |
| **Durable execution** | 可持久、可恢复的执行（checkpoint + resume） | [[concepts/43-durable-execution]] |
| **Rainbow deployment** | 新旧版本并存渐进切流，不打断在跑的 agent | 同上 |
| **Prompt caching** | 缓存稳定前缀降本 | [[concepts/45-token-economics]] |
| **Effort scaling** | 按查询复杂度分配 agent 数与调用次数 | 同上 |
| **Pets vs cattle** | 不可丢失的状态 vs 可替换的执行体 | [[concepts/05-pets-vs-cattle]] |

## 安全

| 术语 | 一句话 | 详见 |
|---|---|---|
| **Prompt injection** | 可信与不可信内容混在同一 context 导致行为被改（≠ jailbreaking） | [[concepts/39-prompt-injection]] |
| **Direct / indirect injection** | 用户输入注入 / 经检索内容、工具输出、记忆注入 | 同上 |
| **Lethal trifecta** | 私有数据 + 不可信内容 + 对外通信，三者齐备即高危 | [[sources/16-lethal-trifecta]] |
| **Rule of Two** | A 不可信输入 / B 敏感数据 / C 状态变更或对外通信，三者齐备须逐动作审批 | [[concepts/39-prompt-injection]] |
| **Excessive Agency** | 功能过多 / 权限过大 / 自治过度（OWASP 2026 第三位） | [[concepts/40-excessive-agency]] |
| **Complete mediation** | 授权在代码里判定，分级执行 audit→warn→block→escalate | 同上 |
| **Guardrail / tripwire** | 并发校验层 / 违规抛出的异常 | [[concepts/37-guardrails]] |
| **HIL（human-in-the-loop）** | 失败阈值或高风险动作时的人工介入 | [[concepts/38-human-in-the-loop]] |
| **Hidden Context Exposure** | OWASP LLM08，原 System Prompt Leakage 的扩围版 | [[sources/14-owasp-genai-llm-top-10-2026]] |
| **Unbounded Consumption** | OWASP LLM06，资源/成本耗尽 | 同上 |

## 关键数字（背下来）

| 数字 | 含义 |
|---|---|
| **+90.2%** | 多 agent（Opus 4 lead + Sonnet 4 sub）相对单 agent 的 research eval 提升 |
| **80% / 95%** | token 用量单独解释的性能方差 / 加上工具调用次数与模型选择后 |
| **4× / 15×** | agent / 多 agent 相对 chat 的 token 消耗 |
| **-98.7%（150k→2k）** | 代码执行替代直接 tool call 的 token 节省 |
| **25,000 token** | Claude Code 默认单次工具响应上限 |
| **约 20 条** | 起步 eval 集规模 |
| **1k–2k token** | 子 agent 回传摘要的目标体量 |
| **40%** | tool-testing agent 改进工具描述后任务完成时间下降 |
| **90%** | 并行化后复杂研究任务时间下降上限 |
| **92% vs 15%** | GAIA 上人类 vs 带插件的 GPT-4 |
| **约 5 篇文档 → 约 90%** | RAG 语料投毒的攻击成功率 |
