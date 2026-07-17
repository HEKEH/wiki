---
title: 新架构术语表
date: 2026-07-09
tags: [glossary, reference, new-architecture]
sources: [2026-07-09-reactnative-dev-architecture-deep-dive.md, 2026-07-09-reactnative-dev-new-architecture.md]
---

# 新架构术语表

快速查阅新架构核心术语。展开机制见 [[concepts/01-new-architecture]] 与 [[concepts/02-fabric-render-pipeline]]。

- **React Element / Element Tree** — 描述屏幕内容的纯 JS 对象(props/styles/children),仅存在于 JS;可为 Composite 或 Host 组件的实例。
- **React Shadow Node / Shadow Tree** — 由 Fabric 创建,代表一个待挂载的 Host Component;含来自 JS 的 props + 布局信息(x, y, w, h);新架构下位于 C++。
- **Host Component** — 视图实现由宿主视图提供的组件(`<View>`、`<Text>`;ReactDOM 中的 `<div>`、`<p>`)。
- **Host View / Host View Tree** — 宿主平台的原生视图树(Android `ViewGroup`/`TextView`,iOS `UIView`);尺寸/位置来自 Yoga 的 `LayoutMetrics`。
- **Fabric Renderer** — 让 React 渲染到原生视图而非 DOM;JS 侧对接 C++ 接口。
- **JSI (JavaScript Interface)** — 将 JS 引擎嵌入 C++ 应用的轻量 API;Fabric ↔ React 通信基础;取代序列化异步桥。
- **TurboModules** — JSI 之上的原生模块系统,类型安全、按需懒加载。
- **Codegen** — 以 JS 类型声明为准,生成类型安全的 C++/Java/ObjC 接口(TurboModules 与 Fabric 的前提)。
- **Yoga / Yoga Tree** — Flexbox 布局引擎;每个 Shadow Node 通常对应一个 Yoga Node(非强制)。
- **JNI (Java Native Interface)** — Fabric C++ 核心 ↔ Android 的通信 API。
- **Hermes** — RN 随包分发的默认 JS 引擎(见 [[entities/01-hermes]])。
- **React Composite Components** — `render` 归约为其它 composite/host 组件的组件。
- **React Native Framework** — 以 React 范式交付到原生目标的框架(如 Expo,见 [[entities/02-expo]])。
- **Host Platform** — 承载 RN 的平台(Android、iOS、macOS、Windows)。

## 交叉引用

- 来源:[[sources/03-official-architecture-deep-dive]]、[[sources/02-official-new-architecture]]
