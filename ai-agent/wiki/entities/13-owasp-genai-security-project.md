---
title: OWASP GenAI Security Project
date: 2026-08-11
tags: [organization, security, owasp, entity]
sources: [OWASP-GenAI-LLM-Top-10-2026.md]
---

# OWASP GenAI Security Project

OWASP 下的生成式 AI 安全项目，为 GenAI 系统与应用的安全提供免费指南与资源（`genai.owasp.org`）。

## 主要产出

- **OWASP GenAI LLM Top 10**（当前版本 **2026**，2026-08-04 发布）—— 面向"模型作为应用组件"的风险清单，见 [[sources/14-owasp-genai-llm-top-10-2026]]
- **OWASP Agentic Top 10（ASI 系列）** —— 当模型成为**行动者**（可调工具、跨会话带记忆、在下游造成后果）时的风险清单。LLM 榜单中引用到的条目：ASI02 Tool Misuse & Exploitation、ASI03 Identity & Privilege Abuse、ASI04 Agentic Supply Chain Vulnerabilities、ASI08 Cascading Failures
- AISVS 映射、相关框架映射（NIST、CISA 等）、参考文献集

## 项目负责人

Steve Wilson（Project Lead）、Rock Lambros（Co-Lead）。

## 仓库变迁（引用时注意）

旧仓库 `OWASP/www-project-top-10-for-large-language-model-applications` 已成为历史归档入口，活跃开发迁至 **`GenAI-Security-Project/GenAI-LLM-Top10`**（2026 版正文在 `2026/final/`）。许可：CC BY-SA 4.0。

## 2026 版方法论上的变化

榜单同时依据**从业者投票**与**真实事故记录**，并互相校验 —— 这是此前版本"只能承诺"的部分。相应地，**Excessive Agency 升到第三**、Unbounded Consumption 上升 4 位、Improper Output Handling 从第 5 跌到第 10、System Prompt Leakage 改名扩围为 **Hidden Context Exposure**。

相关概念页：[[concepts/39-prompt-injection]]、[[concepts/40-excessive-agency]]
