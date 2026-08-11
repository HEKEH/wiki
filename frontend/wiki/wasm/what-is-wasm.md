---
title: WebAssembly 是什么 —— 定位、目标与历史
date: 2026-08-10
tags: [wasm, 概念, 历史, 面试]
sources: [mdn-wasm/guides/concepts.md, wasm-design/HighLevelGoals.md, wasm-design/FAQ.md, wasm-design/UseCases.md, wasm-design/Rationale.md]
---

# WebAssembly 是什么

一句话：**WebAssembly（Wasm）是一种为「编译目标」而设计的、可移植的二进制指令格式，
运行在一个安全沙箱里的栈式虚拟机上，目标是让 C/C++/Rust 这类语言以接近原生的速度运行在 Web 上。**

注意这句话里的三个限定，每一个都是面试考点：

1. **为编译目标而设计**——它不是给人手写的语言（虽然有文本格式 WAT 用于阅读和调试）。
2. **可移植的二进制格式**——不是机器码，浏览器拿到后仍要编译成本机机器码。
3. **安全沙箱 + 栈式虚拟机**——决定了它的内存模型、类型系统和很多"看起来别扭"的设计。

> **面试落点**：Wasm 不是"在浏览器里跑机器码"，而是一个**平台无关的虚拟 ISA**；
> 浏览器仍然要把它编译成本机代码，只是这个编译比解析 JS 快得多、且结果可预测得多。

## 四个官方设计目标

来自 W3C WebAssembly Community Group 的 High-Level Goals：

| 目标 | 含义 | 对你的影响 |
| --- | --- | --- |
| **快、高效、可移植** | 利用各平台通用的硬件能力，接近原生速度 | 类型系统只有少量基本类型（见 [[wasm/types-and-abi]]） |
| **可读、可调试** | 有 1:1 对应的文本格式，支持 View Source | 有 WAT（见 [[wasm/text-format-wat]]） |
| **安全** | 沙箱执行，遵守同源策略与权限策略 | 内存隔离、CFI、trap（见 [[wasm/security-sandbox]]） |
| **不破坏 Web** | 与既有 Web 技术共存，向后兼容 | 无版本号、靠 feature detection 演进（见 [[wasm/proposals-and-versions]]） |

还有两条常被忽略但很重要的：

- **增量式规范**：新特性以独立 proposal 推进，核心 spec 只管纯沙箱计算，
  宿主交互（JS API、Web API）被分到更高的规范层。这就是为什么 `WebAssembly.instantiate`
  不在 core spec 里，而在 **js-api** 规范里。
- **不偏向任何语言族**：所以 core wasm 里没有字符串、没有对象、没有 GC（直到 Wasm 3.0 才加 GC）。

## 为什么不用 asm.js？

asm.js 是 Wasm 的前身——一个 JS 的严格子集，靠 `x|0`、`+x` 这类类型标注让引擎能 AOT 编译。
它证明了"把 C++ 编译到 Web 并跑得很快"是可行的，但有两个硬伤：

1. **解析太慢**。asm.js 本质仍是 JS 源码文本。官方实验数据：解码 Wasm 二进制比解析等价
   asm.js 源码**快约 23 倍**；gzip 后的 Wasm 比 gzip 后的 asm.js **小 20–30%**。
   移动端上大型编译产物"光是 parse 就要 20–40 秒"，这是致命的冷启动问题。
2. **被 JS 语义绑死**。asm.js 必须同时满足"是合法 JS"和"能 AOT"两个约束，
   想加 SIMD、多线程、64 位整数这类特性极其别扭。新建一个格式反而更容易演进。

> **面试落点**：Wasm 相对 asm.js 的核心收益是**冷启动**（二进制解码 ~23× 快于解析）
> 和**演进自由度**，而不是稳态执行速度——asm.js 稳态性能本来就不差。

## 为什么不直接用 LLVM bitcode？

这是一道区分度很高的题。LLVM IR 看起来现成，但目标错配：

| Wasm 的要求 | LLVM IR 的现实 |
| --- | --- |
| 可移植：同一程序在所有架构上表示相同 | IR 依赖 target，同一程序不同架构表示不同 |
| 稳定：格式不能随时间改变 | IR 随优化和语言需求持续演进 |
| 编码小、解码快 | bitcode 是为链接期临时序列化设计的，不为压缩 |
| 最小化不确定性 | IR 保留大量 C/C++ undefined behavior |

