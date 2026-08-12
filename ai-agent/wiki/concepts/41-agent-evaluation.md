---
title: Agent 评估（Evaluation）
date: 2026-08-11
tags: [evaluation, llm-as-judge, metrics, production]
sources: [How-We-Built-Our-Multi-Agent-Research-System.md, Writing-Effective-Tools-for-AI-Agents.md, OpenAI-A-Practical-Guide-to-Building-Agents.md]
---

# Agent 评估（Evaluation）

面试里区分"做过 demo"和"做过产品"的关键问题。核心难点：**agent 是非确定性的，同样起点会走完全不同的有效路径**，所以不能像传统测试那样断言"输入 X → 路径 Y → 输出 Z"。

## 五条方法论

### 1. 立刻开始，小样本就够

早期改动效应量极大（一个 prompt 改动能把成功率从 30% 拉到 80%），**约 20 条代表真实使用模式的查询**就能看出变化。

> 很多团队因为"以为必须几百条才有用"而迟迟不做 eval —— 这是错的，先用几个例子做起来。

### 2. LLM-as-judge，做对了能规模化

Anthropic 的具体配置（可直接复用）：

- **单次 LLM 调用、单个 prompt**，输出 **0.0–1.0 分 + pass/fail**
- 他们试过"多个裁判各评一个维度"，发现**单裁判单 prompt 反而最一致、最贴合人类判断**
- rubric 五维：**事实准确性**（论断是否匹配来源）、**引用准确性**（引用是否对应论断）、**完整性**（是否覆盖所有要求）、**来源质量**（是否用一手来源）、**工具效率**（工具用得对不对、次数是否合理）
- 有明确答案时最好用（"是否准确列出了研发预算前三的药企"）

**边界**：ChemCrow 的反例 —— 在需要深度专业知识的领域，LLM 评估会与专家判断严重不一致（LLM 认为 GPT-4 与 ChemCrow 相当，化学专家认为 ChemCrow 大幅胜出）。**缺乏专业知识时它不知道自己不知道**。

### 3. 人工评估捕捉自动化遗漏的东西

人类测试者发现的典型问题：异常 query 上的幻觉、系统性故障、**微妙的来源选择偏差**（早期 agent 一贯偏好 SEO 内容农场而非权威但排名低的学术 PDF）。加了来源质量启发式才解决。**即使在自动化评估的时代，手工测试仍不可替代。**

### 4. End-state evaluation（会改状态的 agent）

对会修改持久状态的多轮 agent：**评最终状态是否正确，而不是逐轮流程**。这承认 agent 可能用不同路径达到同一目标。复杂工作流拆成若干 **checkpoint**，检查特定状态变化是否发生，而不是校验每个中间步骤。

### 5. 工具评估（面向工具而非 agent）

来自 [[sources/08-writing-effective-tools-for-agents]]：

- 任务要基于真实数据源、避免玩具沙盒；强任务需要**多次甚至数十次**工具调用
- 好任务 vs 坏任务：坏任务把解法写在题目里（"搜索 `purchase_complete` 且 `customer_id=9182`"），好任务只给业务目标（"客户报告被扣款三次，找出相关日志并判断是否影响其他客户"）
- **验证器别太严**：格式、标点、等价表述差异不该判错
- 可以指定"期望调用的工具"，但**别过度指定路径**（存在多条正确解法）
- 用直接 API 调用 + 简单 while 循环跑；在 system prompt 里要求输出 reasoning/feedback 块（触发 CoT，也便于诊断）
- 用 **held-out 测试集**防过拟合

## 该收集哪些指标

| 层次 | 指标 |
|---|---|
| 结果 | 任务成功率、pass/fail、rubric 分数、`pass^k`（多次运行的可靠性，τ-bench 提出） |
| 过程 | 工具调用次数、工具错误率、参数无效率、轮次数、是否触发 max_turns |
| 成本 | 总 token、每任务成本、缓存命中率 |
| 延迟 | 端到端时长、单次工具调用时长、首 token 延迟 |
| 安全 | 护栏触发率、误报/漏报、审批升级率 |

**指标反过来指导设计**：大量冗余工具调用 → 该调分页/token 上限默认值；大量参数无效错误 → 工具描述或示例不足。

## 建流程的顺序（OpenAI 的模型选择方法论）

1. **先建 eval 拿基线**
2. 用**最强模型**达到准确率目标
3. 再用小模型替换以优化成本与延迟

—— 先确定能力上限，再压成本，避免过早限制 agent 能力。

## 与其他概念的关系

- [[concepts/42-agent-benchmarks]]：公开基准是外部参照，内部 eval 才决定产品质量
- [[concepts/24-reflection]]：同样用 LLM 评判，但目的是改进而非度量
- [[concepts/43-durable-execution]]：tracing 是 eval 的数据来源

## 来源

- [[sources/07-multi-agent-research-system]]、[[sources/08-writing-effective-tools-for-agents]]、[[sources/05-openai-practical-guide-to-building-agents]]、[[sources/04-llm-powered-autonomous-agents]]
