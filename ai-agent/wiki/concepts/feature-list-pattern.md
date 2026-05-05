# Feature List Pattern

tags: [agent-architecture, harness-design, testing]
date: 2026-04-24
sources: [../../raw/Effective-harnesses-for-long.md]
status: active

## Definition

一种结构化的需求拆解和追踪模式：将用户的高层 prompt 展开为细粒度的 feature 列表，每个 feature 包含描述、验证步骤和通过状态，作为 [Long-Running Agent](long-running-agent.md) 的"客观完成标准"。

## 核心设计

### 格式选择：JSON 而非 Markdown

Anthropic 实验发现，**用 JSON 而非 Markdown 编写 feature list**，因为模型更容易擅自修改或覆盖 Markdown 文件，而 JSON 结构对模型的不当修改更具抵抗力。

### Feature 结构

```json
{
  "category": "functional",
  "description": "New chat button creates a fresh conversation",
  "steps": [
    "Navigate to main interface",
    "Click the 'New Chat' button",
    "Verify a new conversation is created",
    "Check that chat area shows welcome state",
    "Verify conversation appears in sidebar"
  ],
  "passes": false
}
```

关键字段：
- **category** — 分类（functional, UI, error-handling 等）
- **description** — 人类可读的 feature 描述
- **steps** — 端到端验证步骤，指导 Agent 如何测试
- **passes** — 通过状态，初始全为 false

### 细粒度原则

在 claude.ai clone 示例中，高层 prompt 被展开为 **200+ 个 feature**。例如"用户可以发消息"被拆解为打开新对话、输入查询、按回车、看到 AI 响应等多个独立 feature。

## 三个作用

1. **防止 One-shotting** — Agent 不再试图一次完成所有功能，而是每次选一个 feature
2. **防止 Premature Victory** — 有明确的"全部 passes = 完成"标准，Agent 无法自行降低完成门槛
3. **防止过早标记完成** — 每个 feature 只有端到端测试通过才能标 passes

## 使用约束

- **Coding Agent 只能修改 `passes` 字段**，不能删除或编辑 feature 本身
- 需要 "strongly-worded" 指令：*"It is unacceptable to remove or edit tests because this could lead to missing or buggy functionality."*
- Initializer Agent 创建 feature list，Coding Agent 消费和更新

## 与测试的关系

Feature list 天然引导 Agent 进行端到端测试（而非仅单元测试）。每个 feature 的 `steps` 就是测试用例，使用浏览器自动化工具（如 Puppeteer MCP）像人类用户一样验证。

## 局限

- 编写 200+ feature 本身需要 Initializer Agent 有较强的需求理解能力
- JSON 格式对非结构化任务（如创意写作、探索性研究）的适用性有限
- 依赖模型遵守"只改 passes"的约束，需要 strong prompting

## 相关概念

- [Long-Running Agent](long-running-agent.md) — Feature List 服务的问题域
- [Initializer/Coding Agent 模式](initializer-coding-agent.md) — Initializer 创建 feature list，Coding Agent 消费
- [Harness](harness.md) — 承载此模式的编排循环
- [Context Engineering](context-engineering.md) — feature list 是跨 session 传递上下文的一种方式
