---
title: Rust 工具链 —— wasm-bindgen / wasm-pack
date: 2026-08-10
tags: [wasm, rust, wasm-bindgen, wasm-pack, 工具链]
sources: [mdn-wasm/guides/rust_to_wasm.md, wasm-bindgen-doc/introduction.md, wasm-bindgen-doc/reference-deployment.md, wasm-bindgen-doc/reference-optimize-size.md, wasm-bindgen-doc/reference-js-promises-and-rust-futures.md, wasm-bindgen-doc/reference-types-jsvalue.md, wasm-bindgen-doc/wasm-pack-commands-build.md]
---

# Rust 工具链

Rust 是目前**前端集成体验最好**的 Wasm 源语言：没有运行时、没有 GC、
`wasm-bindgen` 生成的胶水代码质量高、产物小。

## 三层工具的分工

```text
rustc  --target wasm32-unknown-unknown   → 原始 .wasm（体积大、只能传数值）
  ↓
wasm-bindgen CLI                          → 剥离冗余 + 生成 JS 胶水 + .d.ts
  ↓
wasm-pack                                 → 串起上面两步 + 生成 package.json + wasm-opt
```

**记住这个分层**，面试问"wasm-pack 和 wasm-bindgen 什么关系"时能答清楚：
`wasm-bindgen` 是 crate（宏）+ CLI（后处理器）；`wasm-pack` 是把整条流水线包起来的
npm 包发布工具。

## 最小示例

```rust
// src/lib.rs
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
extern "C" {
    pub fn alert(s: &str);          // 从 JS 导入
}

#[wasm_bindgen]
pub fn greet(name: &str) {          // 导出给 JS
    alert(&format!("Hello, {}!", name));
}
```

```toml
# Cargo.toml
[lib]
crate-type = ["cdylib"]     # 必须！否则不产出 .wasm

[dependencies]
wasm-bindgen = "0.2"
```

```bash
wasm-pack build --target web
```

`wasm-pack build` 做的五件事：

1. 编译 Rust 到 Wasm
2. 跑 `wasm-bindgen` 生成 JS 包装
3. 建 `pkg/` 目录放产物
4. 从 `Cargo.toml` 生成 `package.json`
5. 拷贝 README

产物：

```plain
pkg/
├── hello_wasm.js          ← JS 胶水（import 这个）
├── hello_wasm.d.ts        ← TS 类型
├── hello_wasm_bg.wasm     ← 真正的 wasm
├── hello_wasm_bg.wasm.d.ts
└── package.json
```

```html
<script type="module">
  import init, { greet } from "./pkg/hello_wasm.js";
  await init();          // 加载并实例化 wasm
  greet("WebAssembly");
</script>
```

## `--target` 怎么选（高频实操题）

| target | 输出形态 | 用在哪 |
| --- | --- | --- |
| `bundler`（默认） | 把 wasm 当作原生 ES module 引用 | Webpack / 需要打包器处理 |
| `web` | 可直接被浏览器 `<script type="module">` 加载，需手动 `init()` | 无打包器、Vite 也常用 |
| `no-modules` | 同上但不用 ES module（老式全局变量） | 老项目、不支持 module 的环境 |
| `nodejs` | CommonJS，`require()` 即用 | Node.js 服务端 |
| `experimental-nodejs-module` | Node ESM，`import` 即用 | Node 12+ ESM 项目 |
| `deno` | Deno 模块 | Deno |
| `module` | 用 **source phase imports** 拿未实例化的 Module | 新式跨工具链方案（esbuild / Node 24+） |

> **面试落点**：默认的 `bundler` target 产物**不能直接跑在浏览器里**——它假设
> "wasm 是原生 ES module"，而这个能力（ESM Integration）还没标准化，必须靠打包器补齐。
> 不用打包器就得选 `--target web`。这是新手最常踩的坑。

`--target web` 的限制：**不能用 npm 依赖**（因为没有打包器解析），也没有 polyfill。

## wasm-bindgen 到底做了什么

核心问题：Wasm 的 ABI 只能传数值和引用，而你写的是 `fn greet(name: &str)`。
`#[wasm_bindgen]` 宏在编译期生成：

- Wasm 侧一个"真实签名"函数，参数变成 `(ptr: i32, len: i32)`
- 一段描述元数据，塞进 wasm 的 custom section
- CLI 后处理时读这些元数据，**生成 JS 胶水**：

```js
// 生成的胶水做的事（简化）
export function greet(name) {
  const ptr = passStringToWasm(name);   // TextEncoder 编码 → __wbindgen_malloc → 写内存
  const len = WASM_VECTOR_LEN;
  wasm.greet(ptr, len);                 // 调真实的 wasm 函数
}
```

反方向（Wasm 调 JS）则生成一张导入表，把 `alert` 之类的函数注入进去。

> **面试落点**：wasm-bindgen 的本质是**双向胶水代码生成器**——
> 用 proc-macro 在编译期记录类型信息到 custom section，
> CLI 在链接后读取并生成对应的 JS 编解码代码。它不是运行时库，是构建期工具。

### 关于测量体积

