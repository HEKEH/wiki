---
title: 旧桥接架构(Legacy Bridge)与其瓶颈
date: 2026-07-09
tags: [legacy-architecture, bridge, jsi, serialization, migration]
sources: [2026-07-09-reactnative-dev-new-architecture.md, 2026-07-09-reactnative-dev-architecture-deep-dive.md]
---

# 旧桥接架构(Legacy Bridge)与其瓶颈

新架构的意义,一半在于它**替换掉了什么**。这页讲旧架构赖以运转的 **Bridge(桥)**,以及新架构用 **JSI** 如何取代它。是 [[concepts/01-new-architecture]] 的"另一半"。

## Bridge 是什么

旧架构里,JavaScript 与原生代码是**两个隔离的世界**,彼此只能通过一条叫 Bridge 的通道通信。三条线程各司其职,但**只能过桥对话**:

```text
JS 线程  ⇄  Bridge  ⇄  原生 / UI 线程
                       (Shadow 线程负责布局计算)
```

Bridge 有三个定义性特征——也正是它的三个毛病:

1. **序列化(serialization)**:JS↔原生的每条消息都先编码成 **JSON 字符串**,到对面再解码。
2. **异步(asynchronous)**:JS **无法同步调用原生并立刻拿到返回值**,只能发消息、等回调。
3. **批量排队(batched)**:消息进队列,定时批量刷过去。

## 为什么"依赖桥"是瓶颈

- **性能开销大**:高带宽场景(快速滚动、手势、相机每秒数 GB 帧数据)下,JSON 编解码 + 排队成为拖累;桥被"塞满"就丢帧卡顿。
- **拿不到同步结果**:想先**测量**视图尺寸再据此渲染时,异步的桥让测量结果晚一拍返回,造成**视觉跳动**(tooltip 定位的经典例子)。
- **启动慢**:所有原生模块在启动时被**全量初始化**注册。
- **单通道**:所有 JS↔原生流量挤在同一条异步序列化通道里。

## 为什么旧架构必须序列化

关键在于 Bridge 是一条**异步消息队列**:JS 侧和原生侧是两个独立的运行时/内存空间,消息要**跨异步边界**传递。你没法安全地把一个"活的内存指针"丢过异步队列(对面取用时它可能已失效),所以只能把数据**拍平成与内存无关的 JSON 文本**再传。序列化不是设计者的偏好,而是"隔离 + 异步"这个结构逼出来的代价。

## 新架构:JSI 如何让 JS 与 C++ 直接通信

新架构用 **JSI(JavaScript Interface)** 拆掉了这堵墙。JSI 是一套轻量、引擎无关的 **C++ API**,用来把 JS 引擎(Hermes / JSC / V8)**嵌入**到 C++ 程序里(见 [[concepts/05-glossary]] 的 JSI 词条)。

核心变化:**JS 引擎与 C++ 现在同处一个进程、同一线程,JSI 让两侧能直接持有对方的引用、直接调用对方的函数** —— 边界没了,也就不需要序列化了。

几个关键机制:

- **`jsi::Value` 直接包裹引擎里的值**。JS 传一个数字给 C++,C++ 收到的是一个直接指向引擎中该值的 `jsi::Value`,`.getNumber()` 当场读出,全程没有 JSON 文本。
- **HostObject**:一个 C++ 对象可以"伪装"成 JS 对象暴露给 JS。JS 访问它的属性/方法时,引擎**直接回调进 C++**(get/set/调用)。
- **HostFunction**:把一个 C++ 函数装进 JS 运行时,变成可被 JS 直接调用的函数。

于是 **TurboModules** 的原生方法调用,本质就是:JS 调一个 HostFunction → 引擎直接执行对应的 C++ 函数,参数以 `jsi::Value` 直接传递(接口由 [[concepts/05-glossary]] 的 Codegen 生成保证类型安全)。**Fabric** 同理——Shadow Tree 直接在 C++ 侧经 JSI 创建、props 直接读取。

> 类比:旧桥像**两人在隔壁房间传纸条**,每句话都得先写下来(序列化);JSI 像**两人同处一室,直接把东西递到手上、直接喊对方名字调用**——既省去抄写,又能当场拿到回话(同步)。

## 新旧对比

| | 旧架构(桥接依赖) | 新架构 |
|---|---|---|
| JS↔原生通信 | Bridge:异步 + JSON 序列化 | **JSI**:JS↔C++ 直接引用,**同步、免序列化** |
| 能否同步取值 | 否(只能回调) | 能(直接函数调用) |
| 原生模块 | 静态模块,启动全量注册 | **TurboModules**:类型安全、按需懒加载 |
| 渲染 | Legacy Renderer | **Fabric**(C++ 核心) |

## 时间线

Bridge 自 RN **0.76 起默认被新架构取代**,并在 **0.82 起彻底移除**。

> JSI 的双向通信机制、C++ 何时/如何调用 JS,以及完整代码例子,见 [[concepts/07-jsi]]。

## 交叉引用

- JSI 双向通信与代码例子:[[concepts/07-jsi]]
- 新架构总览:[[concepts/01-new-architecture]]
- JSI 之上的渲染实现:[[concepts/02-fabric-render-pipeline]]
- 术语(JSI / Codegen / Host Component):[[concepts/05-glossary]]
- 来源:[[sources/02-official-new-architecture]]、[[sources/03-official-architecture-deep-dive]]
