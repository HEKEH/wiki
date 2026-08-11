---
title: JS 互操作 —— JavaScript API 全貌
date: 2026-08-10
tags: [wasm, javascript, api, 互操作, 面试]
sources: [mdn-wasm/reference/javascript_interface.md, mdn-wasm/guides/loading_and_running.md, mdn-wasm/guides/using_the_javascript_api.md, mdn-wasm/guides/exported_functions.md, mdn-wasm/guides/imported_string_constants.md]
---

# JS 互操作

`WebAssembly` 是一个**命名空间对象**（像 `Math`、`Intl`），不是构造函数。

## API 全景

| 分类 | 成员 |
| --- | --- |
| 构造器 | `Module` `Instance` `Memory` `Table` `Global` `Tag` `Exception` `Suspending` |
| 静态方法 | `compile` `compileStreaming` `instantiate` `instantiateStreaming` `validate` `promising` |
| 静态属性 | `JSTag` |
| 错误类型 | `CompileError` `LinkError` `RuntimeError` |

## 四种加载方式，选哪个

**✅ 首选：流式实例化**——边下载边编译，一步到位。

```js
const { instance, module } = await WebAssembly.instantiateStreaming(
  fetch("app.wasm"),
  importObject,
);
```

**只编译不实例化**——要缓存 Module、或要多次实例化时用。

```js
const mod = await WebAssembly.compileStreaming(fetch("app.wasm"));
const inst = await WebAssembly.instantiate(mod, importObject);
```

**❌ 非流式**——多一次 ArrayBuffer 中转，白白等下载完才开始编译。

```js
const bytes = await fetch("app.wasm").then((r) => r.arrayBuffer());
const { instance } = await WebAssembly.instantiate(bytes, importObject);
```

**同步版本**——会阻塞当前线程，只在 Worker / Node 里、且模块很小时才可接受。

```js
const inst = new WebAssembly.Instance(new WebAssembly.Module(bytes), importObject);
```

> **面试落点**：`instantiateStreaming` 之所以更快，是因为它在**网络字节流上直接编译**，
> 下载和编译**重叠进行**，省掉了"等下载完 → 转 ArrayBuffer → 再编译"的串行等待。
> 前提是服务器返回 `Content-Type: application/wasm`，否则浏览器会拒绝流式路径。

### `instantiate` 的两个重载

```js
// 重载 1：传字节 → 得到 { module, instance }
WebAssembly.instantiate(bytes, imports).then(({ module, instance }) => {});

// 重载 2：传已编译的 Module → 直接得到 instance
WebAssembly.instantiate(module, imports).then((instance) => {});
```

返回值形状不同，是个小陷阱。**重载 1 返回对象，重载 2 直接返回 instance。**

## Module 可以 postMessage

`Module` 无状态，可以像 `Blob` 一样跨 Window/Worker 传递：

```js
// 主线程：编译一次
const module = await WebAssembly.compileStreaming(fetch("app.wasm"));
worker.postMessage({ module });   // 结构化克隆，不会重新编译

// Worker 里：直接实例化
self.onmessage = async ({ data }) => {
  const instance = await WebAssembly.instantiate(data.module, imports);
};
```

> **面试落点**：**编译一次，N 个 Worker 复用同一个 Module**，是多线程 Wasm 最重要的
> 启动优化。每个 Worker 单独 `instantiateStreaming` 会重复付编译成本。

## importObject 的结构

Wasm 的 import 是**两级命名空间**，importObject 就是两层嵌套对象：

```wat
(import "console" "log" (func $log (param i32)))
(import "js"      "mem" (memory 1))
```

```js
const importObject = {
  console: { log: (arg) => console.log(arg) },
  js: { mem: new WebAssembly.Memory({ initial: 1 }) },
};
```

缺任何一项、或类型对不上，实例化时抛 **`LinkError`**。

可以导入的东西：**函数、Memory、Table、Global**。

## 从 JS 读写 Wasm 内存

```js
const { memory } = instance.exports;

// 数值
const dv = new DataView(memory.buffer);
dv.setUint32(0, 42, true);   // true = 小端，Wasm 内存永远小端

// 字节数组（零拷贝视图）
const view = new Uint8Array(memory.buffer, ptr, len);

// 字符串
const str = new TextDecoder("utf-8").decode(view);
```

注意 `DataView` / `TypedArray` 建在 `memory.buffer` 上，不是 `memory` 上。
以及 grow 之后视图会失效，见 [[wasm/memory-model]]。

