---
title: 线性内存模型
date: 2026-08-10
tags: [wasm, memory, 线性内存, 面试]
sources: [mdn-wasm/guides/using_the_javascript_api.md, mdn-wasm/guides/understanding_the_text_format.md, mdn-wasm/reference/memory.md, wasm-design/Rationale.md, wasm-design/Security.md, wasm-design/FAQ.md]
---

# 线性内存模型

## 基本形态

**Linear memory = 一段连续、可增长、无类型的字节数组**。模块内的 load/store 指令
可以访问其中任意字节——这是忠实表达 C/C++ 指针语义的必要条件。

关键约束：

- **页大小固定 64KiB**（不是 4KB！这是高频考点）
- 只能**增长**，不能缩小
- 创建时指定 `initial`，可选 `maximum`，单位都是**页**
- **始终小端序**（little-endian）
- 在 JS 里表现为 `ArrayBuffer`（共享内存则是 `SharedArrayBuffer`）

```js
const memory = new WebAssembly.Memory({ initial: 10, maximum: 100 });
// 初始 640KB，最大 6.4MB

const view = new DataView(memory.buffer);
view.setUint32(0, 42, true);   // 第三个参数 true = 小端，必须写
view.getUint32(0, true);       // 42
```

> **面试落点**：Wasm 页固定 64KiB，是因为它是**大量平台/CPU 页大小的最小公倍数**，
> 定死一个值才能保证程序在所有引擎上行为一致。引擎可以借助虚拟内存机制做边界检查，
> 前提就是内存大小页对齐。

## 为什么是"每个 instance 一块独立内存"

原生程序里可用内存跨整个进程地址空间；Wasm instance 能访问的**只有它自己那块 Memory**。
好处：一个 Web 应用可以同时用几十个内部使用 Wasm 的库，
**每个库有完全隔离的内存**，互不干扰。

这也解释了为什么 Wasm 的越界检查很便宜——不需要精细的对象级检查，
只需要在**整块线性内存的边界**上检查一次。

> **⚠️ 常见误答**："Wasm 内存安全，不会有缓冲区溢出" ——
> 精确说法是：**溢出打不出线性内存这个盒子**（会 trap），
> 但**盒子内部**的对象之间可以互相覆写。Wasm 消灭的是"越出沙箱"和"劫持控制流"，
> 消灭不了"程序自己的堆被写坏"。见 [[wasm/security-sandbox]]。

## grow 与 detach —— 最容易踩的坑

```js
memory.grow(1);   // 增长 1 页，返回增长前的页数
```

**`ArrayBuffer.byteLength` 是不可变的**，所以 grow 成功后：

- `memory.buffer` 会返回一个**全新的 ArrayBuffer**
- 所有旧的 ArrayBuffer 变成 **detached（分离）**状态
- **所有基于旧 buffer 建的 TypedArray 视图全部失效**（读会得到 0 或抛错）

这是实际项目里最常见的 Wasm bug：

```js
// ❌ 危险：缓存了视图
const heap = new Uint8Array(instance.exports.memory.buffer);
instance.exports.doSomethingThatAllocates();  // 内部触发 memory.grow
heap[0];  // 💥 heap 已经 detached

// ✅ 正确：每次用之前重新取
function u8() {
  return new Uint8Array(instance.exports.memory.buffer);
}
u8()[0];
```

Emscripten 生成的胶水代码里到处是"访问前刷新 `HEAP*` 视图"的逻辑，就是为了这个。
但如果你用 `--js-library` 之外的方式直接摸 `Module.HEAP*`，就要自己负责。

> **面试落点**：`memory.grow()` 会 **detach** 旧的 ArrayBuffer，
> 所有缓存的 TypedArray 视图必须重建。这是 Wasm 集成里最高频的线上 bug。

指定 `maximum` 的作用：引擎可以**预留虚拟地址空间**，之后 grow 只是"扩大边界"，
不需要重新分配 + 拷贝，因此更快。代价是初始占用更多虚拟地址空间。
如果超过 `maximum` 还 grow，抛 `RangeError`。

## 谁创建 memory：import 还是 export

两种方向都合法：

```wat
(import "js" "mem" (memory 1))     ;; JS 创建，Wasm 导入
(memory (export "memory") 1)       ;; Wasm 创建并导出
```

**import 方向的两个好处**（官方文档明确列出）：

