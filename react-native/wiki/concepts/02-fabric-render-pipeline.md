---
title: Fabric 渲染器与 Render→Commit→Mount 管线
date: 2026-07-09
tags: [fabric, render-pipeline, shadow-tree, view-flattening, yoga, jsi]
sources: [2026-07-09-reactnative-dev-architecture-deep-dive.md]
---

# Fabric 渲染器与渲染管线

Fabric 是新架构的渲染系统(2018 起,2021 年 Facebook app 全量切换)。核心目标:**在 C++ 统一渲染逻辑**、改善与原生平台的互操作、解锁 React 18 能力。是 [[concepts/01-new-architecture]] 的渲染支柱。

## 三棵树

```text
React Element Tree (JS)  ──JSI──▶  React Shadow Tree (C++)  ──▶  Host View Tree (原生)
   组件声明/props/children        含 props + 布局(x,y,w,h)      ViewGroup/UIView 等真实视图
```

- **Element Tree**:纯 JS 对象,由 React 生成,**immutable(不可变)+ temporal(每次渲染临时重建、用完即弃)**,只表达"这一帧该长什么样"。支撑它、并**跨渲染持久保存 state/hooks** 的,是 React 内部的 **fiber 树**(每个组件实例对应一个 fiber);在 RN 中每个 fiber 还通过 JSI **持有指向对应 C++ Shadow Node 的指针**,作为 JS 侧与 C++ 侧的连接点。
- **Shadow Tree**:由 Fabric 在 C++ 侧创建并**常驻**(留作跨 Commit/Mount 阶段 diff);每个 Host Component 生成一个 Shadow Node,composite 组件不生成。immutable,更新靠 clone。
- **Host View Tree**:平台原生视图,尺寸/位置来自 Yoga 计算的 `LayoutMetrics`。

Fabric 关键特性:C++ 核心跨平台共享(一处优化全平台受益)、Codegen 以 JS 声明为准生成 C++ props struct(不匹配即编译报错)、JSI 免序列化、Host Component 懒初始化、同步布局消除嵌入原生视图时的跳变。

## 渲染管线:Render → Commit → Mount

初次渲染与状态更新都走此三阶段。

| 阶段 | 线程 | 同步性 | 做什么 |
|------|------|--------|--------|
| **Render** | JS(+C++) | 同步 | React 跑逻辑生成 Element Tree;经 JSI 同步创建 C++ Shadow Tree |
| **Commit** | JS 或后台 | 异步 | 原生 Yoga 引擎计算布局(Text/TextInput 调用平台测量);把新树"提升"为待挂载树,排入下一 UI tick |
| **Mount** | UI 主线程 | 同步 | C++ diff 生成 `createView/updateView/removeView/deleteView`(含 view flattening);提升为已渲染树;把 mutation 应用到原生视图 |

典型树 600–1000 节点,经 view flattening 后约 ~200。

### Mount 的 mutation 指令集

**mutation = diff 产出的、对原生视图树的最小原子变更指令。** Mount 阶段对比"上次渲染的树"与"新树",算出"改到新样子最少需要哪几步",每步即一个 mutation:

| mutation | 含义 |
|----------|------|
| `createView` | 新建一个原生视图 |
| `updateView` | 更新视图的 props / 样式 |
| `insertView` / `removeView` | 挂载 / 摘除子视图 |
| `deleteView` | 销毁视图 |

- 得益于结构共享,变更通常只产出最小 mutation(如背景色红→黄 = 一条 `updateView(Node, {backgroundColor:'yellow'})`),而非重建整树。
- 思路等价于 Web 上 React 对 DOM 打的增删改补丁,只是目标换成原生视图而非 DOM。
- **分工**:mutation 由 **C++ 核心算出**,但真正"执行"(实例化 `UIView` / `android.view.View`)仍在**平台原生层、UI 线程**完成——只有平台能创建原生控件(见 [[concepts/03-threading-model]])。

## 状态更新的两条路径

- **React state 更新**:结构共享(structural sharing)——只 clone 受影响节点及其祖先,未变节点在新旧树间共享。Mount diff 只产出最小 mutation(如 `UpdateView(Node, {backgroundColor:'yellow'})`)。
- **C++ state 更新**(如 ScrollView 偏移):**跳过 Render 阶段**,可来自任意线程;clone 节点写入新 state 后提交,若与其它更新冲突则**重试**直到成功,避免竞态。