**不要测 `target/wasm32-unknown-unknown/release/foo.wasm`**——
那是 wasm-bindgen CLI 处理**之前**的产物，故意包含冗余，CLI 会剥掉。
要测的是 `pkg/foo_bg.wasm`。生成的 `foo.js` 未压缩未混淆，交给你的打包器处理。

## JsValue 与类型映射

`JsValue` 表示"任意 JS 值"，是不透明句柄：

```rust
use wasm_bindgen::JsValue;

let values: Vec<JsValue> = vec![1.into(), "hello".into(), true.into()];
// &[JsValue] 传给 JS 时表现为 Array
```

配套两个 crate：

- **`js-sys`**：JS 标准库绑定（`Array`、`Promise`、`Map`、`Date`、`Reflect`……）
- **`web-sys`**：Web API 绑定（`window`、`Document`、`HtmlCanvasElement`、`fetch`、
  `WebGL`、`WebSocket`……），按 feature 开关，**只编进你用到的部分**

## async 互操作

`js_sys::Promise` 实现了 `IntoFuture`，可以直接 `.await`：

```rust
async fn get_from_js() -> Result<JsValue, JsValue> {
    let promise = js_sys::Promise::resolve(&42.into());
    let result = promise.await?;    // 成功 → Ok，reject → Err
    Ok(result)
}
```

导入 JS 的 async 函数：

```rust
#[wasm_bindgen]
extern "C" {
    #[wasm_bindgen(catch)]
    async fn fetchData() -> Result<JsValue, JsValue>;
}
```

导出 Rust 的 async 函数（JS 侧拿到 Promise）：

```rust
#[wasm_bindgen]
pub async fn foo() -> Result<JsValue, JsValue> { /* ... */ }
```

```js
const result = await foo();   // Ok → resolve，Err → reject（相当于 throw）
```

批量并发时，**如果输入本来就是 JS Promise，用 `Promise::all_iterable` 等原生组合子
比 Rust 的 `join_all` 更好**——避免 Rust executor 每次唤醒都轮询所有子 future。

其它常用：`spawn_local`（在 JS 微任务队列上跑一个 `Future<Output=()>`）、
`future_to_promise`（Rust Future → JS Promise）。

⚠️ **import 路径按版本不同**：这两个函数现在的实现在 `js_sys::futures` 里，
但一直从 `wasm_bindgen_futures` re-export（该 crate 已变成薄壳）。
稳妥写法是 `use wasm_bindgen_futures::spawn_local;`——新旧版本都能编过；
`js_sys::futures::spawn_local` 只在较新的 wasm-bindgen 上存在。

> **面试落点**：Rust 的 async 在 Wasm 上**没有自己的 runtime**（不是 tokio），
> 它复用 **JS 的事件循环和微任务队列**——`spawn_local` 把 future 挂到微任务上驱动。
> 这也意味着 Wasm 里的 `.await` 会真正让出到 JS 事件循环。

## 体积优化清单

```toml
[profile.release]
opt-level = "z"        # 或 "s"；"z" 更激进地压体积
lto = true             # 跨 crate 内联和死代码消除
codegen-units = 1      # 牺牲编译速度换优化质量
panic = "abort"        # 去掉 unwinding 机制（省不少）
strip = true
```

```bash
wasm-pack build --release
wasm-opt -Oz pkg/foo_bg.wasm -o pkg/foo_bg.wasm   # binaryen，wasm-pack 可自动调用
```

其它手段：

- 换更小的分配器（省几 KB，代价是分配性能差）。
  ⚠️ **不要再用 `wee_alloc`**——它已被 rustwasm 归档、不再维护，且有未修复的内存泄漏问题。
  现在的选择是 `lol_alloc` / `talc`，或者干脆用默认的（默认分配器本身也不算大）
- 用 `twiggy` 分析产物里谁占体积最大
- 避免 `format!`/`panic!` 的格式化机制被链进来（它会拖进一大坨 `core::fmt`）
- `console_error_panic_hook` 只在 debug 构建里启用

## 常见坑

> **⚠️ 常见误答**："Rust 编译到 Wasm 要 GC" —— 不需要。Rust 没有 GC，
> 所有权系统在编译期就解决了内存管理，Wasm 产物里只有一个分配器。
> 这正是 Rust 在 Wasm 生态占优的核心原因之一。

- **忘了 `crate-type = ["cdylib"]`** → 编不出 `.wasm`
- **在 `--target web` 下忘了 `await init()`** → 调用导出函数时报 wasm 未初始化
- **Rust 的 panic 在 Wasm 里表现为 `unreachable` trap**，堆栈信息很糟糕，
  开发期务必上 `console_error_panic_hook`
- **多线程需要额外配置**：`wasm-bindgen-rayon` + nightly + `atomics` target feature +
  COOP/COEP，见 [[wasm/threads-and-workers]]

## 相关页面

- C/C++ 工具链对比：[[wasm/toolchain-emscripten]]
- 打包器集成：[[wasm/build-integration]]
- ABI 与类型映射：[[wasm/types-and-abi]]
- 体积与性能：[[wasm/performance]]
