---
title: React Native 新架构(New Architecture)
date: 2026-07-09
tags: [new-architecture, fabric, turbomodules, jsi, codegen, yoga, hermes, performance]
sources: [2026-07-09-reactnative-dev-new-architecture.md]
---

# React Native 新架构

新架构是 RN 从「桥接依赖(bridge-reliant)」向「高性能并发 C++ 核心」的范式转变。理解它是进阶 RN 的分水岭。

## 一句话概括

用 **JSI**(JS↔C++ 直接引用)取代序列化异步桥,在其上重建**渲染层 Fabric** 与**原生模块层 TurboModules**,由 **Codegen** 保证类型安全,**Yoga** 负责布局,**Hermes** 作为默认 JS 引擎。

## 支柱之间的关系

```
        JavaScript (React 18 并发)
                │
              JSI  ← 直接内存引用,免序列化(取代 bridge)
        ┌───────┴────────┐
   TurboModules        Fabric Renderer
   (原生模块)           (UI 渲染)
        │                │
     Codegen(类型安全接口生成)
        │                │
   原生平台 API        Yoga(布局计算)→ 原生视图
```

> **图例(两条分支的终点)**
>
> - **原生平台 API** = 操作系统的**非 UI 功能能力**:相机、定位、文件、蓝牙、通知、传感器等(iOS 的 CoreLocation/AVFoundation、Android 的 LocationManager/Camera2)。由 **TurboModules** 访问 —— 决定 App"能做什么"。
> - **原生视图** = 屏幕上真实的 UI 控件,即 [[concepts/05-glossary]] 中的 **Host View**(iOS `UIView`/`UILabel`、Android `TextView`/`ViewGroup`)。由 **Fabric** 渲染 —— 决定 App"画什么"。

- **JSI** 是地基:提供 JS 与 C++ 的同步、直接互操作能力。
- **TurboModules** 在 JSI 上实现原生模块:类型安全、按需懒加载,消除启动时全量注册开销。
- **Fabric** 在 JSI 上实现渲染:C++ 核心、跨平台共享、支持同步布局与 React 18 并发。
- **Codegen** 从 JS 的类型声明生成 C++/Java/ObjC 接口,是 TurboModules 与 Fabric 类型安全的前提。
- **Yoga** 做 Flexbox 布局计算;**Hermes** 是随包分发的默认 JS 引擎。

## 渲染管线:Render → Commit → Mount

1. **Render** — React 执行,生成 React Element Tree,并在 C++ 侧构造 React Shadow Tree。
2. **Commit** — 后台线程用 Yoga 计算布局,并把新 Shadow Tree 提升为待挂载树。
3. **Mount** — UI 线程做 C++ diff(含 view flattening),生成 mutation 应用到 Host View Tree。

> 三阶段、结构共享、C++ state 更新等细节见 [[concepts/02-fabric-render-pipeline]];线程调度见 [[concepts/03-threading-model]]。

## 新架构解决的三类问题

1. **同步布局与副作用** — 消除测量后再更新 state 造成的视觉跳动(单次 commit 内完成)。
2. **并发渲染** — 自动批处理、Suspense、Transitions、`useTransition` 可中断更新。
3. **高带宽 JS/原生互操作** — JSI 免序列化,支撑相机/动画等高频数据场景。

## 版本与迁移要点

- RN 0.68 实验性 → **0.76+ 默认开启** → **0.82 起彻底移除 legacy bridge**。
- 默认启用;关闭:Android `newArchEnabled=false`,iOS `RCT_NEW_ARCH_ENABLED=0`。
- ⚠️ 迁移到新架构本身不自动提速;需重构以利用同步 API、并发特性与 JSI 才能兑现收益。
- 社区基准(需独立复核):相较 bridge 时代冷启动 ~40%↓、渲染 ~35%↑、内存 ~25%↓、JS→native 调用延迟约 40×↓。

## 交叉引用

- 它取代的旧架构(Bridge)与 JSI 通信机制:[[concepts/06-legacy-bridge]]
- 渲染管线深入:[[concepts/02-fabric-render-pipeline]]
- 线程模型:[[concepts/03-threading-model]]
- 术语表:[[concepts/05-glossary]]
- 性能优化:[[concepts/04-performance-optimization]]
- 来源摘要:[[sources/02-official-new-architecture]]、[[sources/03-official-architecture-deep-dive]]
- 入门主线:[[sources/01-official-getting-started]]