## Table 与动态链接

`WebAssembly.Table` 是引用的可增长数组，JS 侧可以 `get`/`set`/`grow`：

```js
const tbl = instance.exports.tbl;
tbl.get(0)();              // 取出函数引用并调用
tbl.set(0, otherFunction); // 替换——运行时改变间接调用目标
```

**动态链接**就是让多个 instance 共享同一个 Memory + Table：

```js
const importObj = {
  js: {
    memory: new WebAssembly.Memory({ initial: 1 }),
    table: new WebAssembly.Table({ initial: 1, element: "anyfunc" }),
  },
};

const [a, b] = await Promise.all([
  WebAssembly.instantiateStreaming(fetch("shared0.wasm"), importObj),
  WebAssembly.instantiateStreaming(fetch("shared1.wasm"), importObj),
]);
// shared1 里 call_indirect 调到的是 shared0 放进 table 的函数
b.instance.exports.doIt();
```

这就是 Wasm 版的 `.dll`/`.so`：多个模块共用同一个"地址空间"。
`dlopen` = 编译实例化新模块并存进宿主的表；`dlsym` = 找到导出、追加进函数表、返回索引
（C 函数指针在 Wasm 里本来就是 table 索引，天然对上）。

> **面试落点**：Wasm 的动态链接不是语言特性，而是**宿主能力 + import/export 共享
> memory/table 的组合**。因为 table 可变，所以能实现 interposition、weak symbol 这类高级特性。

## JS Promise Integration（JSPI）

这是 Phase 5（已标准化，待并入 spec）的重要 proposal，解决一个真实痛点：
**Wasm 里的同步代码（比如移植过来的 C 库里的 `fread`）想调 JS 的异步 API**。

传统方案是 Emscripten 的 **Asyncify**——把整个 Wasm 改写成可暂停/恢复的状态机，
体积膨胀严重（常见 +50%~100%）、性能有损。JSPI 让引擎原生支持挂起：

```js
// JS 侧的异步函数包装成"可挂起的导入"
const importObject = {
  env: { readFile: new WebAssembly.Suspending(async (path) => { /* ... */ }) },
};

// Wasm 侧看起来是同步调用；导出函数包装后对 JS 返回 Promise
const asyncMain = WebAssembly.promising(instance.exports.main);
await asyncMain();
```

> **面试落点**：JSPI 让"Wasm 里的同步代码调 JS 异步 API"变成引擎原生能力，
> 替代 Asyncify 这种**整体改写字节码**的方案。能说出"Asyncify 靠 CPS 变换实现，
> 体积和性能代价大"就很到位了。

## 异常处理（Wasm 3.0）

```js
WebAssembly.Tag        // 一种异常类型
WebAssembly.Exception  // 一个可跨 Wasm/JS 边界抛接的异常实例
WebAssembly.JSTag      // 内建 tag，让 Wasm 能捕获 JS 抛出的异常
```

Wasm 侧指令：`throw` / `throw_ref` / `try_table` + `catch` / `catch_all` /
`catch_ref` / `catch_all_ref`，异常值的类型是 `exnref`。

> **面试落点**：在原生异常处理进 spec 之前，C++ 异常在 Wasm 上要靠 JS 的
> try/catch 模拟，开销大到 Emscripten 默认在 `-O1` 以上**直接关掉 catch 块生成**。
> Wasm 3.0 的异常处理就是来消灭这个历史包袱的。

## ESM 集成的现状

**Wasm 目前还不能直接 `import` 一个 `.wasm` 文件**——`<script type="module">` 和
`import` 语句都还不支持 Wasm 模块。ESM Integration proposal 处于 **Phase 3**。

配套的还有 **source phase imports**（TC39 提案），能拿到"未实例化的 Module"：

```js
import source wasmModule from "./module.wasm";
const instance = new WebAssembly.Instance(wasmModule, imports);
```

目前只有 esbuild 和 Node.js 24+ 支持。wasm-bindgen 的 `--target module` 已经在用它了。

在此之前，实践上都靠打包器（Vite/Webpack）把 `.wasm` 处理成 URL 或内联，
再用 `instantiateStreaming` 加载。见 [[wasm/build-integration]]。

## 相关页面

- 类型映射细节：[[wasm/types-and-abi]]
- 内存读写与陷阱：[[wasm/memory-model]]
- 打包器集成：[[wasm/build-integration]]
- proposal 阶段说明：[[wasm/proposals-and-versions]]
