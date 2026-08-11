---
title: 性能 —— 为什么快、什么时候不快
date: 2026-08-10
tags: [wasm, 性能, 优化, 面试]
sources: [wasm-design/Rationale.md, wasm-design/FAQ.md, wasm-design/Portability.md, emscripten-doc/Optimizing-Code.rst, wasm-bindgen-doc/reference-optimize-size.md, mdn-wasm/reference/simd.md]
---

# 性能

这一页是面试的**主战场**。大多数候选人只会说"Wasm 比 JS 快"，而真正拉开差距的回答是
**"快在哪个环节、慢在哪个环节、怎么量"**。

## Wasm 为什么快：四个原因

### 1. 解码快，不是"执行快"

二进制格式 vs 文本源码，官方实验数据：**解码 Wasm 比解析等价 asm.js 快约 23 倍**，
gzip 后体积小 20–30%。原因很直接：

- 名字全部编码成**整数索引**，读取是数组下标而不是字典查找
- 结构规整，可以单遍线性解码
- 可以**边下载边解码**（streaming）

### 2. 类型静态确定，不需要推测优化

JS 引擎为了跑快，必须做 **类型推测 → 内联缓存 → 反优化（deopt）** 这一整套。
一次类型假设失败就得回退到解释器重跑。

Wasm 的类型在编译期就确定且验证过，引擎**直接生成机器码，没有反优化**。

> **面试落点**：Wasm 的性能优势不只是"更快"，更重要的是**性能可预测**——
> 没有 deopt、没有 GC 停顿（除非用 GC proposal）、没有隐藏类失效。
> 对游戏、音视频这类要求稳定帧率的场景，可预测性比峰值速度更值钱。

### 3. 结构化控制流 → 验证快、可流式编译

单遍线性验证，不需要不动点计算（对比早期 JVM 字节码验证），
所以能在下载过程中就编译。

### 4. 直接映射硬件能力

只有 i32/i64/f32/f64/v128，全是 CPU 原生类型；小端序、补码、IEEE754 都是硬性可移植性假设。
不需要在语义和硬件之间做昂贵的翻译。

## 但 Wasm 也有固有开销

| 开销来源 | 说明 |
| --- | --- |
| **跨边界调用** | 每次 JS ↔ Wasm 调用有固定成本（参数转换、栈切换） |
| **数据编解码** | 字符串、数组、对象都要序列化进线性内存或反序列化出来 |
| **`call_indirect` 双检查** | 边界检查 + 签名检查，虚函数密集的 C++ 代码受影响明显 |
| **边界检查** | 每次 load/store 理论上都要检查（引擎用虚拟内存技巧大幅优化，但不免费） |
| **无法直接访问 DOM** | 每个 DOM 操作都要绕道 JS |
| **i64 ↔ BigInt** | BigInt 装箱有真实成本 |
| **没有 JIT 反馈** | 引擎无法基于运行时 profile 做投机优化（JS 引擎可以） |

> **⚠️ 常见误答**："Wasm 比 JS 快 10 倍" —— 没有普适倍数。
> **纯计算内核**上 Wasm 通常是优化过的 JS 的 **1–3 倍**（有时 JS 更快，因为 JIT 拿到了运行时信息）。
> **字符串/对象密集**或**高频跨边界**的场景，Wasm 常常**更慢**。

## 启动成本拆解

Wasm 的"性能"包含两部分，面试时一定要分开讲：

```text
总时间 = 下载 + 编译 + 实例化 + 执行
         ↑      ↑       ↑        ↑
      体积决定  体积决定  imports  算法决定
                        数量决定
```

| 阶段 | 优化手段 |
| --- | --- |
| **下载** | brotli 压缩、CDN、代码分割、按需加载 |
| **编译** | `instantiateStreaming`（与下载重叠）、缩小体积、浏览器编译缓存 |
| **实例化** | 减少 import 数量、`-sEVAL_CTORS` 把初始化预跑成快照 |
| **执行** | SIMD、多线程、算法本身 |

引擎侧还有**分层编译**：先用快速基线编译器出可执行代码，
后台再用优化编译器重编译热函数并替换。以 V8 为例，基线是 **Liftoff**，优化层是 **TurboFan**；
Wasmtime 里对应的是 **Winch**（基线）和 **Cranelift**（优化）。

所以"第一次调用慢、之后变快"在 Wasm 里也存在，只是幅度远小于 JS——
因为基线编译器产出的已经是真机器码，而不是解释执行。

Branch Hinting（Wasm 3.0）就是给基线编译器提示分支概率的机制，让**第一层编译的代码就更好**。

## 跨边界调用成本：最关键的实践结论

