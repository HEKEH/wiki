# Home — React Native Knowledge Base

系统学习 React Native 的知识库,由 LLM agent 增量维护。`raw/` 为不可变原始抓取,`wiki/` 为 LLM 组织的摘要、概念页与综合。

## Quick Links

- [[index]] — 全部页面目录
- [[log]] — 活动日志
- [[analysis/01-source-backlog]] — 精选源清单与抓取计划(学习顺序在此)

## Current State

**已建立新架构主干 + 性能主线(2026-07-09)。**

学习动线:

1. 入门:[[sources/01-official-getting-started]]
2. 新架构总览:[[concepts/01-new-architecture]]
3. 渲染管线深入:[[concepts/02-fabric-render-pipeline]]
4. 线程模型:[[concepts/03-threading-model]]
5. 性能优化:[[concepts/04-performance-optimization]]
6. 速查:[[concepts/05-glossary]]、[[entities/01-hermes]]
7. 生态/工具链:[[entities/02-expo]](React Native Framework)

## Core Insights

- RN 用 JS/React 渲染**真实原生组件**,非 WebView。
- 2026 年的 RN = 新架构:**JSI 取代序列化异步桥**,其上重建 Fabric(渲染)与 TurboModules(原生模块),Codegen 保证类型安全,Yoga 布局,Hermes 为默认引擎。
- 渲染 = **Render(JS 同步)→ Commit(后台异步,Yoga)→ Mount(UI 线程同步,C++ diff)**;树不可变 + 结构共享 → 无锁线程安全与最小 mutation。
- 60fps 需 **JS 线程与 UI 线程都不掉帧**;性能优化的第一原则是**先 profile**,再按影响排序。
- 新架构自 0.76 默认、**0.82 起移除 legacy bridge**;但迁移不自动提速,须重构利用新 API。

## Open Questions

- 官方 architecture 尚缺:Cross-Platform Implementation、Bundled Hermes、Architecture Overview。
- 尚未系统覆盖:状态管理、原生模块(TurboModule/Fabric component)开发、测试(RNTL)、列表虚拟化深挖。导航方向已有 [[entities/02-expo]](含 Expo Router 概览),React Navigation 独立深挖仍待补。
- 社区性能数字(冷启动 ~40%↓、FlashList 快 5× 等)在真实项目的可复现性,需实测验证。
