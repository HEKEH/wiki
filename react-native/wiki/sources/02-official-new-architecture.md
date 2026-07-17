---
title: 官方文档 — About the New Architecture
date: 2026-07-09
tags: [official-docs, new-architecture, fabric, turbomodules, jsi]
sources: [2026-07-09-reactnative-dev-new-architecture.md]
---

# 官方文档:About the New Architecture

React Native 新架构的官方总览页,是理解 Fabric / TurboModules / JSI / Codegen / Yoga 的**一手权威来源**。本页为来源摘要;机制的综合梳理见 [[concepts/01-new-architecture]]。

## 定位与时间线

新架构是对 RN 核心内部的全面重设计:2018 年启动,2024 年在 Meta 生产环境大规模验证。RN 0.68 实验性可选,**0.76+ 默认开启**。

## 五大支柱

- **Fabric Renderer** — 新渲染系统,支持同步布局/副作用与 React 18+ 并发特性。
- **TurboModules** — 更快、类型安全、按需懒加载的原生模块系统。
- **JSI (JavaScript Interface)** — 取代异步桥,JS 与 C++ 直接持有对方引用、直接调用,免序列化。
- **Codegen** — 从类型定义生成类型安全的原生模块/组件接口。
- **Yoga** — 布局引擎(v2.0.0+ 一致性改进)。

## 新架构带来的三大改进

1. **同步布局与副作用** — 旧 `onLayout` 可能在绘制后才更新 state,导致视觉跳动;新架构在单次 commit 内完成测量+更新(`useLayoutEffect`)。
2. **并发渲染 + React 18** — 自动批处理、Suspense、Transitions、`useTransition`。
3. **快速 JS/原生互操作 (JSI)** — 移除序列化的异步桥,改为直接内存引用,支撑高带宽场景(如 VisionCamera ~2 GB/s 帧数据)。

## 新旧对比

| 方面 | Legacy | New Architecture |
|------|--------|------------------|
| JS/原生通信 | 异步桥(序列化) | JSI(直接内存引用) |
| 渲染 | Legacy Renderer | Fabric Renderer |
| 原生模块 | 静态模块 | TurboModules(更快、类型安全) |

## 采用建议

- 0.76 起默认;除非遇到问题否则直接使用。
- 关闭:Android `newArchEnabled=false`;iOS `ENV['RCT_NEW_ARCH_ENABLED']='0'`。
- **注意**:不重构以利用新 API 时,并非自动获得性能提升,主要价值是面向未来。

## 待深入的子页(官方 architecture 专章)

Fabric、Render/Commit/Mount 管线、Cross-Platform Implementation、View Flattening、Threading Model、Bundled Hermes、Glossary。→ 见 [[analysis/01-source-backlog]] 中的抓取计划。

## 交叉引用

- 综合梳理:[[concepts/01-new-architecture]]
- 入门主线:[[sources/01-official-getting-started]]