```js
// ❌ 反模式：把 Wasm 当成一堆小函数用
for (let i = 0; i < 1_000_000; i++) {
  sum += wasm.add(arr[i], 1);       // 一百万次跨边界
}

// ✅ 正确：把整个循环搬进 Wasm，一次调用
const ptr = wasm.alloc(arr.length * 4);
new Int32Array(wasm.memory.buffer, ptr, arr.length).set(arr);
const sum = wasm.sumAll(ptr, arr.length);   // 一次跨边界
wasm.dealloc(ptr, arr.length * 4);
```

> **面试落点**：Wasm 优化的第一原则是 **"把边界画在粗粒度的地方"**——
> 传大块数据、做大块工作、返回大块结果。细粒度的 JS↔Wasm 往返会把所有性能优势吃掉。
> 现代引擎的跨边界调用已经优化得很好（可内联），但**数据编解码的成本消不掉**。

## SIMD

`v128` + 定宽 SIMD（Wasm 2.0 进 spec）提供 128 位向量运算：
算术、位运算、类型转换、提取（extract）、load/store。

典型收益：图像处理、音视频编解码、加密、数值计算上 **2–4 倍**。

限制：

- **只有 128 位定宽**。更宽的向量在 Flexible Vectors proposal（Phase 1）里
- `v128` **不能跨 JS 边界传递**，数据必须走线性内存
- **Relaxed SIMD**（Wasm 3.0）用**结果不确定性**换性能——同一指令在不同硬件上
  可能给出不同结果（比如 FMA 的舍入），换来直接映射本机指令

C/C++ 用法：`<wasm_simd128.h>`、LLVM vector extensions，或依赖自动向量化。
Rust 用 `core::arch::wasm32` 或 `packed_simd`。

> **面试落点**：Relaxed SIMD 是 Wasm 里**罕见的、故意引入不确定性**的设计——
> 因为强行统一各家硬件的浮点行为代价太大。它和 NaN 位模式不确定性是同一类权衡。

## 体积优化

体积同时影响**下载**和**编译**两个阶段，所以是启动性能的核心。

| 手段 | 典型收益 |
| --- | --- |
| `wasm-opt -Oz` | 10–30% |
| `-Os`/`-Oz` 编译 | 与上面叠加 |
| LTO | 跨编译单元内联和 DCE |
| `-fno-rtti -fno-exceptions`（C++） | 官方举例 Box2D **减少 15%** |
| `--closure 1`（Emscripten 胶水 JS） | 胶水 JS 大幅缩小 |
| strip name/debug section | 生产构建必做 |
| `panic = "abort"`（Rust） | 去掉 unwinding |
| `-sFILESYSTEM=0`、`-sENVIRONMENT=web` | 各省几 KB |
| brotli 传输压缩 | 通常再压到 1/3 |

分析工具：`twiggy`（Rust）、`wasm-objdump -h`、Emscripten 的 `--memoryprofiler`。

**测量的正确对象**：Rust 项目要测 `pkg/foo_bg.wasm`（wasm-bindgen 处理**后**），
不是 `target/wasm32-unknown-unknown/release/foo.wasm`（处理前，故意冗余）。

## Profiling

```bash
emcc -O2 --profiling file.cpp       # 保留函数名，可读的火焰图
```

浏览器 DevTools 的 Performance 面板能显示 Wasm 函数（前提是保留了 name section）。
Chrome DevTools 还能在 Wasm 帧上做采样。

**必须多浏览器测**——各家引擎的 Wasm 实现差异比 JS 大得多（官方 Emscripten 文档明确建议）。

## 一个务实的性能决策清单

判断"该不该上 Wasm"时按顺序问：

1. **热点确实是计算吗？** 先 profile。很多时候瓶颈在网络、渲染、布局，Wasm 帮不上。
2. **JS 版本优化到位了吗？** 用 TypedArray、避免装箱、避免多态——优化过的 JS 常常够用。
3. **数据能一次性大块传吗？** 如果必须频繁小批往返，收益会被边界成本吃掉。
4. **能容忍额外的构建复杂度和产物体积吗？** 一个几十 KB 的 Wasm 换 20% 提速，可能不划算。
5. **有现成的 C/C++/Rust 实现吗？** 移植成熟库（编解码器、压缩、CV）是 Wasm 最稳的收益场景。

完整框架见 [[analysis/when-to-use-wasm]]。

## 相关页面

- 内存与零拷贝：[[wasm/memory-model]]
- 多线程：[[wasm/threads-and-workers]]
- 构建与加载：[[wasm/build-integration]]
- 决策框架：[[analysis/when-to-use-wasm]]
