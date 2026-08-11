---
title: C/C++ 工具链 —— Emscripten
date: 2026-08-10
tags: [wasm, emscripten, c, cpp, 工具链]
sources: [mdn-wasm/guides/c_to_wasm.md, mdn-wasm/guides/existing_c_to_wasm.md, emscripten-doc/Interacting-with-code.rst, emscripten-doc/Optimizing-Code.rst, emscripten-doc/pthreads.rst, emscripten-doc/emscripten-runtime-environment.rst, wasm-design/CAndC++.md]
---

# Emscripten（C/C++ → Wasm）

Emscripten 的定位不是"C 编译器"，而是**"把 POSIX/C 生态搬到 Web 的兼容层"**。
它做的三件事：

1. `clang + LLVM` 把 C/C++ 编译成 Wasm
2. **生成 JS 胶水代码**——因为 Wasm 不能直接访问 Web API
3. 用 Web API 实现一大堆 C 库：libc、SDL、OpenGL（映射到 WebGL）、OpenAL、pthreads、
   甚至一个虚拟文件系统

> **面试落点**：Emscripten 的胶水代码不只是"加载 wasm"，它还**用 Web API 重新实现了
> 半个 POSIX**——这就是为什么 Emscripten 的产物比 Rust 的大得多，也是为什么
> 一个 C 库能几乎不改代码就跑在浏览器里。

## 最小流程

```c
#include <emscripten.h>
#include <math.h>        // sqrt

EMSCRIPTEN_KEEPALIVE     // 防止被 DCE 掉
int int_sqrt(int x) { return (int)sqrt(x); }
```

```bash
emcc test.c -o out.html \
  -sEXPORTED_FUNCTIONS=_int_sqrt \
  -sEXPORTED_RUNTIME_METHODS=ccall,cwrap
```

产出 `out.html` + `out.js` + `out.wasm`。也可以 `-o out.js` 只要 JS+wasm。

## 四种调用方向

### 1. JS 调 C：`ccall` / `cwrap`

```js
// cwrap：包一次，反复调
const int_sqrt = Module.cwrap("int_sqrt", "number", ["number"]);
int_sqrt(12);   // 3

// ccall：调一次
Module.ccall("int_sqrt", "number", ["number"], [28]);   // 5
```

支持的类型只有 `"number"`（整数/浮点/指针）、`"string"`（`char*`）、
`"array"`（`Uint8Array`/`Int8Array`）。

**注意**：只对 **C** 函数有效，C++ 会 name mangling，必须包 `extern "C"`。

### 2. JS 直接调（更快、更麻烦）

编译产物里 C 函数名前会加下划线：

```js
Module._int_sqrt(12);            // 直接调，无类型转换开销
const ptr = stringToNewUTF8(s);  // JS string → char*，需要手动 _free(ptr)
UTF8ToString(ptr);               // char* → JS string
```

### 3. C 调 JS：`EM_JS` / `EM_ASM` / `emscripten_run_script`

```c
#include <emscripten.h>

// EM_JS：声明一个 JS 实现的 C 函数（推荐）
EM_JS(void, call_alert, (), {
  alert('hello world!');
});

// EM_ASM：内联 JS，像内联汇编
int main() {
  EM_ASM({ console.log('inline js'); });
  call_alert();
}
```

`emscripten_run_script("alert('hi')")` 本质是 `eval()`——**最慢，避免使用**。

### 4. C++ 类绑定：Embind

```cpp
#include <emscripten/bind.h>
using namespace emscripten;

class Counter { public: int value = 0; void inc() { value++; } };

EMSCRIPTEN_BINDINGS(my_module) {
  class_<Counter>("Counter")
    .constructor<>()
    .function("inc", &Counter::inc)
    .property("value", &Counter::value);
}
```

```js
const c = new Module.Counter();
c.inc();
c.value;     // 1
c.delete();  // ⚠️ 必须手动释放！JS 的 GC 管不了 C++ 对象
```

> **⚠️ 常见误答**：忘记 Embind 对象要手动 `.delete()`。JS GC 不知道线性内存里的
> C++ 对象，不调用 `delete()` 就是**内存泄漏**。这是 Embind 项目最常见的线上问题。

## 把数据搬进 Wasm：典型模式

以"用 libwebp 把 canvas 图像编码成 WebP"为例（MDN 的完整案例）：

```c
#include <emscripten.h>
#include <stdlib.h>      // malloc / free
#include <stdint.h>      // uint8_t

// C 侧导出内存管理函数
EMSCRIPTEN_KEEPALIVE
uint8_t* create_buffer(int width, int height) {
    return malloc(width * height * 4 * sizeof(uint8_t));
}
EMSCRIPTEN_KEEPALIVE
void destroy_buffer(uint8_t* p) { free(p); }
```

```js
const image = ctx.getImageData(0, 0, w, h);        // Uint8ClampedArray, RGBA
const p = api.create_buffer(image.width, image.height);
Module.HEAP8.set(image.data, p);                   // 拷贝进线性内存
api.encode(p, image.width, image.height, 100);

// 取结果：视图 vs 拷贝的区别很关键
const resultView = new Uint8Array(Module.HEAP8.buffer, ptr, size);  // 视图，不拷贝
const result = new Uint8Array(resultView);                          // 这一步才拷贝
api.free_result(ptr);
api.destroy_buffer(p);
```

> **面试落点**：`new Uint8Array(buffer, ptr, len)` 建**视图**，
> `new Uint8Array(typedArray)` 做**拷贝**。释放 Wasm 内存前必须先拷出来，
> 否则视图指向的内存已被复用。

内存不够时的报错解法是 `-sALLOW_MEMORY_GROWTH=1`。

