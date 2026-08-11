---
title: 多线程 —— SharedArrayBuffer / Atomics / Worker
date: 2026-08-10
tags: [wasm, 多线程, worker, sharedarraybuffer, coop-coep]
sources: [mdn-wasm/guides/understanding_the_text_format.md, emscripten-doc/pthreads.rst, wasm-design/Nondeterminism.md, wasm-design/Portability.md, wasm-design/proposals-README.md]
---

# 多线程

## 整体架构

Wasm 自己**不创建线程**。线程是宿主的能力：在浏览器里就是 **Web Worker**。
Wasm 提供的是"多个 Worker 共享同一块内存"的机制：

```text
主线程          Worker 1        Worker 2
  │               │               │
  └───────────────┴───────────────┘
              ↓
      同一个 WebAssembly.Memory (shared: true)
      → buffer 是 SharedArrayBuffer
      → 用 atomic 指令同步
```

Threads proposal 目前在 **Phase 4**（正在标准化），但浏览器早已广泛实现。

## 两个组成部分

### 1. 共享内存

```js
const memory = new WebAssembly.Memory({
  initial: 10,
  maximum: 100,      // shared 内存必须指定 maximum
  shared: true,
});
memory.buffer;       // SharedArrayBuffer
worker.postMessage({ memory });   // 可传给 Worker
```

```wat
(memory 1 2 shared)
```

**为什么 shared 必须指定 maximum**：共享内存 grow 时**不能搬家**——
否则其它线程持有的 SharedArrayBuffer 全部失效。所以引擎必须提前预留地址空间，
grow 只是把可用边界往外推。这也意味着**共享内存 grow 不会 detach**。

### 2. 原子指令

Wasm 提供 `i32.atomic.load` / `i32.atomic.store` / `i32.atomic.rmw.*`（read-modify-write）
/ `memory.atomic.wait32` / `memory.atomic.notify` 等指令，
用来实现互斥锁、条件变量这类高层同步原语。

对应 JS 侧的 `Atomics.load` / `Atomics.store` / `Atomics.wait` / `Atomics.notify`。

> **面试落点**：Wasm 多线程 = **SharedArrayBuffer（共享数据）+ atomic 指令（同步）+
> Web Worker（执行单元）**。三者缺一不可，Wasm 本身只负责中间那层。

## COOP / COEP：绕不过的部署门槛

因为 Spectre 类侧信道攻击，浏览器把 `SharedArrayBuffer` 锁在
**跨源隔离（cross-origin isolated）**状态之后。要开启，服务器必须返回：

```text
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

检测：

```js
if (self.crossOriginIsolated) {
  // SharedArrayBuffer 可用
}
```

**代价非常实在**，这是面试里能体现工程经验的地方：

- 开了 COEP 之后，**所有跨域子资源**（图片、字体、iframe、第三方脚本、分析 SDK）
  都必须带 `Cross-Origin-Resource-Policy: cross-origin` 或走 CORS，否则被浏览器拦截
- COOP 会**切断与 opener 窗口的引用关系**，`window.opener` 变 null——
  OAuth 弹窗登录、支付回调这类流程会挂
- `credentialless` 是 COEP 的宽松变体（`Cross-Origin-Embedder-Policy: credentialless`），
  允许无凭据地加载跨域资源，兼容性代价小一些

> **面试落点**：多线程 Wasm 的最大障碍**不是技术，是部署**。
> COOP/COEP 会波及页面上所有第三方资源，很多项目权衡后放弃多线程。
> 能说出"COEP 会拦第三方资源、COOP 会断 window.opener"就说明真踩过坑。

## 单构建 vs 双构建

> **⚠️ 常见误答**："一个 Wasm 产物可以有线程时用线程、没线程时降级" ——
> **做不到**。Emscripten 官方明确说明：只能出**两套构建**，运行时靠
> `crossOriginIsolated` 或特性检测二选一。

因为线程版和非线程版的内存布局、原子指令使用、TLS 处理都不同，无法在同一份字节码里切换。

## Emscripten pthreads 的实操约束

```bash
emcc -pthread \
     -sPTHREAD_POOL_SIZE=navigator.hardwareConcurrency \
     -sPROXY_TO_PTHREAD \
     main.c
