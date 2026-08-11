---
title: 浏览器之外 —— WASI 与 Component Model
date: 2026-08-10
tags: [wasm, wasi, component-model, 服务端]
sources: [wasi-doc/README.md, wasm-design/NonWeb.md, wasm-design/Portability.md, wasm-design/proposals-README.md, wasm-design/HighLevelGoals.md]
---

# 浏览器之外

**"支持非浏览器嵌入"是 Wasm High-Level Goals 的第 4 条**，不是事后补的。
理解这一点，才能回答"Wasm 除了浏览器还能干什么"。

## 核心机制：Wasm 不定义任何 API

Wasm 规范里**没有任何系统调用、没有任何 API**——只有 **import 机制**，
可用的 import 完全由**宿主（embedder）**决定。

```text
浏览器宿主   → 提供 JS 函数作为 import → 间接访问 DOM / fetch / WebGL
WASI 宿主    → 提供标准化的系统接口     → 文件、网络、时钟、随机数
自定义宿主   → 提供你自己的能力集       → 插件 API、业务 SDK
```

> **面试落点**：这个设计让 Wasm 天然是**能力安全**的：
> **你没导入的能力，模块就绝对拿不到**。一个只导入了 `add` 的模块，
> 无论代码怎么写都不可能读文件、发网络请求。这就是它适合跑不可信代码的根本原因。

## WASI（WebAssembly System Interface）

WASI 就是"给非浏览器宿主的一套标准 import 集合"。由 WebAssembly CG 下的 WASI 子组推进。

| 版本 | 别名 | 描述方式 | 状态 |
| --- | --- | --- | --- |
| **WASI 0.1** | Preview 1 | witx IDL | 广泛使用中，事实标准 |
| **WASI 0.2** | Preview 2 | **Wit IDL**，基于 Component Model | 模块化、类型系统更强、可虚拟化 |
| **WASI 0.3** | Preview 3 | Wit + Component Model 原生 async | 当前 preview |

WASI 的主要影响来源是 **POSIX 和 CloudABI**。

### 三代的演进逻辑

- **0.1**：一个大而全的扁平接口，像 POSIX 的子集。够用，但语言支持有限、无法拆分。
- **0.2**：拆成模块化的一组接口（`wasi:filesystem`、`wasi:sockets`、`wasi:clocks`、
  `wasi:http`……），用 Wit 描述。关键收益：
  - 支持更广的源语言（不只是 C 系）
  - **可虚拟化**——可以用另一个 Wasm 组件实现某个 WASI 接口，做假文件系统、假网络
  - 更强的类型系统（record、variant、resource）
- **0.3**：用 Component Model 原生的 `future` / `stream` 类型
  取代 0.2 里显式的 streams + polling 接口，异步变成语言级的。

> **面试落点**：WASI 0.2 相对 0.1 的质变是**模块化 + 可虚拟化**。
> "可以用一个 Wasm 组件去实现另一个组件依赖的 WASI 接口"——
> 这让沙箱能力可以被**精细裁剪和 mock**，是多租户平台的关键能力。

## Component Model

**注意：Component Model 目前是 Phase 1**，不要说成现状。

它想解决的是 core wasm 的一个根本限制：**core wasm 只能传数值和引用**。
两个 Wasm 模块想互相传字符串、record、list，必须约定内存布局——
而不同语言的约定不同，于是无法互操作。

Component Model 引入：

- **Wit IDL**：语言无关的接口描述（类似 protobuf/IDL，但描述的是函数接口）
- **Canonical ABI**：高层类型（string / list / record / variant / resource）
  如何 lift/lower 到 core wasm 的标准约定
- **组件组合**：组件之间可以直接连接，各自有独立的线性内存

理想状态：**用 Rust 写的组件和用 Go 写的组件直接互相调用，传 string 和 struct，
不需要任何胶水代码**。

```wit
// 一个 Wit 接口示例
interface image-processor {
  record dimensions { width: u32, height: u32 }
  resize: func(data: list<u8>, target: dimensions) -> result<list<u8>, string>;
}
```

## 主流运行时