## 优化选项速查

| flag | 作用 |
| --- | --- |
| `-O0`~`-O3` | 常规优化等级，`-O3` 编译慢但最快 |
| `-Os` / `-Oz` | 优先压体积，`-Oz` 更激进 |
| `--closure 1` | 用 Closure Compiler 压缩**胶水 JS**（体积收益巨大，但需要正确的 annotation） |
| `-flto` | 链接期优化，编译和链接都要加 |
| `-sWASM_BIGINT` | i64 直接用 BigInt，**免去 legalization**，链接更快 |
| `-sALLOW_MEMORY_GROWTH` | 允许内存增长 |
| `-sFILESYSTEM=0` | 不打包虚拟文件系统代码 |
| `-sENVIRONMENT=web` | 只生成 web 环境代码，省 ~2KB |
| `-sMODULARIZE` | 产物是工厂函数，`require()` 后调用返回 Promise |
| `-sEVAL_CTORS` | 编译期预跑全局构造函数并把结果"快照"进 wasm，加快启动 |
| `-fno-rtti -fno-exceptions` | 关掉 RTTI 和异常（Box2D 实测**减少 15% 体积**） |
| `-sMALLOC=emmalloc` / `mimalloc` | 换分配器：小 / 多线程友好 |

Emscripten 在链接阶段还会额外跑：

- **Binaryen 优化器**（做 LLVM 做不到的全程序 Wasm 级优化）
- **JS 优化器** + 可选的 Closure
- **meta-DCE**：跨 JS/Wasm 边界的死代码消除，最小化 import/export

> **面试落点**：能说出"**Binaryen** 是 Wasm 层面的优化器和工具库（`wasm-opt`），
> 与 LLVM 是两个阶段"就很好。Emscripten 和 wasm-pack 都在最后跑 `wasm-opt`。

## C++ 异常与 RTTI 的历史包袱

在 Wasm 原生异常处理（Wasm 3.0）之前，C++ 异常只能用 JS 的 try/catch 模拟，
开销大到 **Emscripten 在 `-O1` 及以上默认不生成 catch 块**——
抛异常直接终止程序。要恢复需要 `-sDISABLE_EXCEPTION_CATCHING=0`。

即使不生成 catch，只要没加 `-fno-exceptions`，异常支持代码仍有体积开销。

现在可以用 `-fwasm-exceptions` 走原生异常路径（需要目标浏览器支持）。

## Asyncify

移植过来的 C 代码常常是同步的（`fread`、`sleep`、主循环），但 Web API 是异步的。
**Asyncify**（`-sASYNCIFY`）把 Wasm 改写成可暂停/恢复的形式，
让同步 C 代码能"等待"一个 JS Promise。

代价：**体积显著膨胀、性能下降**（要保存/恢复整个调用栈状态）。
这也是 JSPI（见 [[wasm/js-interop]]）想取代它的原因。

## pthreads

```bash
emcc -pthread -sPTHREAD_POOL_SIZE=navigator.hardwareConcurrency \
     -sPROXY_TO_PTHREAD main.c
```

几个必须知道的约束：

- **需要 COOP/COEP 响应头**才能用 SharedArrayBuffer，否则线程代码根本跑不起来
- **不能"有线程就用、没线程就降级"**——只能出两套构建产物，运行时二选一
- `pthread_create` **需要返回事件循环**才能真正创建 Worker。所以创建线程后立即
  `pthread_join` 会挂。解法：`-sPTHREAD_POOL_SIZE`（预创建）或 `-sPROXY_TO_PTHREAD`
- **主线程不能 `Atomics.wait`**，所以 `pthread_join`/`pthread_mutex_lock` 在主线程上
  变成**忙等**，会卡死 UI。默认会警告；`ALLOW_BLOCKING_ON_MAIN_THREAD=0` 会直接报错
- `-sPROXY_TO_PTHREAD` 把 `main()` 挪到 worker 上跑，主线程只处理代理过来的事件——
  **这是推荐做法**
- 不支持 POSIX 信号（除 `pthread_kill`）、不支持 `fork()`
- **pthreads + ALLOW_MEMORY_GROWTH 组合很麻烦**：JS 访问 Wasm 内存会变慢，
  且 `Module.HEAP*` 视图需要在每次访问前刷新

> **面试落点**：Emscripten 多线程的三大约束——**COOP/COEP 响应头、
> 主线程不能阻塞（要 PROXY_TO_PTHREAD）、pthread_create 需要回到事件循环**。
> 答出这三条就说明真做过。

## Emscripten vs Rust：怎么选

| | Emscripten | Rust + wasm-bindgen |
| --- | --- | --- |
| 适合 | **移植既有 C/C++ 代码库** | **新写的计算模块** |
| 产物体积 | 大（带 libc + 各种 shim） | 小（无运行时） |
| JS 集成体验 | ccall/cwrap/Embind，偏手工 | 自动生成胶水 + `.d.ts`，体验最好 |
| POSIX 兼容 | 强（文件系统、SDL、OpenGL） | 弱（需要自己找 crate） |
| 学习曲线 | 会 C 就能上手，但 flag 多如牛毛 | 需要学 Rust |

> **面试落点**：**有既有 C/C++ 代码就用 Emscripten，从零写新模块就用 Rust。**
> ffmpeg.wasm、SQLite WASM、Photoshop Web 走的都是 Emscripten 路线；
> 新写的图像/压缩/加密内核多用 Rust。

## 相关页面

- Rust 路线：[[wasm/toolchain-rust]]
- 多线程细节：[[wasm/threads-and-workers]]
- 真实案例：[[wasm/use-cases]]
- 性能与体积：[[wasm/performance]]
