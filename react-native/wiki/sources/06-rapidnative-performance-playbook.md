---
title: RapidNative — RN 性能优化 2026 Playbook
date: 2026-07-09
tags: [performance, playbook, benchmarks, profiling, third-party]
sources: [2026-07-09-rapidnative-performance-playbook.md]
---

# RapidNative:性能优化 2026 Playbook

第三方实践 playbook,价值在于给出**可量化的性能预算**与**按影响排序的优化清单**。已并入 [[concepts/04-performance-optimization]]。

> ⚠️ 其性能数字(冷启动 < 2.0s、FlashList 快 5×、新架构 ~40×/35%/25% 等)为第三方口径,需在自己项目上复核,不作权威结论。

## 要点

- 量化预算:冷启动、FPS、交互延迟、内存、体积、bundle(见 concept 页表格)。
- 优先级:profile → 新架构 → Hermes → FlashList → 减重渲染 → 动画上 UI 线程 → 冷启动 → 内存 → 网络。
- 核心信条:"不要优化没出现在 profiler trace 里的东西。"

## 交叉引用

- 系统方法:[[concepts/04-performance-optimization]]
- 官方性能:[[sources/04-official-performance]]
