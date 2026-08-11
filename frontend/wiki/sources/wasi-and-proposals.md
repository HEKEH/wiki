---
title: 源材料 —— WASI 与 Proposals 仓库
date: 2026-08-10
tags: [源材料, wasi, proposal]
sources: [wasi-doc/README.md, wasm-design/proposals-README.md, wasm-design/proposals-finished.md]
---

# 源材料：WASI 与 Proposals 仓库

**位置**：`raw/wasi-doc/README.md`、`raw/wasm-design/proposals-*.md`
**来源**：`github.com/WebAssembly/WASI`、`github.com/WebAssembly/proposals`
**抓取日期**：2026-08-10

## `proposals-finished.md` —— 全库时效性的锚点

这是**唯一一份能确定"现在到底有什么"的权威材料**，
记录了每个已完成 proposal 的：名称、champion、WG 通过日期、影响的规范、**spec 版本**。

最后一条记录是 **2025-07-23 的 WG 会议**（Exception handling、JS String Builtins、
Memory64 三项进入 3.0）。

按 spec 版本分组：

| 版本 | proposals |
| --- | --- |
| **1.0** | MVP、Import/Export of Mutable Globals |
| **2.0** | Non-trapping float-to-int、Sign-extension、**Multi-value**、**JS BigInt↔i64**、**Reference Types**、**Bulk memory**、**Fixed-width SIMD** |
| **3.0** | Tail call、Extended Const、Typed Function References、**GC**、**Multiple memories**、**Relaxed SIMD**、Custom Annotation Syntax、**Branch Hinting**、**Exception handling**、**JS String Builtins**、**Memory64** |

> **使用规则**：[[wasm/proposals-and-versions]] 里的版本表就是从这份文件直接整理的。
> 任何"Wasm 有没有 X"的问题，先查这里。

## `proposals-README.md` —— 活跃 proposal 的阶段名单

按 Phase 0–5 列出全部活跃提案。抓取时的快照：

- **Phase 5**：JS Promise Integration、Web Content Security Policy
- **Phase 4**：Threads
- **Phase 3**：ESM Integration、Wide Arithmetic、Stack Switching、
  Compact Import Section、Custom Page Sizes、Custom Descriptors and JS Interop
- **Phase 2**：Relaxed dead code validation、Numeric Values in WAT Data Segments、
  Extended Name Section、Rounding Variants、Compilation Hints、JS Primitive Builtins、
  Acquire-Release Atomics、Multibyte Array Access、FP16
- **Phase 1**：Type Imports、**Component Model**、Wasm C/C++ API、Flexible Vectors、
  Memory control、Reference-Typed Strings、Profiles、**Shared-Everything Threads**、
  Frozen Values、More Array Constructors、JIT Interface、Type Reflection、
  JS Text Encoding Builtins

⚠️ **这份名单会变**。引用前最好去 <https://webassembly.org/features/> 核对引擎支持情况——
proposal 进 spec 和引擎实现是两件事。

面试里最需要记住阶段的三个：

- **ESM Integration → Phase 3**（还不能 `import` 一个 `.wasm`）
- **Threads → Phase 4**（虽然浏览器早就实现了）
- **Component Model → Phase 1**（**别说成现状**）

## `wasi-doc/README.md`

只有 45 行，但把三代 WASI 的演进逻辑说清楚了：

| 版本 | 别名 | IDL | 关键变化 |
| --- | --- | --- | --- |
| WASI 0.1 | Preview 1 | witx | 主要影响来源是 **POSIX 和 CloudABI**，现在广泛使用 |
| WASI 0.2 | Preview 2 | **Wit** | 模块化、支持更广的源语言、更强类型系统、**可虚拟化** |
| WASI 0.3 | Preview 3 | Wit | 用 Component Model 原生的 `future`/`stream` 取代显式 streams + polling |

**关键理解**：WASI 0.2 起构建在 Component Model 之上——
而 **Component Model 本身还在 Phase 1**。这个"依赖关系 + 阶段落差"是理解
WASI 生态现状的钥匙：0.1 是事实标准，0.2 是方向，但基础规范尚未定稿。

## 抓取局限

- WASI 的具体接口定义分散在各自独立的仓库里（`wasi-filesystem`、`wasi-http`、
  `wasi-sockets` 等），本次没抓
- `proposals/inactive-proposals.md` 没抓（记录被搁置的提案，参考价值低）
- Component Model 的 Wit 规范没抓

需要深入时的入口：

- <https://github.com/WebAssembly/component-model/blob/main/design/mvp/WIT.md>
- <https://github.com/WebAssembly/WASI/blob/main/docs/Proposals.md>

## 覆盖的考点

- proposal 阶段与版本演进（三档深度题）
- 非 Web 场景的 WASI 基础
- ❌ 不覆盖：WASI 接口的具体用法、各运行时的差异

## 相关页面

- [[wasm/proposals-and-versions]]
- [[wasm/beyond-browser]]
- [[sources/wasm-design-repo]]
