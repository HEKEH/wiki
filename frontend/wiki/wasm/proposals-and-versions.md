---
title: 版本演进与 Proposal 全景
date: 2026-08-10
tags: [wasm, proposal, 版本, gc, 面试]
sources: [wasm-design/proposals-README.md, wasm-design/proposals-finished.md, wasm-design/HighLevelGoals.md, wasm-design/FeatureTest.md, mdn-wasm/reference/exception_handling.md, mdn-wasm/reference/value_types.md]
---

# 版本演进与 Proposal 全景

## Wasm 没有版本号

二进制头里的版本字段**至今仍是 `1`**。Wasm 走的是 Web 平台一贯的
**"无版本、特性检测、向后兼容"**演进路线。

所谓 "Wasm 2.0 / 3.0" 指的是 **spec 文档的版本**——某个时间点上"已进入 spec 的 proposal 集合"。
引擎的实际支持情况看 <https://webassembly.org/features/>。

> **面试落点**：Wasm 靠 **proposal 独立推进 + feature detection** 演进，不靠版本号。
> 这是它的 High-Level Goals 第 2、3 条明确写死的："维持 Web 的无版本、可特性检测、
> 向后兼容的演进故事"。

## Proposal 的六个阶段

| Phase | 名称 | 含义 |
| --- | --- | --- |
| 0 | Pre-Proposal | 只是 design 仓库里的一个 issue |
| 1 | Feature Proposal | CG 同意值得探索，有独立仓库 |
| 2 | Proposed Spec Text | 有规范文本草案 |
| 3 | Implementation Phase | 规范基本定型，引擎开始实现 |
| 4 | Standardize the Feature | WG 投票通过，进入 spec |
| 5 | The Feature is Standardized | 完全标准化 |

> **面试落点**：引用某个特性时**一定要标阶段**。
> 说"Wasm 有组件模型"是错的——**Component Model 还在 Phase 1**。
> 说"Wasm 有 GC"是对的——**GC 在 Wasm 3.0 里已经进 spec 了**。

## 已完成的 proposal（按 spec 版本）

### Wasm 1.0（2019 W3C Recommendation）

- MVP
- Import/Export of Mutable Globals

### Wasm 2.0（2022）

| Proposal | 意义 |
| --- | --- |
| **Multi-value** | 函数可以返回多个值；指令序列可消费/产生多个栈值 |
| **Reference Types** | 引入 `externref`，Wasm 可直接持有宿主对象；新增 table 操作指令 |
| **Bulk memory operations** | `memory.copy`/`fill`/`init` 等，让 memcpy/memset 一条指令搞定 |
| **Fixed-width SIMD** | `v128` + 128 位向量指令 |
| **JS BigInt ↔ i64** | i64 在 JS 边界上映射为 BigInt |
| Non-trapping float-to-int | `trunc_sat` 系列，超范围饱和而不 trap |
| Sign-extension operators | `extend8_s`/`extend16_s`/`extend32_s` |

### Wasm 3.0（2025）—— 这一版最值得讲

| Proposal | 意义 |
| --- | --- |
| **Garbage collection** | 引入 struct/array 等**引擎管理的堆类型**，GC 语言不必自带 GC |
| **Exception handling** | `throw`/`try_table`/`catch*` + `exnref`，异常可跨 Wasm/JS 边界 |
| **Memory64** | 64 位线性内存，突破 4GiB 限制 |
| **Multiple memories** | 一个 instance 多块内存 |
| **Tail call** | 尾调用优化，函数式语言和状态机的必需品 |
| **Relaxed SIMD** | 用结果不确定性换性能，直接映射本机指令 |
| **JS String Builtins** | Wasm 可直接操作 JS 字符串（`externref`） |
| **Branch Hinting** | 给基线编译器分支概率提示 |
| Extended Const Expressions | 初始化表达式支持更多运算 |
| Typed Function References | 带具体签名的函数引用类型 |
| Custom Annotation Syntax | 文本格式的自定义注解 |

## 为什么 GC 是 Wasm 3.0 最重要的特性

在 GC 之前，Java/Kotlin/Dart/C#/OCaml 这类 GC 语言编译到 Wasm 只有一条路：
**把整个 GC 运行时也编译进线性内存**。后果：

- 产物体积暴涨（一个 GC 实现动辄几百 KB）
- **和 JS 的对象无法共享**——两个世界各有各的堆
- **跨堆循环引用无法回收**（JS 对象引用 Wasm 对象，反之亦然，谁也回收不了对方）
- 引擎的 GC 优化（分代、并发、增量）完全用不上

有了 GC proposal，Wasm 可以声明 `struct`/`array` 类型，**由引擎的 GC 管理**，
和 JS 对象活在同一个堆里：

```wat
(type $Point (struct (field $x f64) (field $y f64)))
;; 配套指令：struct.new / struct.get / array.new / ref.cast / ref.test ...
```

### GC 不回收线性内存

