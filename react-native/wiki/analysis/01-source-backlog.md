---
title: 精选源清单与抓取计划
date: 2026-07-09
tags: [backlog, sources, curation, learning-path]
sources: []
---

# 精选源清单与抓取计划

经检索与筛选(过滤掉大量 SEO 型 Medium 短文)后,确认的**成体系、高质量** React Native 学习源。按优先级排序;已入库项标注 ✅。

## 一线权威(官方 / 书籍级)

| 优先级 | 源 | 类型 | 状态 |
|---|---|---|---|
| P0 | [官方文档 Getting Started](https://reactnative.dev/docs/getting-started) | 官方主线 | ✅ [[sources/01-official-getting-started]] |
| P0 | [官方 New Architecture 专章](https://reactnative.dev/architecture/landing-page) | 官方 | ✅ [[sources/02-official-new-architecture]] |
| P0 | 官方 architecture 子页:Fabric / Render-Commit-Mount / View Flattening / Threading Model / Glossary | 官方 | ✅ [[sources/03-official-architecture-deep-dive]] |
| P0 | 官方 architecture 子页:Cross-Platform Implementation / Bundled Hermes / Architecture Overview | 官方 | ⬜ 待抓取 |
| P1 | [React-Native-Advanced-Guide (anisurrahman072)](https://github.com/anisurrahman072/React-Native-Advanced-Guide) | 开源书 | ✅ TOC/路线图见 [[sources/05-rn-advanced-guide]];各章正文待深挖 |
| P1 | [Callstack — Ultimate Guide to React Native Optimization](https://www.callstack.com/ebooks/the-ultimate-guide-to-react-native-optimization) | 权威团队 ebook | ⬜ 待抓取(表单/PDF gated) |

## 二线专题(补充深度)

| 优先级 | 源 | 类型 | 状态 |
|---|---|---|---|
| P1 | [官方 Performance Overview](https://reactnative.dev/docs/performance) | 官方 | ✅ [[sources/04-official-performance]] |
| P2 | [RapidNative — RN Performance Optimization 2026 Playbook](https://www.rapidnative.com/blogs/react-native-performance-optimization-2026-playbook) | 实操(带量化指标) | ✅ [[sources/06-rapidnative-performance-playbook]] |
| P2 | [callstack/optimization-best-practices (GitHub)](https://github.com/callstack/optimization-best-practices) | ebook 配套代码 | ⬜ 待抓取 |
| P2 | [awesome-react-native (jondot)](https://github.com/jondot/awesome-react-native) | 生态索引 | ⬜ 待抓取(用作发现入口) |

## 下一批建议(尚未覆盖的主题)

- **官方子指南**:Profiling、Optimizing FlatList Configuration、Optimizing JavaScript loading、Animations、Debugging。
- **导航**:React Navigation / Expo Router —— Expo 平台首批入库 [[entities/02-expo]](含 Expo Router 概览);
  React Navigation 独立深挖、Expo Router 实操细节仍待抓取。
- **状态管理**:Redux Toolkit / Zustand / Jotai + RN 语境。
- **原生模块开发**:TurboModule / Fabric Native Component 实操(官方 Native Development 章 + Advanced Guide Ch5)。
- **测试**:RNTL(Advanced Guide Ch3)。
- **列表深挖**:虚拟化与 FlashList cell recycling(Advanced Guide Ch7/8)。

## 筛选原则(供后续检索复用)

- 优先:官方文档、核心贡献团队(Meta、Callstack)、结构完整的开源书/ebook、配套可运行代码的仓库。
- 降权:标题党、"2026 最佳/Top N"、缺乏一手验证的 Medium 单篇转述。
- 关键背景需独立复核:新架构性能数字、版本移除时间点(bridge 于 0.82 移除)。

## 建议学习顺序

1. 官方 Getting Started 主线 → 核心组件 / 列表 / 输入 / 样式。
2. 新架构专章 + architecture 子页 → 底层心智模型([[concepts/01-new-architecture]] → [[concepts/02-fabric-render-pipeline]] → [[concepts/03-threading-model]])。
3. React-Native-Advanced-Guide → 进阶系统化(测试、工具链、反模式)。
4. 性能专项:[[concepts/04-performance-optimization]] + Callstack ebook。
5. awesome-react-native → 按需补库/工具。
