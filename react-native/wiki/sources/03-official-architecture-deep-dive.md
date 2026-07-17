---
title: 官方文档 — Architecture 深入(Fabric/管线/线程/Flattening/术语)
date: 2026-07-09
tags: [official-docs, fabric, render-pipeline, threading-model, view-flattening, glossary]
sources: [2026-07-09-reactnative-dev-architecture-deep-dive.md]
---

# 官方 Architecture 专章子页(合集)

一次性抓取的五个官方 architecture 子页,构成新架构的**一手权威深度来源**。综合与展开已拆入各 concept 页,本页仅作来源索引与要点。

## 覆盖的子页

| 子页 | 对应 concept 页 |
|------|----------------|
| Fabric Renderer | [[concepts/02-fabric-render-pipeline]] |
| Render, Commit, and Mount | [[concepts/02-fabric-render-pipeline]] |
| Threading Model | [[concepts/03-threading-model]] |
| View Flattening | [[concepts/02-fabric-render-pipeline]](View Flattening 节) |
| Glossary | [[concepts/05-glossary]] |

## 要点摘录

- Fabric:C++ 核心跨平台共享;三棵树 Element → Shadow → Host View;Codegen 类型安全;JSI 免序列化;懒初始化。
- 管线:Render(JS 同步)→ Commit(后台异步,Yoga 布局 + 树提升)→ Mount(UI 线程同步,C++ diff + view flattening + 挂载)。
- 结构共享:React state 更新只 clone 受影响节点及祖先;C++ state 更新跳过 render、可跨线程、冲突重试。
- 线程:JS 线程跑 render/布局,UI 线程独占视图操作;不可变数据结构保证无锁线程安全。
- View Flattening:合并 Layout-Only 节点,600–1000 → ~200 节点。

## 待抓取(同专章尚缺)

- Cross-Platform Implementation、Bundled Hermes、Architecture Overview。见 [[analysis/01-source-backlog]]。

## 交叉引用

- 上层:[[sources/02-official-new-architecture]]、[[concepts/01-new-architecture]]
