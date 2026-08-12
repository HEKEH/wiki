---
title: Simon Willison
date: 2026-08-11
tags: [person, security, author, entity]
sources: [The-Lethal-Trifecta-for-AI-Agents.md, Effective-Context-Engineering-for-AI-Agents.md]
---

# Simon Willison

开发者、作家（Django 联合创始人之一），个人博客 `simonwillison.net` 是 LLM 应用安全领域被引用最多的独立来源之一。

## 两个进入行业词汇表的贡献

1. **"Prompt injection" 一词的提出者**（2022-09，仿照 SQL injection 命名），并坚持区分它与 jailbreaking：前者是"可信与不可信内容混在同一 context"的应用架构问题，后者是模型安全问题
2. **Lethal Trifecta（致命三要素）**（2025-06）：访问私有数据 + 暴露于不可信内容 + 对外通信能力，三者同时具备即高危 —— 已被 **OWASP GenAI LLM Top 10 (2026)** 在 LLM01 缓解章节正式引用。见 [[sources/16-lethal-trifecta]]

## 另一处被引用

Anthropic 在 [[sources/06-effective-context-engineering]] 中采用了他对 agent 的简洁定义：

> **LLMs autonomously using tools in a loop.**

见 [[concepts/21-agent-loop]]。

## 他持续追踪的主题

exfiltration 攻击案例集（M365 Copilot、GitHub MCP server、GitLab Duo、Slack AI、ChatGPT Operator 等生产系统的注入漏洞），以及对"guardrail 产品能拦 95% 攻击"这类宣称的批评（"在 Web 安全里 95% 是不及格分"）。相关：[[concepts/37-guardrails]]、[[concepts/39-prompt-injection]]
