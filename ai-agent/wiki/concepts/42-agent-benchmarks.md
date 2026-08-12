---
title: Agent 评测基准（Benchmarks）
date: 2026-08-11
tags: [benchmarks, evaluation, swe-bench, tau-bench, gaia]
sources: [Foundational-Agent-Papers-Abstracts.md, How-We-Built-Our-Multi-Agent-Research-System.md]
---

# Agent 评测基准

被问"你怎么衡量 agent 能力"时，除了自建 eval，还应能说清主流公开基准**各测什么、为什么难**。

## 主要基准

| 基准 | 测什么 | 关键设计 / 数字 |
|---|---|---|
| **SWE-bench**（2310.06770） | 真实 GitHub issue → 生成修复补丁 | 需跨文件理解仓库、执行测试验证；Verified 子集是业界公认的编码 agent 标尺 |
| **τ-bench**（2406.12045） | 工具–agent–**用户**三方交互 | 用 LLM 模拟用户做动态对话；比较**会话结束时的数据库状态**与标注目标状态；提出 **`pass^k`** 衡量多次运行的一致性。SOTA function-calling agent（gpt-4o）成功率 <50%，零售域 **pass^8 <25%** |
| **GAIA**（2311.12983） | 真实助手任务（推理 + 多模态 + 浏览 + 工具） | 对人简单对 AI 难：**人类 92% vs 带插件的 GPT-4 15%** |
| **AgentBench**（2308.03688） | 8 个不同环境下把 LLM 当 agent 评 | 多轮开放式生成，首个系统性多环境 agent 基准 |
| **WebArena**（2307.13854） | 端到端网页任务 | **可复现的真实网站环境**（自建站点而非真实互联网） |
| **BrowseComp** | 浏览 agent 定位"难找信息"的能力 | Anthropic 用它做归因分析：token 用量单独解释 80% 性能方差 |
| **API-Bank** | 工具使用三级能力 | Level-1 会调用 / Level-2 会检索工具 / Level-3 会规划多次调用 |
| **AgentDojo / JailbreakBench** | 提示注入与越狱的攻防基线 | OWASP 建议先用它们打基线，再用**自适应攻击**红队 |

## 怎么读这些数字（面试加分点）

1. **区分"能力上限"与"可靠性"**：τ-bench 的 `pass^k` 正是为此设计 —— 一次能做对不代表八次都能做对。生产上重要的是后者
2. **警惕污染与过拟合**：公开基准的任务可能进了训练数据；厂商报数常带特定 scaffold（不同 harness 差异巨大）
3. **基准 ≠ 你的业务**：Anthropic 的做法是**自建约 20 条真实查询**起步（[[concepts/41-agent-evaluation]]）；公开基准用于选模型与对外沟通
4. **状态型任务用 end-state 评估**：τ-bench 比对最终数据库状态就是这一思想的基准化实现

## 与其他概念的关系

- [[concepts/41-agent-evaluation]]：内部 eval 方法论
- [[concepts/45-token-economics]]：BrowseComp 的归因结论直接支持"多花 token 换性能"的取舍分析
- [[concepts/39-prompt-injection]]：安全类基准的用法与局限

## 来源

- [[sources/15-foundational-agent-papers]]、[[sources/07-multi-agent-research-system]]、[[sources/14-owasp-genai-llm-top-10-2026]]
