# AI Agent — Index

按类别组织的全部 wiki 页面目录，每次 ingest 更新。共 **29 个内容页**（另有 home/index/log 三个导航页）。序号反映加入顺序。

## Overview

- [[home]] — 顶层综合与导航

## Concepts

- [[concepts/01-session]] — Agent 运行中的追加写入事件日志，持久化存储
- [[concepts/02-harness]] — Agent 的"大脑"，调用 Claude 并路由工具的循环
- [[concepts/03-sandbox]] — Agent 的"双手"，代码执行和文件编辑环境
- [[concepts/04-brain-hands-session]] — 将 Agent 虚拟化为三个可独立替换的组件
- [[concepts/05-pets-vs-cattle]] — 基础设施运维的经典比喻，Pet 不可丢失 vs Cattle 可替换
- [[concepts/06-context-anxiety]] — Claude 感知 context window 将满时提前收工的行为
- [[concepts/07-meta-harness]] — 不限定特定 harness 实现的系统设计
- [[concepts/08-bitter-lesson]] — Sutton：通用方法胜过特定设计
- [[concepts/09-context-engineering]] — 管理 LLM context window 内容的工程实践
- [[concepts/10-long-running-agent]] — 跨多个 context window 持续工作的 Agent
- [[concepts/11-initializer-coding-agent]] — 双 Agent 分工：Initializer 搭建环境，Coding Agent 增量推进
- [[concepts/12-feature-list-pattern]] — 用结构化 JSON 拆解需求并追踪完成状态
- [[concepts/13-agentic-systems]] — Anthropic 对 Agentic Systems 的统一分类：Workflows vs Agents
- [[concepts/14-augmented-llm]] — 一切 Agent 系统的基础构件：LLM + 检索 + 工具 + 记忆
- [[concepts/15-prompt-chaining]] — 串行分解任务的 Workflow 模式
- [[concepts/16-routing]] — 分类输入并路由到专门处理的 Workflow 模式
- [[concepts/17-parallelization]] — 并行分片/投票的 Workflow 模式
- [[concepts/18-orchestrator-workers]] — 中心 LLM 动态拆解委派的 Workflow 模式
- [[concepts/19-evaluator-optimizer]] — 生成-评估循环迭代的 Workflow 模式
- [[concepts/20-aci]] — 为 Agent 设计工具接口，类比 HCI

## Entities

- [[entities/01-anthropic]] — AI 安全公司，Claude 系列模型的开发商
- [[entities/02-managed-agents]] — Anthropic 托管式 Agent 服务，解耦架构
- [[entities/03-claude]] — Anthropic 的大语言模型系列
- [[entities/04-erik-schluntz]] — Anthropic 工程师，Building Effective AI Agents 合著者
- [[entities/05-barry-zhang]] — Anthropic 工程师，Building Effective AI Agents 合著者
- [[entities/06-justin-young]] — Anthropic 工程师，Effective Harnesses for Long-Running Agents 作者

## Sources

- [[sources/01-scaling-managed-agents-decoupling]] — Anthropic 工程博客：解耦 Brain、Hands、Session 的架构设计
- [[sources/02-effective-harnesses-for-long-running-agents]] — Anthropic 工程博客：Initializer/Coding Agent 双模式解决跨 context window 长时运行问题
- [[sources/03-building-effective-ai-agents]] — Anthropic 工程博客：Agent 系统分类框架与五种 Workflow 模式

## Analysis

<!-- 对比分析、综合探索页面 -->
