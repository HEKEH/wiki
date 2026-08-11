---
title: 构建集成 —— Vite / Webpack / 加载与缓存
date: 2026-08-10
tags: [wasm, vite, webpack, 构建, 工程化]
sources: [mdn-wasm/guides/rust_to_wasm.md, mdn-wasm/guides/loading_and_running.md, wasm-bindgen-doc/reference-deployment.md]
---

# 构建集成

这是最"前端"的一页，也是面试里最容易被追问细节的一页——
因为它检验的是**你到底有没有在真实项目里集成过 Wasm**。

## 核心矛盾

Wasm **还不是原生 ES module**（ESM Integration 仍在 Phase 3），
所以打包器必须补齐这一层。各家补法不同，这就是集成麻烦的根源。

## Vite

Vite 对 Wasm 有内建支持：

```js
// 方式一：拿到一个初始化函数（推荐）
import init from "./my_module.wasm?init";
const instance = await init({ imports: { /* ... */ } });
instance.exports.foo();

// 方式二：只要 URL，自己 fetch
import wasmUrl from "./my_module.wasm?url";
const { instance } = await WebAssembly.instantiateStreaming(fetch(wasmUrl));
```

配 `wasm-pack --target web` 的产物时，直接 import 生成的 JS 胶水即可：

```js
import init, { greet } from "./pkg/hello_wasm.js";
await init();
greet("Vite");
```

用 `wasm-pack --target bundler` 产物时，需要 `vite-plugin-wasm` +
`vite-plugin-top-level-await`（因为 bundler target 的胶水依赖顶层 await）。

**dev 与 build 行为差异**：Vite dev server 走原生 ESM，
`.wasm` 直接由 server 提供且带正确 MIME；build 后走 Rollup 的 asset 流程。

⚠️ **构建后一定要打开 `dist/` 确认 `.wasm` 是独立文件**，不是被内联成了 base64 data URI。
一旦被内联，`instantiateStreaming` 的流式路径就没了，而且 base64 会让体积涨约 1/3。
各打包器对"小资源内联"的阈值和是否豁免 `.wasm` 的策略**不一致、且随版本变**
（Vite 的 `build.assetsInlineLimit`、Webpack 的 `asset/inline`），
所以这条靠查文档不如**直接看产物**。

## Webpack 5

```js
// webpack.config.js
module.exports = {
  experiments: {
    asyncWebAssembly: true,     // 支持 import wasm（异步）
    // syncWebAssembly: true,   // 旧行为，不推荐
  },
};
```

开了 `asyncWebAssembly` 之后，`import * as wasm from "hello-wasm"` 就能直接用，
Webpack 会自动处理加载和实例化。这也是 `wasm-pack --target bundler` 的目标环境。

Rust + npm 包的完整链路：

```bash
wasm-pack build --target bundler     # 产出 pkg/
cd site && npm i ../pkg              # 本地安装
```

```js
import * as wasm from "hello-wasm";
wasm.greet("WebAssembly with npm");   // 用起来跟普通 npm 包无异
```

## 服务器配置（最容易被忽略）

```nginx
# 1. MIME 类型——不设置流式编译会失败
types { application/wasm wasm; }

# 2. 压缩——wasm 压缩率很高，必开
gzip_types application/wasm;
# 更好：预压缩 .wasm.br 用 brotli

# 3. 长缓存——文件名带 hash 时
location ~* \.wasm$ {
  add_header Cache-Control "public, max-age=31536000, immutable";
}
```

> **面试落点**：`instantiateStreaming` **要求响应的 `Content-Type` 必须是
> `application/wasm`**，否则会直接失败（Chrome 会退化并警告）。
> 这是自建静态服务时最常见的坑。

## 加载策略