PNaCl 确实做过"裁一个可移植 LLVM IR 子集"的尝试，但每加一层定制就少享受一分通用基础设施的好处。

> **面试落点**：LLVM IR 是**为编译器优化设计的中间表示**，Wasm 是**为分发和快速启动设计的
> 目标格式**。前者要表达力和优化便利，后者要稳定、紧凑、快速解码、行为确定。

## 与 JavaScript 的关系

**Wasm 不是来取代 JS 的**，这是官方 FAQ 的原话。它们的分工：

```text
JavaScript  →  高层、动态、生态庞大 → UI、胶水、业务逻辑
WebAssembly →  低层、静态、可预测   → 计算密集内核、移植既有 C/C++/Rust 代码
```

同一个 VM 现在加载并运行两类代码，二者可以**同步互相调用**。
更精确地说：Wasm 无法直接访问 DOM 和 Web API，必须调用导入的 JS 函数来间接访问
（GC proposal 之后 externref 让传递宿主对象方便了很多，但"直接调用 Web API"仍是未来工作）。

现实中的四种组合形态：

1. 整个应用编译成 Wasm，JS 只做胶水（Unity/Unreal 游戏、AutoCAD Web）。
2. 主体在 Wasm + 一块 canvas，UI 用 HTML/CSS/JS（Figma、Google Earth）。
3. 大部分是普通 Web 应用，只有少数计算模块用 Wasm（图像处理、编解码、压缩、加解密）。
4. 把某个语言的 VM 整个编译进 Wasm（Pyodide 跑 CPython、Ruby.wasm）——代价是体积大、
   丢失浏览器 devtools 集成。

详见 [[wasm/use-cases]] 与 [[analysis/when-to-use-wasm]]。

## 版本时间线

| 时间 | 里程碑 |
| --- | --- |
| 2015 | WebAssembly CG 成立，asm.js 的继任者立项 |
| 2017 | MVP 达成跨浏览器共识，四大引擎发布支持 |
| 2019 | **Wasm 1.0** 成为 W3C Recommendation |
| ~2022 | **Wasm 2.0**：多返回值、引用类型、bulk memory、定宽 SIMD、非陷阱浮点转整数、符号扩展 |
| ~2025 | **Wasm 3.0**：**GC**、**异常处理**、**memory64**、多内存、尾调用、relaxed SIMD、分支提示、JS String Builtins |

⚠️ 只有 **1.0 是明确的 W3C Recommendation**。2.0 / 3.0 指的是**规范文档的版本号**——
它们是"截至某时点已并入 spec 的 proposal 集合"的快照，走的是 living standard 式的演进，
W3C 流程状态另算。年份是各自最后一批 proposal 通过 WG 表决的大致时间
（依据 `raw/wasm-design/proposals-finished.md`：2.0 的最后一项 Fixed-width SIMD 是
2021-07-14，3.0 的最后一批是 2025-07-23），**不要当成精确的发布日期**。

> **面试落点**：说得出 "Wasm 3.0 把 GC 和异常处理带进了 spec" 就已经超过大多数候选人——
> 这意味着 Java/Kotlin/Dart/OCaml 这类 GC 语言不必再自带一个 GC 编进线性内存，
> 产物体积和与 JS 的对象互操作都有质变。详见 [[wasm/proposals-and-versions]]。

## 常见误解清单

### "Wasm 比 JS 快 10 倍"

> **⚠️ 常见误答**：没有这种普适倍数。稳态计算密集场景 Wasm 通常是 JS 的 **1–3 倍**；
> 但涉及大量 JS 互操作或字符串/对象操作时，**Wasm 可能更慢**。详见 [[wasm/performance]]。

### "Wasm 可以直接操作 DOM"

> **⚠️ 常见误答**：不能。必须通过导入的 JS 函数间接调用。
> `externref` 让你能持有 DOM 引用而不必编号管理，但调用仍要过 JS。

### "Wasm 是给浏览器用的"

> **⚠️ 常见误答**：浏览器只是**一种 embedder**。
> WASI + wasmtime/wasmer 让 Wasm 成为服务端沙箱、插件系统、边缘计算的运行时。
> 官方 High-Level Goals 第 4 条就是"支持非浏览器嵌入"。

## 相关页面

- 执行模型：[[wasm/core-model]]
- 文本格式：[[wasm/text-format-wat]]
- 与 JS 互操作：[[wasm/js-interop]]
- 何时该用：[[analysis/when-to-use-wasm]]
