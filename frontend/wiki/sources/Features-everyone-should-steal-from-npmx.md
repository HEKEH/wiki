---
title: "Features Everyone Should Steal From npmx"
date: 2026-05-24
tags: [npmx, package-registry, ux, supply-chain, accessibility]
sources: [Features-everyone-should-steal-from-npmx.md]
---

# Features Everyone Should Steal From npmx

## 摘要

npmx.dev 是 Daniel Roe 于 2026 年 1 月创建的 npm 注册表替代前端，采用 MIT 开源许可。其成功迫使 npmjs.com 多年来首次推出实质性更新（如暗色模式）。文章系统梳理了 npmx 的 20 项可借鉴功能特性，适用于任何构建包注册表网站的场景。

## 核心要点

### 依赖安全与透明度
- **传递性安装大小** — 展示包及所有依赖的解压总大小，而非单一 tarball 大小。类似工具：bundlephobia、packagephobia
- **安装脚本披露** — 渲染 preinstall/install/postinstall 脚本内容及 npx 拉取的包，可追溯供应链攻击入口
- **过时与漏洞依赖树** — 递归标注每个依赖节点与最新版的差距及 OSV 漏洞状态，类似 deps.dev

### 开发者体验
- **语义版本范围解析** — `^1.0.0` 旁显示当前解析到的具体版本，省去 CLI 查询
- **模块替代建议** — 基于 e18e module-replacements 数据集，标注轻量替代方案和原生 API（附 MDN 链接）
- **模块格式与类型徽章** — ESM/CJS/双模式、TypeScript 类型来源、Node 引擎范围一目了然
- **跨注册表可用性** — 标注同一包是否存在于 JSR，兼作依赖混淆检查
- **Playground 链接提取** — 从 README 提取 StackBlitz/CodeSandbox/CodePen 等可交互链接

### 对比与可视化
- **并排包对比** — 最多 10 个包的指标表格 + 牵引力/人体工学散点图
- **版本差异** — 浏览器内逐文件 diff 任意两个版本（类似 diff.hex.pm、cargo-vet）
- **发布时间线与大小标注** — 时间线上标记安装大小显著跳升的版本，便于定位问题发布
- **版本下载分布** — 周下载量按 major/minor 线分解，展示生态迁移进度

### 平台特性
- **命令面板** — `⌘K` 打开，支持当前页所有操作和全局导航，语义版本范围过滤
- **国际化** — 30+ 语言含 RTL，类似 PyPI Warehouse
- **可访问性** — 图表/视频配 aria-label 和 figcaption，专用可访问性声明
- **Agent Skill 检测** — 列出包中的 Agent Skills 清单及工具兼容性
- **AT Protocol 社交功能** — 包点赞为 atproto 记录，评论为 Bluesky 线程，自定义 lexicon 开放
- **本地 CLI 管理连接器** — 管理操作通过本地 npm CLI 代理，网站无需持有凭证
- **暗色模式与自定义调色板**

## 影响

- 竞争压力已推动 npmjs.com 发布暗色模式（5 年来最高票请求）
- .NET 生态已出现类似项目 nugx.org（"inspired by npmx"）
- 所有功能均有 MIT 开源参考实现，非截图级设计

## 关键引用

> 如果你想给注册表加评论/评价但顾虑审核负担，借用现有网络的身份和内容层是一个可辩护的方案
