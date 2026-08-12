---
title: 权威资料清单与扩展阅读
date: 2026-08-11
tags: [analysis, sources, reading-list, github]
sources: [Foundational-Agent-Papers-Abstracts.md]
---

# 权威资料清单与扩展阅读

本页记录**已入库**的一手资料（`raw/` 有存档，可离线读）与**推荐扩展**（尚未入库）。抓取日期：2026-08-11。

## 已入库：厂商工程材料（最高优先级）

| 资料 | 来源 | 存档 | wiki 摘要 |
|---|---|---|---|
| Building Effective AI Agents | Anthropic | `raw/Building-Effective-AI-Agents.md` | [[sources/03-building-effective-ai-agents]] |
| A Practical Guide to Building Agents | OpenAI（34 页 PDF） | `raw/OpenAI-A-Practical-Guide-to-Building-Agents.pdf` + `.md` | [[sources/05-openai-practical-guide-to-building-agents]] |
| Effective Context Engineering for AI Agents | Anthropic | `raw/Effective-Context-Engineering-for-AI-Agents.md` | [[sources/06-effective-context-engineering]] |
| How We Built Our Multi-Agent Research System | Anthropic | `raw/How-We-Built-Our-Multi-Agent-Research-System.md` | [[sources/07-multi-agent-research-system]] |
| Writing Effective Tools for AI Agents | Anthropic | `raw/Writing-Effective-Tools-for-AI-Agents.md` | [[sources/08-writing-effective-tools-for-agents]] |
| Code Execution with MCP | Anthropic | `raw/Code-Execution-with-MCP.md` | [[sources/09-code-execution-with-mcp]] |
| Effective Harnesses for Long-Running Agents | Anthropic | `raw/Effective-harnesses-for-long.md` | [[sources/02-effective-harnesses-for-long-running-agents]] |
| Scaling Managed Agents: Decoupling Brain/Hands/Session | Anthropic | `raw/Scaling-Managed-Agents-Decoupling.md` | [[sources/01-scaling-managed-agents-decoupling]] |

## 已入库：综述与协议规范

| 资料 | 来源 | wiki 摘要 |
|---|---|---|
| LLM Powered Autonomous Agents | Lilian Weng（Lil'Log，含 13 张配图存档） | [[sources/04-llm-powered-autonomous-agents]] |
| MCP Architecture Overview（spec `2026-07-28`） | modelcontextprotocol.io | [[sources/10-mcp-architecture-overview]] |
| OpenAI Agents SDK 核心文档（7 章合辑） | openai/openai-agents-python | [[sources/11-openai-agents-sdk-docs]] |
| LangGraph: Workflows and Agents + Overview | docs.langchain.com | [[sources/12-langgraph-workflows-and-agents]] |
| A2A Protocol README | a2aproject/A2A（Linux Foundation） | [[sources/13-a2a-protocol-overview]] |
| OWASP GenAI LLM Top 10（2026 全文） | GenAI-Security-Project/GenAI-LLM-Top10 | [[sources/14-owasp-genai-llm-top-10-2026]] |
| The Lethal Trifecta for AI Agents | Simon Willison | [[sources/16-lethal-trifecta]] |
| 28 篇基础论文 arXiv 摘要汇编 | arxiv.org | [[sources/15-foundational-agent-papers]] |

## 已入库论文清单（按主题）

推理规划：CoT 2201.11903 · Self-Consistency 2203.11171 · ToT 2305.10601 · Plan-and-Solve 2305.04091 · PAL 2211.10435
工具行动：ReAct 2210.03629 · Toolformer 2302.04761 · MRKL 2205.00445 · ReWOO 2305.18323 · LLM Compiler 2312.04511
反思：Reflexion 2303.11366 · Self-Refine 2303.17651
记忆检索：RAG 2005.11401 · MemGPT 2310.08560 · RAG Survey 2312.10997 · GraphRAG 2404.16130 · Memory Survey 2404.13501
多智能体：Generative Agents 2304.03442 · Voyager 2305.16291 · AutoGen 2308.08155
综述：2308.11432 · 2309.07864
基准：SWE-bench 2310.06770 · τ-bench 2406.12045 · GAIA 2311.12983 · AgentBench 2308.03688 · WebArena 2307.13854
安全：Indirect Prompt Injection 2302.12173

**冲刺只精读 5 篇**：ReAct → Reflexion → RAG → MemGPT → SWE-bench。

## 推荐扩展（尚未入库，需要时再抓）

### 官方文档与仓库

| 资源 | 地址 | 价值 |
|---|---|---|
| Anthropic Cookbook | `github.com/anthropics/anthropic-cookbook` | 五种 workflow 模式的可运行 notebook；tool evaluation cookbook |
| Claude 官方文档：Agent SDK / Agent Skills / tool use | `platform.claude.com/docs` | 一手 API 与最佳实践（含 `llms.txt`） |
| MCP servers 参考实现 | `github.com/modelcontextprotocol/servers` | 读几个 server 源码是理解 MCP 最快的方式 |
| MCP 完整规范 | `modelcontextprotocol.io/specification/latest` | 面试深挖协议细节时用 |
| Hugging Face Agents Course | `github.com/huggingface/agents-course` | 免费系统课程（unit1–4 + bonus），含 smolagents 实践 |
| OpenAI Agents SDK 仓库 | `github.com/openai/openai-agents-python` | 读 `Runner` 实现能彻底搞懂 agent loop |
| LangGraph 文档 | `docs.langchain.com/oss/python/langgraph` | persistence / interrupt / subgraph 等生产能力 |
| OWASP Agentic Top 10（ASI 系列） | `genai.owasp.org` | agent 作为"行动者"时的风险清单（本库仅通过 LLM 榜单间接覆盖） |
| A2A 规范与教程 | `a2a-protocol.org` | Agent Card、任务生命周期细节 |

### 值得跟的独立来源

- **Simon Willison's Weblog**（`simonwillison.net`）：注入与外传攻击案例的持续追踪
- **Anthropic Engineering 博客**：本库主干来源，新文值得第一时间读
- **Chroma 的 context rot 研究**（`research.trychroma.com/context-rot`）：长上下文退化的实证依据
- **Cloudflare "Code Mode"**（`blog.cloudflare.com/code-mode`）：与 Anthropic 代码执行结论互相印证
- **CaMeL（Google DeepMind）与 "Design Patterns for Securing LLM Agents against Prompt Injections"**：注入缓解的两个正经方向

### 面向面试的实践建议

比读更重要的是**动手做一次**，建议最小项目：

1. 用裸 API 写一个 20 行 agent 循环（工具 + `max_turns` + 结构化输出）
2. 给它 3 个工具，其中一个刻意设计得糟糕，观察失败模式
3. 写 20 条 eval + LLM-as-judge rubric，量化改工具描述前后的差异
4. 加一层输入护栏与一个高风险动作审批
5. 记录 token/延迟/成功率的前后对比 —— 这套数字就是面试里最有说服力的材料

## 抓取方法备注（便于后续增量）

- GitHub 上的 markdown/mdx 直接用 `raw.githubusercontent.com` 取；仓库结构用 `api.github.com/repos/<owner>/<repo>/contents/<path>` 探
- Mintlify 文档站（`docs.langchain.com`、`modelcontextprotocol.io`）在 URL 后加 `.md` 即得 markdown 原文
- 普通 HTML 文章用 bs4 转 markdown 后归档（脚本见 session scratchpad 的 `h2m.py` 思路：article/main 提取 + 标签映射）
- arXiv 摘要抓 `arxiv.org/abs/<id>` 页面的 title/authors/dateline/abstract