1. JS 可以在模块编译**之前或同时**就准备好内存初始内容（省一次串行等待）。
2. **多个模块实例可以导入同一个 Memory** —— 这是实现动态链接的基础构件。

Rust/wasm-bindgen 默认是 Wasm 创建并导出；Emscripten 两种都用过，
现在默认也是 Wasm 侧创建。

## data 段与 bulk memory 指令

`data` 段在实例化时把一段字节写进内存的指定偏移，类似原生可执行文件的 `.data` 段。

Wasm 2.0 起有 **bulk memory operations**，让 `memcpy`/`memset` 这类操作一条指令搞定，
而不是编译成循环：

| 指令 | 作用 |
| --- | --- |
| `memory.copy` | 内存区域间拷贝（≈ memmove） |
| `memory.fill` | 用某字节填充一段区域（≈ memset） |
| `memory.init` | 从 passive data 段拷贝到内存 |
| `data.drop` | 丢弃 data 段（释放它占的资源） |
| `table.copy` / `table.init` / `elem.drop` | table 的对应操作 |
| `memory.size` / `memory.grow` | 查询 / 增长，单位是页 |

> **面试落点**：bulk memory 指令让引擎能用**本机 memcpy**（可能是 SIMD 优化过的）
> 实现大块拷贝，比展开成 Wasm 循环快一个量级。它也是**共享内存下线程安全初始化**的基础。

## 多内存（multi-memory，Wasm 3.0）

一个 instance 现在可以有多块内存，每块有独立索引，所有内存指令都能指定索引：

```wat
(module
  (import "js" "mem0" (memory 1))
  (import "js" "mem1" (memory 1))
  (memory $mem2 1)
  (export "memory2" (memory $mem2))

  (data (memory 0) (i32.const 0) "Memory 0 data")
  (data (memory 1) (i32.const 0) "Memory 1 data")
  (data (memory 2) (i32.const 0) "Memory 2 data")
  (data (i32.const 13) " (Default)"))   ;; 省略索引 = memory 0
```

用途：把公开数据和私有数据分开、把需要持久化的数据隔离、突破 32 位地址空间限制、
不同线程共享策略不同的内存分区。**老代码完全不受影响**——省略索引就是 memory 0。

## 共享内存（shared memory）

```js
const memory = new WebAssembly.Memory({ initial: 10, maximum: 100, shared: true });
memory.buffer;  // SharedArrayBuffer，不是 ArrayBuffer
```

```wat
(memory 1 2 shared)   ;; shared 内存必须显式指定 maximum
```

- `shared: true` 的内存可以通过 `postMessage()` 传给 Worker
- **必须指定 maximum**——因为共享内存 grow 时**不能移动**（否则其它线程持有的
  SharedArrayBuffer 全都失效），只能在预留区间内扩大边界
- 需要 **COOP/COEP 响应头**才能使用 SharedArrayBuffer，见 [[wasm/threads-and-workers]]

> **面试落点**：普通内存 grow 会 detach 并搬家；**共享内存 grow 保证原地扩容**，
> 已有的 SharedArrayBuffer 继续有效。代价就是必须提前声明 maximum、预留地址空间。

## 为什么 maximum 是可选的

官方 Rationale 列了四个互相冲突的约束：

1. 想在应用启动早期抢一大块连续内存（地址空间还没碎片化）
2. 一个页面可能有几十个库各自带 Wasm 模块，都要能跑在 32 位地址空间里
3. 不能强迫每个开发者都精确知道自己的堆峰值
4. 共享内存不能靠 realloc 实现增长

结论：**maximum 可选**。不写就让引擎按需保守分配；写了就换取更高效的 grow。
共享内存则强制要求写。

## mmap 怎么办

Wasm 没有 `mmap`，因为它把 mmap 那一堆重载功能**拆开**了：
增长用 `memory.grow`，页保护/映射等能力在 memory-control proposal（Phase 1）里。

唯一真正缺失的是**分配不连续的虚拟地址区间**。不提供的原因：
不连续分配意味着 Wasm 内存与宿主其它分配交错，既难做高效安全检查，
又会引入分配的不确定性——而 Wasm 一贯避免不确定性。
用户态 libc 完全可以用现有原语模拟出兼容的 `mmap`。

## 相关页面

- 类型与零拷贝视图：[[wasm/types-and-abi]]
- 多线程与 SharedArrayBuffer：[[wasm/threads-and-workers]]
- 安全边界：[[wasm/security-sandbox]]
- 性能影响：[[wasm/performance]]
