---
title: 反思与自我修正（Reflection & Self-Critique）
date: 2026-08-11
tags: [reflection, reflexion, self-refine, evaluation, papers]
sources: [Foundational-Agent-Papers-Abstracts.md, LLM-Powered-Autonomous-Agents.md]
---

# 反思与自我修正

让 agent 从自己的失败中学习，而**不更新权重**。这是 agent 与普通 LLM 调用最本质的差异之一：允许试错。

## 三条主线

### Reflexion（2303.11366）

- 标准 RL 设置但奖励是**二元**的；动作空间沿用 ReAct（任务动作 + 语言）
- 每次动作后计算**启发式函数 h**，据反思结果决定是否**重置环境重开一轮**
- 启发式判定两种失败：
  - **低效规划**：轨迹太长仍未成功
  - **幻觉**：连续相同动作导致相同观察（很实用的死循环检测规则）
- 反思由两个 few-shot 示例引导生成（每例 = 失败轨迹 + 理想反思），反思结果写入**工作记忆，最多保留 3 条**作为后续查询的上下文
- 实验（ALFWorld / HotpotQA）：ALFWorld 中幻觉比低效规划更常见
- 一句话总结：**用语言而非梯度做强化学习**

### Self-Refine（2303.17651）

同一个 LLM 扮演三个角色：生成 → 自评反馈 → 依反馈修订，迭代进行，**无需额外训练数据或奖励模型**。这就是 [[concepts/19-evaluator-optimizer]] 的单模型版本。

### Chain of Hindsight / Algorithm Distillation

- **CoH**：把"带人类反馈标注的历史输出序列"按奖励排序后放进 context，微调模型只预测最后（最好）那个 —— 学会顺着改进趋势走
- **AD**：把跨 episode 的 RL 学习历史拼接进 context，学的是**RL 算法本身**而非某个任务策略

## 何时有效、何时无效（面试关键）

**有效的前提是有可靠的反馈信号**：
- 有：编译错误、测试失败、类型检查、lint、工具返回的显式错误、可验证的数值结果 → 反思效果好，这也是编码 agent 表现突出的原因
- 弱：自由文本质量、需要深度专业知识的判断 → 容易自我欺骗

**反例（必须记住）**：ChemCrow 中 LLM 评估认为 GPT-4 与 ChemCrow 相当，化学专家评估却认为 ChemCrow 大幅胜出 —— **在需要深度专业知识的领域，LLM 不适合评判自己**（[[concepts/41-agent-evaluation]] 里 LLM-as-judge 的边界）。

## 工程化的三种落地形态

1. **循环内自检**：工具返回错误 → agent 读错误信息自行调整。Anthropic 的经验："告诉 agent 工具失败了，让它自己适应，效果出乎意料地好"
2. **显式 evaluator 节点**：generator ↔ evaluator 成环，evaluator 给结构化 grade + feedback（LangGraph 的 evaluator-optimizer 实现）
3. **跨 session 反思**：把失败教训写进持久文件（progress file / `CLAUDE.md` / 记忆），下个 session 读回 —— [[concepts/10-long-running-agent]]

## 与其他概念的关系

- [[concepts/19-evaluator-optimizer]]：把反思固化成 workflow 的两节点回路
- [[concepts/23-react]]：Reflexion 建立在 ReAct 的动作空间上
- [[concepts/41-agent-evaluation]]：自我评估与外部评估共用 LLM-as-judge 技术，但目的不同（改进 vs 度量）
- [[concepts/44-generative-agents]] 里的 "reflection" 是另一个意思：把琐碎观察**综合成更高层洞见**，不是纠错

## 来源

- [[sources/15-foundational-agent-papers]]、[[sources/04-llm-powered-autonomous-agents]]、[[sources/07-multi-agent-research-system]]