| 运行时 | 出品方 | 特点 |
| --- | --- | --- |
| **Wasmtime** | Bytecode Alliance | 最主流的独立运行时，Cranelift 优化后端，WASI/Component Model 跟得最紧 |
| **Wasmer** | Wasmer | 多后端，WAPM 包管理，嵌入 API 丰富 |
| **WasmEdge** | CNCF | 面向云原生/边缘，AI 推理扩展 |
| **wazero** | Tetrate | **纯 Go 实现，零 CGO 依赖**，Go 项目里嵌 Wasm 的首选 |
| **wasm3 / WAMR** | — | 解释器，极小体积，IoT/嵌入式 |
| **V8 / SpiderMonkey / JSC** | 浏览器厂商 | 也可以脱离浏览器用（Node.js 就是 V8） |

> 规范意义上的**参考实现**是 `WebAssembly/spec` 仓库里的 OCaml 解释器，
> 它只用于验证规范本身，不是给生产用的。上表里的都是生产运行时。

## 典型非 Web 场景

### 1. 多租户 Serverless / 边缘计算

Fastly Compute@Edge、Cloudflare Workers、Shopify Functions。

卖点是**冷启动**：Wasm 实例微秒级启动，容器百毫秒级。
加上能力安全模型，可以在同一进程里安全地跑成千上万个租户的代码。

> ⚠️ "微秒级 vs 百毫秒级"是**厂商公开宣称的量级对比**（Fastly、Cloudflare 等），
> 不在 `raw/` 的源文档里，也高度依赖测量口径
> （只算实例化？还是含模块编译、WASI 初始化？）。
> 面试里说"量级差异"是安全的，别报具体数字。

### 2. 插件系统

Envoy/Istio 的 WasmPlugin、Zellij、Zed 编辑器扩展、各类数据库的 UDF。

**核心价值**：插件用任意语言写、宿主不用重编译、崩溃不影响宿主、能力可精确裁剪。

### 3. 可移植分发

一个 `.wasm` 跑在任何架构上，不用交叉编译矩阵。
对 CLI 工具、嵌入式设备固件更新有实际价值。

### 4. 区块链 / 智能合约

Wasm 的**确定性**和**可计量性**（可以精确统计执行了多少条指令 → gas）
让它成为多条链的合约执行引擎（Polkadot、NEAR、CosmWasm）。

> **面试落点**：智能合约选 Wasm 是因为它的**确定性设计**——
> 而 Wasm 里那几处不确定性（NaN 位模式、Relaxed SIMD、共享内存）
> 恰好是链上环境会**明确禁用**的。这也是 Profiles proposal（Phase 1）想标准化的事情：
> 定义不同的"配置档"，比如"确定性档"禁掉所有不确定来源。

## 可移植性的硬性假设

Wasm 要求宿主平台满足（官方 Portability 文档）：

- 8 位字节，按字节寻址
- 支持非对齐访问，或有可靠的陷阱机制供软件模拟
- 32 位（可选 64 位）补码有符号整数
- IEEE 754-2019 的 32/64 位浮点（少数例外）
- **小端序**
- 内存区域可用 32 位指针/索引高效寻址（wasm64 用 64 位）
- 能强制模块间的安全隔离
- 对所有执行线程有**前进保证（forward progress guarantee）**
- 8/16/32 位自然对齐的**无锁原子操作**（至少要有 CAS）

不满足的平台仍然可以跑 Wasm，但需要软件模拟，性能会差。

> **面试落点**：**Wasm 假设小端序**。大端平台跑 Wasm 需要在每次内存访问时做字节序转换，
> 性能代价明显。这是"Wasm 不是完全抽象的虚拟机，而是贴着主流硬件设计的虚拟 ISA"的又一证据。

## 与浏览器的关系

浏览器和 WASI 是**两个不同的 embedder**，不要混淆：

- 浏览器里**没有 WASI**。想在浏览器里跑 WASI 程序，需要一个 JS 实现的 WASI polyfill
  （如 `@bjorn3/browser_wasi_shim`、`@wasmer/wasi`），把文件系统等接口用 IndexedDB/内存模拟。
- Emscripten 生成的产物里能看到 `wasi_snapshot_preview1.fd_write` 这类 import——
  它内部就是用 WASI 0.1 的接口形态，再由胶水 JS 实现。

## 相关页面

- 安全与能力模型：[[wasm/security-sandbox]]
- proposal 阶段：[[wasm/proposals-and-versions]]
- 案例：[[wasm/use-cases]]
