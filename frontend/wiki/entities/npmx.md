---
title: npmx.dev
date: 2026-05-24
tags: [npmx, package-registry, open-source, atproto]
sources: [Features-everyone-should-steal-from-npmx.md]
---

# npmx.dev

npm 注册表的开源替代前端，由 Daniel Roe 于 2026 年 1 月创建，MIT 许可。

## 核心特点

- **URL 兼容** — 所有 npmjs.com URL 直接替换主机名即可访问（npmx.dev 或 xnpmjs.com），类似 Invidious/Nitter 的兼容策略
- **多代码托管平台支持** — GitHub、GitLab、Bitbucket、Codeberg、Gitee、Sourcehut、Forgejo、Gitea、Radicle、Tangled
- **AT Protocol 集成** — 运行自有 PDS（npmx.social），包点赞为 atproto 记录，评论为 Bluesky 线程
- **本地 CLI 代理** — 管理操作通过本地 npm CLI 执行，不持有用户凭证

## 影响

- 两周内吸引 1000+ issues/PRs、100+ 贡献者
- 迫使 npmjs.com 发布暗色模式
- 启发 .NET 生态的 nugx.org

## 详见

- [[sources/Features-everyone-should-steal-from-npmx]] — 完整功能列表与设计理念