> **⚠️ 常见误答**："Wasm 3.0 之后引擎会帮我回收内存了" —— **线性内存永远不会被 GC**。

线性内存就是一块 `ArrayBuffer`——引擎根本不知道里面哪些字节构成"一个对象"、
哪个 i32 是指针，它看到的只有字节。所以引擎既无法也不会回收它，
那里面的分配始终由你自己管（`malloc`/`free`，或 Rust 的所有权）。

GC proposal 加的是**另一套东西**：

| | 线性内存里的数据 | GC 堆对象 |
| --- | --- | --- |
| 谁管生命周期 | 你自己 | 引擎的 GC |
| 引擎能看懂布局吗 | 看不懂，只是字节 | 能——类型声明里写明了哪些字段是引用 |
| 能存进线性内存吗 | 本身就在里面 | **永远不能**（同 `externref`/`funcref` 规则） |
| 和 JS 对象同堆吗 | 不同 | **同一个堆** |

这也正是"Rust/C++ 不受影响"的原因：它们的对象在线性内存里，引擎看不见。

> **面试落点**：Wasm GC 让 **Kotlin/Wasm、Dart/Flutter Web、Java (TeaVM/CheerpJ)、
> Scala.js** 这类语言的 Web 产物体积和互操作性发生质变。
> 但注意：**Rust/C++ 不用 GC**——它们仍然用线性内存，GC proposal 对它们没影响。

## 值得关注的活跃 proposal

### Phase 5（已标准化，尚未并入 spec 主干）

- **JS Promise Integration (JSPI)** —— 让 Wasm 里的同步代码调 JS 异步 API，取代 Asyncify
- **Web Content Security Policy** —— `'wasm-unsafe-eval'` 等 CSP 关键字

### Phase 4

- **Threads** —— 共享内存 + 原子操作（浏览器早已实现）

### Phase 3

- **ESM Integration** —— `import` 一个 `.wasm` 文件，这是前端最期待的一个
- **Stack Switching** —— 通用的协程/续体能力，比 JSPI 更底层
- **Custom Page Sizes** —— 打破 64KiB 固定页大小（对 IoT/嵌入式很重要）
- **Wide Arithmetic**、**Compact Import Section**、**Custom Descriptors and JS Interop**

### Phase 2

- **FP16** —— 半精度浮点（机器学习推理场景）
- **Compilation Hints**、**Acquire-Release Atomics**、**Extended Name Section**、
  **JS Primitive Builtins**

### Phase 1（值得知道名字，但别当成现状）

- **Component Model** —— Wasm 的"接口 + 链接"标准，WASI 0.2+ 的基础
- **Shared-Everything Threads** —— 共享 table/global/GC 对象，让 GC 语言能真正多线程
- **Type Imports**、**Flexible Vectors**、**Memory Control**、**Stringref**、**Profiles**、
  **JIT Interface**

## Feature detection 怎么做

原理是：**构造一个最小模块，里面只用到目标特性的一条指令，然后 `WebAssembly.validate()` 它**。
引擎不支持该 proposal 就解不出这条指令，验证返回 `false`。

以 SIMD 为例，要检测的模块就是这么一小段：

```wat
(module
  (func (result v128)
    i32.const 0
    i8x16.splat))
```

把它 `wat2wasm` 成字节数组，再交给 `validate`：

```js
const bytes = new Uint8Array([/* wat2wasm 产出的字节 */]);
const simdSupported = WebAssembly.validate(bytes);
```

> **⚠️ 不要手写这个字节数组**。section size、body size 都是 LEB128 编码的长度前缀，
> 手算极易差一个字节；而且长度错了 `validate` 只会安静地返回 `false`——
> 看起来就像"引擎不支持"，排查起来非常费时。**用工具生成，或直接用库。**

社区库 **`wasm-feature-detect`** 把每个特性的检测模块都预先编译好了，直接用它：

```js
import { simd, threads, bigInt, exceptions, gc } from "wasm-feature-detect";
if (await simd()) { /* 加载 SIMD 版本 */ }
```

典型做法是**准备多个构建产物，运行时选择**：

```text
app.wasm            ← 基线
app.simd.wasm       ← 启用 SIMD
app.threads.wasm    ← 启用多线程（需 crossOriginIsolated）
```

官方 FeatureTest 文档里描述的正是这个场景：
开发者要在"支持矩阵爆炸"和"覆盖更多用户"之间做权衡。

> **面试落点**：Wasm 的 feature detection 是**用 `WebAssembly.validate()` 验证
> 一段最小模块能不能通过**——这比 `typeof` 检查可靠，因为它测的是引擎的实际解码能力。

## 相关页面

- 各特性的具体用法：[[wasm/types-and-abi]]、[[wasm/memory-model]]、[[wasm/js-interop]]
- 非 Web 方向（WASI / Component Model）：[[wasm/beyond-browser]]
- 性能相关特性（SIMD / Branch Hinting）：[[wasm/performance]]