```

### `pthread_create` 需要回到事件循环

创建 Web Worker 是异步的。所以"创建线程后立即同步等待它"会挂：

```c
// ❌ 在 Emscripten 里会挂
pthread_create(&t, NULL, work, NULL);
pthread_join(t, NULL);            // worker 还没启动
```

三个解法：

1. 回到主事件循环（`emscripten_set_main_loop`）
2. `-sPTHREAD_POOL_SIZE=N` —— **提前创建好 worker 池**，`pthread_create` 直接取用
3. `-sPROXY_TO_PTHREAD` —— 把 `main()` 挪到 worker 上，`pthread_create` 被代理回主线程处理

### 主线程不能阻塞

`Atomics.wait` **在主线程上是被禁止的**。而 `pthread_join`、`pthread_mutex_lock`、
`pthread_cond_wait`、`usleep` 底层都是 futex wait。

Emscripten 的应对是在主线程上**忙等（busy-wait）**——能跑，但会卡死 UI、浪费电。
默认对 `pthread_join`/`pthread_cond_wait` 在主线程发警告，
`ALLOW_BLOCKING_ON_MAIN_THREAD=0` 时直接报错。

更糟的是**死锁**：主线程忙等的同时，worker 正在尝试把某个操作 proxy 到主线程执行 →
主线程永远不会响应 → 双方僵死。

**推荐做法就是 `-sPROXY_TO_PTHREAD`**：主线程只做事件分发和 DOM 操作，
应用逻辑全在 worker 上，天然避开这类问题。

### 主线程代理（proxying）

DOM 操作只能在主线程做。Emscripten 的 JS 库函数带 `__proxy: 'sync' | 'async'` 标注，
从后台线程调用时**自动代理到主线程**：

- `sync`：调用线程阻塞，等主线程执行完拿返回值
- `async`：立即返回，主线程稍后执行

一个巧妙的组合：标了 `__proxy: 'sync'` + `__async: 'auto'` 的函数返回 Promise 时，
**后台线程可以阻塞等待异步操作完成——不需要 Asyncify**。
（因为后台线程在 Worker 里，`Atomics.wait` 是允许的。）

### 其它限制

- **不支持 POSIX 信号**（除 `pthread_kill`）——无法抢占 Worker 的执行
- **不支持 `fork()`**
- 线程回调函数指针的签名必须严格匹配（少写 `void*` 参数在 x86 上能跑，在 Wasm 上会 trap）
- **pthreads + `ALLOW_MEMORY_GROWTH` 很麻烦**：JS 侧访问 Wasm 内存会变慢，
  且 `Module.HEAP*` 视图需要在每次访问前刷新
- 分配器竞争：默认 `dlmalloc` 有全局锁，多线程下换 `-sMALLOC=mimalloc`
  （每线程独立分配上下文，但体积更大、内存占用更高）

## Rust 多线程

```toml
# 需要 nightly + build-std
[dependencies]
wasm-bindgen-rayon = "1"
```

```bash
RUSTFLAGS='-C target-feature=+atomics,+bulk-memory,+mutable-globals' \
  cargo build --target wasm32-unknown-unknown -Z build-std=std,panic_abort
```

`wasm-bindgen-rayon` 让 `rayon` 的并行迭代器（`par_iter()`）在浏览器里可用，
底层就是一组 Worker + 共享内存。同样受 COOP/COEP 约束。

## 不确定性

共享内存**引入了 Wasm 里少有的不确定性**：
`load`、read-modify-write、`wait`、`notify` 的结果依赖线程调度，是不确定的。

官方 Nondeterminism 文档把它和 NaN 位模式、Relaxed SIMD、资源耗尽并列为
Wasm 允许不确定性的几个地方之一。

同时，Wasm 的可移植性假设里明确要求宿主提供：
**对所有执行线程的前进保证（forward progress guarantee）**，
以及 8/16/32 位自然对齐的**无锁原子操作**（至少要有 CAS）。

## Shared-Everything Threads

当前的 threads proposal 只共享**线性内存**，不共享 table、global、GC 对象。
**Shared-Everything Threads**（Phase 1）想把共享范围扩大到所有实体——
这是让 Java/Kotlin 这类"共享一切对象"的语言真正能在 Wasm 上多线程的前提。

## 实践架构建议

```text
主线程          → UI、DOM、事件、渲染
Worker (Wasm)   → 计算内核
```

即使**不用共享内存**，"Wasm 跑在 Worker 里 + postMessage 传数据"也是很好的架构：

- 不需要 COOP/COEP
- 用 **Transferable**（`ArrayBuffer`）传数据可以零拷贝转移所有权
- 主线程完全不被计算阻塞

只有在**多个线程需要同时读写同一大块数据**时才真正需要 SharedArrayBuffer。

> **面试落点**：先问"是不是真的需要共享内存"。
> **单 Worker + Transferable** 就能解决大部分"别卡主线程"的需求，
> 且没有 COOP/COEP 的部署代价。这是很务实的判断，比直接上多线程更能体现工程判断力。

## 相关页面

- 共享内存细节：[[wasm/memory-model]]
- Emscripten 编译选项：[[wasm/toolchain-emscripten]]
- 部署响应头：[[wasm/build-integration]]
- 安全背景（Spectre）：[[wasm/security-sandbox]]