```js
// ❌ 阻塞首屏：入口就 await 一个大 wasm
import init from "./heavy.js";
await init();

// ✅ 按需加载：真正要用时才拉
let wasmReady;
function ensureWasm() {
  wasmReady ??= import("./heavy.js").then((m) => m.default());
  return wasmReady;
}
button.onclick = async () => {
  await ensureWasm();
  doHeavyWork();
};

// ✅ 空闲预热：不阻塞首屏但提前编译
requestIdleCallback(() => ensureWasm());
```

三种时机各有适用场景：

| 策略 | 适用 |
| --- | --- |
| 立即加载 | Wasm 就是应用主体（游戏、CAD、设计工具） |
| 交互时加载 | Wasm 是可选功能（导出 PDF、图像滤镜） |
| 空闲预热 | 大概率会用但不在首屏关键路径 |

## 编译缓存

浏览器会**自动缓存编译后的 Wasm 代码**（Chrome/Firefox 都有实现），
但触发条件比较苛刻：

- 必须走 `compileStreaming`/`instantiateStreaming`
- 需要 HTTP 缓存头允许（相同 URL + 有效缓存）
- 模块要足够大才值得缓存

不能依赖它做架构决策，但**这是"用 `instantiateStreaming` 而不是手动 fetch+compile"
的又一个理由**。

想要可控的缓存，可以自己把 `WebAssembly.Module` 存进 **IndexedDB**
（Module 是可结构化克隆的），但要自己管失效——实践中收益通常不如直接依赖 HTTP 缓存 + CDN。

## CSP

Wasm 的编译在 CSP 里受 `script-src` 管辖。严格 CSP 环境下可能需要：

```text
Content-Security-Policy: script-src 'self' 'wasm-unsafe-eval'
```

`'wasm-unsafe-eval'` 是专门为 Wasm 加的关键字——它允许编译 Wasm
但**不允许 `eval()` JS**，比笼统地开 `'unsafe-eval'` 安全得多。
对应的 Web Content Security Policy proposal 目前在 Phase 5。

## 多线程需要的响应头

用 SharedArrayBuffer（多线程 Wasm 必需）必须同时设：

```text
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

**代价很大**：开了 COEP 之后，所有跨域子资源（图片、iframe、第三方脚本）
都必须显式带 `Cross-Origin-Resource-Policy` 或走 CORS，否则被拦。
很多项目就是因为这个放弃了多线程 Wasm。详见 [[wasm/threads-and-workers]]。

## TypeScript

`wasm-pack` 会生成 `.d.ts`，直接可用。手写 Wasm 的话需要自己声明：

```ts
interface MyWasmExports {
  memory: WebAssembly.Memory;
  add(a: number, b: number): number;
  alloc(size: number): number;
}

const { instance } = await WebAssembly.instantiateStreaming(fetch("app.wasm"));
const exports = instance.exports as unknown as MyWasmExports;
```

`WebAssembly.Instance["exports"]` 的官方类型是 `WebAssembly.Exports`
（`Record<string, ExportValue>`），必须断言才能用。

## 产物体积治理

```bash
# 分析 Rust wasm 里谁占体积
twiggy top pkg/foo_bg.wasm

# 查看 section 分布
wasm-objdump -h app.wasm

# 手动再优化一遍
wasm-opt -Oz app.wasm -o app.min.wasm
```

**别忘了 strip 掉 name section**（调试符号）——生产构建里它可能占很大比例。
`wasm-opt --strip-debug --strip-producers` 或者 wasm-pack 的 release 模式会处理。

> **面试落点**：Wasm 的**传输体积**要看 gzip/brotli 后的大小，
> Wasm 的压缩率通常很好（二进制格式规整）。但**编译时间**和原始体积成正比，
> 所以"压缩后小"不等于"启动快"，两者都要看。

## 相关页面

- Rust 工具链：[[wasm/toolchain-rust]]
- Emscripten：[[wasm/toolchain-emscripten]]
- 加载与编译的性能分析：[[wasm/performance]]
- 多线程约束：[[wasm/threads-and-workers]]