### 什么是 C++ state 更新

指 Shadow Node 上那份**由原生/C++ 侧持有、不经过 React 渲染就能直接更新**的状态。它是与 React state 并列的第二种状态来源。

**为什么需要**(两类场景):

1. **高频更新,不该惊动 React**:典型是 **ScrollView 滚动偏移** —— 手指一滑 offset 每帧都变,若每像素都 `setState → 重渲染` 会打爆 JS 线程,故直接更到 C++ state。
2. **只有原生才知道的值**:如 `<Image>` 加载后才知的**原始尺寸**、文本测量出的**内容尺寸**,由原生算出后写回 C++ state 供布局使用。

**对比 React state 更新:**

| | React state 更新 | C++ state 更新 |
|---|---|---|
| 触发方 | JS 里 `setState` | 原生侧(滚动、图片加载、测量…) |
| 走 Render 阶段 | ✅ 生成新 Element/Shadow Tree | ❌ **跳过** |
| 线程 | JS 线程 | **任意线程**(UI 或后台) |
| 之后 | Commit → Mount | Commit(带重试)→ Mount |

两者最终都汇入同一 Mount 阶段(重算布局 → diff → 打 mutation)。

**为什么"冲突就重试"**:C++ state 可来自任意线程,同期 React 或另一 C++ 更新可能在改同一节点,存在竞态。做法是**乐观提交**:取节点最新已提交版本 → clone 写入新 state → 尝试提交 → 若期间树被改过(冲突)则回到第一步重试,直到成功。既免加锁,又保证不覆盖他人更新,呼应下文 [不可变性与线程安全](#不可变性与线程安全)。

> 直觉:**React state** = JS 主动说"UI 该变了,请重渲染";**C++ state** = 原生侧就地记一笔"我这块实际情况(滚到哪/图多大)变了",不劳 React 重跑,直接更到 shadow node。

## View Flattening

把只影响布局的 **Layout-Only 节点**(仅 margin/padding/opacity/backgroundColor 等)合并进父节点,削减视图层级深度。

- React 的组合式 API 天然产生大量"只管布局"的嵌套 View。
- 算法**内嵌在 diff 阶段**,不额外耗 CPU;检测 Layout-Only → 样式并入父视图 → 删除对应 host view。
- C++ 实现,全平台默认生效,对用户零可见变化。原是 Android 优化,现惠及 iOS。

## 不可变性与线程安全

Element Tree 与 Shadow Tree 均 immutable(C++ const correctness):每次更新 clone 新对象而非原地改。因此框架无需锁即可暴露**线程安全的同步 API**,同时结构共享降低内存开销。

## 为什么核心用 C++

Fabric 核心(含 diff / mutation 计算、Shadow Tree)刻意用 C++,而非 Java/Kotlin/Swift:

1. **一次编写,全平台共享**。C++ 核心 iOS/Android 共用一份;旧架构里同套逻辑在 Java 与 Objective-C 各写一遍、行为易漂移。一处优化(如 [view flattening](#view-flattening))即惠及全平台。
2. **两平台的"最大公约数"**。C++ 可经 **JNI** 调 Android、经 **Objective-C++** 调 iOS;Java / Swift 都无法被两端共享。
3. **JSI 本身就是 C++ API**。整个 Fabric 对接 C++ 接口,渲染器住在 C++ 里就能贴着 JSI 跑,免跨语言边界与序列化(见 [[concepts/05-glossary]] 的 JSI 词条)。
4. **性能 + 确定性内存,无 GC 停顿**。diff/mount 是高频热点;C++ 编译为机器码,靠 RAII 与 const correctness 管理不可变树,既快又实现无锁线程安全,不依赖某 runtime 的 GC。

一句话:C++ 是"能被 iOS/Android 同时调用、又快又可精确控制内存、还正好是 JSI 母语"的唯一选择,也是新架构相对旧桥接架构的根本升级点之一。

## 交叉引用

- 线程调度细节:[[concepts/03-threading-model]]
- 术语表:[[concepts/05-glossary]]
- 上层总览:[[concepts/01-new-architecture]]
- 来源:[[sources/03-official-architecture-deep-dive]]
