---
tags: [agent, tool, design, interface]
date: 2026-04-25
sources:
  - sources/Building-Effective-AI-Agents.md
---

# Agent-Computer Interface (ACI)

类比 Human-Computer Interface (HCI) 的概念：**为 Agent 设计工具接口**应获得与为人设计界面同等的投入。

## 核心思想

工具定义和规范应获得与 prompt 工程同等的重视。工具是 Agent 感知和作用于世界的唯一渠道——接口设计直接决定 Agent 的能力上限。

## 设计原则

### 格式选择

- 给模型足够 token 在写代码前"思考"，避免把自己写入死角
- 格式接近模型在互联网上自然见到的文本（如 Markdown 内的代码 > JSON 内的代码）
- 避免格式开销（如 diff chunk header 需预知行数、JSON 内需额外转义）

### 设计建议

1. **换位思考**：站在模型角度看工具描述和参数——是否一目了然？好的工具定义应包含示例用法、边界情况、输入格式要求和与其他工具的清晰边界
2. **优化参数命名和描述**：像给初级开发者写优秀 docstring 一样，尤其在使用多个相似工具时
3. **测试迭代**：在 Workbench 中用多组输入测试，观察模型犯错模式并迭代
4. **Poka-yoke（防呆）**：改变参数设计使错误难以发生

## Poka-yoke 实例

SWE-bench Agent 开发中发现：Agent 离开根目录后使用相对路径会出错。解决方案：**强制要求绝对路径**——模型立刻无错使用。

这是经典的 Poka-yoke：不是训练模型"记住用对路径"，而是**让错误不可能发生**。

## 与其他概念的关系

- [The Augmented LLM](augmented-llm.md) 中"工具"增强的接口设计就是 ACI
- [Context Engineering](context-engineering.md) 管理 ACI 中工具描述的上下文内容
- [Harness](harness.md) 的工具路由逻辑需要遵循 ACI 设计原则

## 来源

- [Building Effective AI Agents](../sources/Building-Effective-AI-Agents.md) — Anthropic 工程博客
