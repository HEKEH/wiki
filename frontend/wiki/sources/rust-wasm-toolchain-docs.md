---
title: 源材料 —— wasm-bindgen / wasm-pack 文档
date: 2026-08-10
tags: [源材料, rust, wasm-bindgen]
sources: [wasm-bindgen-doc/introduction.md, wasm-bindgen-doc/reference-deployment.md, wasm-bindgen-doc/reference-optimize-size.md, wasm-bindgen-doc/reference-types.md, wasm-bindgen-doc/reference-js-promises-and-rust-futures.md, wasm-bindgen-doc/wasm-pack-commands-build.md]
---

# 源材料：wasm-bindgen / wasm-pack 文档

**位置**：`raw/wasm-bindgen-doc/`（20 个文件）
**来源**：`github.com/rustwasm/wasm-bindgen` 的 `guide/src/`
+ `github.com/rustwasm/wasm-pack` 的 `docs/src/`
**抓取日期**：2026-08-10

## 这份材料是什么

Rust → Wasm 工具链的官方指南，滚动更新，**时效性最好的一份源材料**。

抓取的是 reference 部分（不含大量 examples），侧重"机制"而不是"教程"。

## 最有价值的几篇

### `reference-deployment.md`

**`--target` 的完整对照表**：`bundler` / `web` / `no-modules` / `nodejs` /
`experimental-nodejs-module` / `deno` / `module`。

关键信息：**默认的 `bundler` target 假设 "wasm 是原生 ES module"，
而这个能力任何 JS 实现都还没原生支持**，所以必须配打包器。
这解释了新手最常踩的坑。

还提到了 `--target module` 用 **source phase imports**
（`import source wasmModule from "./module.wasm"`）——目前只有 esbuild 和 Node.js 24+ 支持。

### `reference-optimize-size.md`

**关键纠正**：不要测 `target/wasm32-unknown-unknown/release/foo.wasm`——
那是 wasm-bindgen CLI 处理**之前**的产物，故意包含冗余。
要测 `pkg/foo_bg.wasm`。

以及：生成的 `foo.js` 未压缩未混淆，**期望你用自己的打包器去处理它**。

### `reference-js-promises-and-rust-futures.md`

async 互操作的完整说明：

- `js_sys::Promise` 实现 `IntoFuture`，可以直接 `.await`
- 导入 JS async 函数（`async fn` + `#[wasm_bindgen(catch)]`）
- 导出 Rust `async fn`（JS 侧得到 Promise），以及允许的返回类型
- `spawn_local` / `future_to_promise`
- **重要实践建议**：输入本来就是 JS Promise 时，
  用 `Promise::all_iterable` 等**原生组合子**比 Rust 的 `join_all` 更好——
  避免 Rust executor 每次唤醒都轮询所有子 future
- `wasm-bindgen-futures` 现在只是 `js-sys` 的 re-export shim

### `reference-types*.md`

`JsValue`（任意 JS 值的不透明句柄）、导出 Rust 类型、`js-sys` 的
**可擦除泛型**（`Array<T>` / `Promise<T>` / `Map<K,V>`）。

## 其余文件

`introduction.md`、`reference-index.md`、`reference-js-snippets.md`、
`reference-passing-rust-closures-to-js.md`、`reference-attributes-index.md`、
`reference-weak-references.md`、`reference-reference-types.md`、
`reference-browser-support.md`、`reference-debug-info.md`、
以及 3 个 wasm-pack 文档（`build` 命令、`pack-and-publish`、npm 教程）。

## 时效性

✅ **最新**。这份文档反映的是当前主线：

- `js_sys::futures` 已经吸收了 `wasm-bindgen-futures`
- 可擦除泛型（`Promise<T>`）是较新的能力
- `--target module` + source phase imports 是很新的方向

⚠️ 少数文件里有 mdBook 的 `{{#include ...}}` 占位符（如 `reference-types-jsvalue.md`），
raw 抓取时没有展开，所以看不到实际示例代码。

## 覆盖的考点

- 二档的 wasm-bindgen 原理、target 选择、体积测量
- Rust async 与 JS 事件循环的关系
- ❌ 不覆盖：Wasm 本身的语义、C/C++ 路线、多线程实操

## 局限

**只讲 Rust 路线**。有既有 C/C++ 代码库的场景要看 [[sources/emscripten-docs]]。

另外这份文档假设你会 Rust——不会 Rust 的话，
[[wasm/toolchain-rust]] 里的三层工具分工图和 `--target` 表是最需要记的部分。

## 相关页面

- [[wasm/toolchain-rust]]
- [[sources/emscripten-docs]]
- [[wasm/build-integration]]
