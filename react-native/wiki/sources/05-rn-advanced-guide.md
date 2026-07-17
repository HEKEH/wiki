---
title: React-Native-Advanced-Guide(开源进阶书)
date: 2026-07-09
tags: [advanced-guide, roadmap, testing, hermes, performance, patterns]
sources: [2026-07-09-rn-advanced-guide-toc.md]
---

# React-Native-Advanced-Guide(anisurrahman072)

系统性的开源进阶书,12 章 / 70+ 主题,iOS & Android。作为**进阶学习路线图**,补齐官方文档不覆盖的测试、调试工具链、反模式与工程实践。

## 章节地图

| 章 | 主题 | 已在库中对应 |
|----|------|--------------|
| 1 | 新架构深入(Codegen/JSI/Hermes/TurboModules/Fabric/Yoga) | [[concepts/01-new-architecture]]、[[concepts/02-fabric-render-pipeline]] |
| 2 | 调试/剖析/优化(Dev Menu、DevTools、FPS、线程模型、Flipper、Instruments、AS Profiler) | [[concepts/03-threading-model]]、[[concepts/04-performance-optimization]] |
| 3 | RNTL 组件测试(Jest、query/events、WaitFor、mocking) | — 待建 |
| 4 | Hermes & Static Hermes(bytecode、release bundle) | [[entities/01-hermes]] |
| 5 | 启用新架构(环境、Android/iOS 命令、验证) | — 待建 |
| 6 | 性能优化(列表虚拟化、内存、图片、缓存、动画调度) | [[concepts/04-performance-optimization]] |
| 7 | 列表虚拟化(VirtualizedList/FlatList/SectionList) | — 待建 |
| 8 | FlashList & cell recycling(RecyclerListView、迁移) | — 待建 |
| 9–10 | 反模式(嵌套虚拟化、组件调用违规) | — 待建 |
| 11 | 内购(iOS/Google IAP、RevenueCat) | — 待建 |
| 12 | 进阶模式(HOC、render props、custom hooks、state lifting) | — 待建 |

## 用途

- 作为进阶主线,建议在吃透新架构后按 Ch2 → Ch6/7/8 → Ch3 顺序深入。
- 后续可将 Ch3(测试)、Ch7/8(列表)、Ch12(模式)抓取为独立 concept 页。

## 交叉引用

- 学习顺序:[[analysis/01-source-backlog]]
