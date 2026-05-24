---
title: 包注册表体验设计
date: 2026-05-24
tags: [package-registry, ux, supply-chain, developer-experience]
sources: [Features-everyone-should-steal-from-npmx.md]
---

# 包注册表体验设计

包注册表网站（npmjs.com、crates.io、PyPI、Hex 等）的用户体验设计模式，源自 npmx 的创新实践。

## 安全透明度

注册表应主动暴露供应链风险信号：
- 传递性安装大小（实际磁盘占用 vs 单包大小）
- 安装脚本内容（供应链攻击常见入口点）
- 依赖树漏洞与过时状态（递归标注，对接 OSV）

跨生态参考：bundlephobia/packagephobia（JS）、deps.dev（跨生态）

## 开发者决策支持

帮助开发者快速评估"这个包适合我吗"：
- 语义版本范围解析（`^1.0.0` → 当前具体版本）
- 模块格式/类型系统/引擎兼容性徽章
- 轻量替代建议（附原生 API 链接）
- 跨注册表存在检查（兼作依赖混淆检测）

## 可视化与对比

- 并排包对比（指标表 + 散点图）
- 版本 diff（浏览器内逐文件对比）
- 发布时间线（标注大小异常跳升点）
- 下载分布（按版本线分解，展示生态迁移进度）

## 平台基础能力

- 命令面板（`⌘K`，全局导航 + 上下文操作）
- 国际化（30+ 语言含 RTL）
- 可访问性（aria-label、figcaption、专用声明）
- 暗色模式

## 社交与治理

- 去中心化社交层（AT Protocol / atproto）避免审核孤岛
- 本地 CLI 管理代理（网站不持有凭证）
- 开放 lexicon（自定义记录类型公开，允许第三方读写）

## 跨生态参考

| 特性 | JS (npmx) | Rust (crates.io) | Python (PyPI) | Elixir (Hex) |
|------|-----------|-------------------|---------------|-------------|
| 传递性大小 | ✅ | ❌ (仅 tarball) | ❌ | ❌ |
| 版本 diff | ✅ | ✅ (cargo-vet) | ❌ | ✅ (diff.hex.pm) |
| 国际化 | ✅ (30+) | ❌ | ✅ (Warehouse) | ❌ |
| 暗色模式 | ✅ | ✅ | ✅ | ❌ |

## 详见

- [[sources/Features-everyone-should-steal-from-npmx]] — npmx 20 项功能详解
- [[entities/npmx]] — npmx 项目概览
