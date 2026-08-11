---
title: 源材料 —— Emscripten 文档
date: 2026-08-10
tags: [源材料, emscripten, c, cpp]
sources: [emscripten-doc/Interacting-with-code.rst, emscripten-doc/Optimizing-Code.rst, emscripten-doc/pthreads.rst, emscripten-doc/emscripten-runtime-environment.rst]
---

# 源材料：Emscripten 文档

**位置**：`raw/emscripten-doc/`（5 个 `.rst` 文件）
**来源**：`github.com/emscripten-core/emscripten` 的 `site/source/docs/`
**抓取日期**：2026-08-10

## 抓了哪几篇

只抓了对面试和实践最关键的四篇（原站文档量极大，全抓没有意义）：

| 文件 | 内容 |
| --- | --- |
| `Interacting-with-code.rst`（854 行） | JS↔C 的**全部四种调用方向** |
| `Optimizing-Code.rst`（271 行） | 优化等级、体积治理、LTO、EVAL_CTORS、异常/RTTI 开销 |
| `pthreads.rst`（187 行） | 多线程的**全部实操约束** |
| `emscripten-runtime-environment.rst` | 运行时环境模型 |
| `emscripten.h.rst`（1803 行） | C API 参考，速查用 |

## `Interacting-with-code.rst` —— 最实用的一篇

四种调用方向讲得很完整：

1. **JS 调 C**：`ccall` / `cwrap`（只支持 `number`/`string`/`array` 三种类型，
   **只对 C 有效，C++ 要包 `extern "C"`**）
2. **JS 直接调**：`Module._funcName`，更快但要自己做类型转换
   （`stringToNewUTF8` / `UTF8ToString`，且要手动 `_free`）
3. **C 调 JS**：`emscripten_run_script`（本质 `eval`，最慢）、
   **`EM_JS`**（推荐）、`EM_ASM`（内联 JS）
4. **C++ 类绑定**：Embind / WebIDL-Binder

关键工程细节：

- `EXPORTED_FUNCTIONS` 里函数名要加 `_` 前缀，且 **`_main` 不写就会被 DCE 掉**
- `EXPORTED_RUNTIME_METHODS` 决定 `ccall`/`cwrap` 等运行时方法是否被保留
- `-O2` 以上会 minify 函数名，导出是保住原名的唯一途径
- `-sMODULARIZE` 让产物变成工厂函数，`require()` 后调用返回 Promise

## `Optimizing-Code.rst`

优化等级（`-O0`~`-O3`、`-Os`、`-Oz`、`-Og`）之外，几个有信息量的点：

- **Emscripten 在链接阶段跑三层优化**：Binaryen 优化器（LLVM 做不到的
  Wasm 级 + 全程序优化）、JS 优化器 + 可选 Closure、
  **meta-DCE**（跨 JS/Wasm 边界的死代码消除）
- ⚠️ Binaryen 的全程序优化会做 LLVM IR 上标了 `noinline` 也拦不住的内联
- `-sWASM_BIGINT` 免去 **legalization**（把 i64 拆成两个 i32），链接更快
- `-sERROR_ON_WASM_CHANGES_AFTER_LINK` 可以强制"链接后不许改 Wasm"，用来诊断慢链接
- `-sEVAL_CTORS` 把全局构造函数（甚至 `main`）在编译期预跑并把结果**快照进 wasm**，
  加快启动。会在遇到 import 调用时停止——文档给了很具体的调优建议
  （比如把纯计算放在创建 GL 上下文之前）
- **`-fno-rtti -fno-exceptions` 在 Box2D 上实测减少 15% 体积**
- 分配器可选：`dlmalloc`（默认）/ `emmalloc`（更小更慢）/ `mimalloc`（多线程友好但更大）

## `pthreads.rst` —— 多线程实操约束的权威来源

[[wasm/threads-and-workers]] 那一页的实操部分几乎全部来自这里：

- **COOP/COEP 是硬前提**
- **不能单产物做"有线程就用、没线程降级"**——只能出两套构建
- `pthread_create` **需要返回事件循环**才能创建 Worker →
  三个解法（回事件循环 / `PTHREAD_POOL_SIZE` / `PROXY_TO_PTHREAD`）
- **主线程不能 `Atomics.wait`** → `pthread_join` 等变成忙等 →
  可能与 proxying 互相死锁 → 推荐 `-sPROXY_TO_PTHREAD`
- **proxying 机制**：JS 库函数用 `__proxy: 'sync'|'async'` 标注自动代理到主线程；
  `__proxy:'sync'` + `__async:'auto'` 的组合让后台线程可以阻塞等待异步操作
  **而不需要 Asyncify**
- 不支持 POSIX 信号（除 `pthread_kill`）、不支持 `fork()`
- **pthreads + `ALLOW_MEMORY_GROWTH` 很麻烦**：JS 访问 Wasm 内存变慢，
  `Module.HEAP*` 视图需要每次访问前刷新

## 时效性

✅ 滚动更新的文档，整体较新。

⚠️ 局部陈旧：

- `pthreads.rst` 结尾说"只能在 Firefox Nightly 里跑，SharedArrayBuffer 还在实验阶段"——
  **严重过时**，SharedArrayBuffer 早已标准化，所有主流浏览器在 COOP/COEP 下都支持
- 同文件里提到 Firefox 的 `dom.workers.maxPerDomain` 默认 20 的限制，
  是很老的信息
- `Optimizing-Code.rst` 里"eventually Wasm should gain native support for exceptions"——
  **Wasm 3.0 已经有了**，现在可以用 `-fwasm-exceptions`

## 覆盖的考点

- C/C++ 路线的全部实操问题
- 多线程的全部工程约束（这是面试里最能体现"真做过"的部分）
- 体积优化的具体手段和量化收益
- ❌ 不覆盖：Wasm 语义本身、Rust 路线

## 局限

Emscripten 文档极其庞大且 flag 数量惊人（`src/settings.js` 里有几百个）。
这里只抓了核心四篇。需要查具体 flag 时应该去
<https://emscripten.org> 或直接读 `src/settings.js`。

## 相关页面

- [[wasm/toolchain-emscripten]]
- [[wasm/threads-and-workers]]
- [[sources/rust-wasm-toolchain-docs]]
