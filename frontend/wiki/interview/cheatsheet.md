---
title: WebAssembly 速查卡
date: 2026-08-10
tags: [wasm, 速查, 面试]
sources: []
---

# WebAssembly 速查卡

面试前 10 分钟扫这一页。

## 核心四件套

| 概念 | 状态 | 一句话 |
| --- | --- | --- |
| `Module` | **无状态** | 编译后的代码，**可 postMessage 给 Worker** |
| `Instance` | 有状态 | Module + memory + table + imports |
| `Memory` | 有状态 | 可增长的线性字节数组，JS 侧是 ArrayBuffer |
| `Table` | 有状态 | 可增长的**引用**数组，主要放函数引用 |

## 关键数字

| 数字 | 含义 |
| --- | --- |
| **64 KiB** | Wasm 页大小（固定） |
| **4 GiB** | wasm32 的线性内存上限（memory64 突破） |
| **~23×** | 二进制解码 vs 解析 asm.js 的速度比 |
| **20–30%** | gzip 后二进制比 gzip 后 asm.js 小的比例 |
| **1.0 / 2.0 / 3.0** | spec 版本（2019 / 2022 / 2025） |
| **1** | 二进制头里的版本号（**至今没变过**） |
| **128 位** | SIMD 向量宽度（定宽） |
| **`\0asm`** | 二进制 magic number |

## 值类型

```text
数值：i32  i64  f32  f64
向量：v128
引用：funcref  externref  exnref
GC(3.0)：anyref  eqref  structref  arrayref  i31ref ...
```

**没有字符串、没有对象、没有数组。**

## JS ↔ Wasm 类型映射

| Wasm | JS |
| --- | --- |
| i32 | Number（`ToInt32`） |
| **i64** | **BigInt**（必须 `123n`） |
| f32 / f64 | Number |
| externref | 任意 JS 值 |
| funcref | JS 函数包装 |
| **v128** | **不能跨边界** |

## 加载代码模板

```js
// ✅ 首选
const { instance, module } = await WebAssembly.instantiateStreaming(
  fetch("app.wasm"),
  importObject,
);

// 编译一次，多 Worker 复用（module 无状态，可结构化克隆）
const mod = await WebAssembly.compileStreaming(fetch("app.wasm"));
worker.postMessage({ module: mod });
```

**服务器必须返回 `Content-Type: application/wasm`。**

## 三种错误

```text
CompileError  → 解码/验证阶段  → 二进制坏了、用了不支持的 proposal
LinkError     → 实例化阶段    → importObject 缺东西 / 类型不对
RuntimeError  → 运行阶段      → trap（越界、除零、签名不匹配、栈溢出）
```

## 会 trap 的操作

- 线性内存越界
- `call_indirect` 签名不匹配 / 索引越界
- 整数除零、`INT_MIN / -1`
- 浮点转整数超范围（`trunc_sat` 除外）
- 调用栈耗尽

## 内存操作模板

```js
// 读写数值
const dv = new DataView(memory.buffer);
dv.setUint32(0, 42, true);        // true = 小端，Wasm 永远小端

// 视图（不拷贝） vs 拷贝
new Uint8Array(memory.buffer, ptr, len);   // 视图
new Uint8Array(someTypedArray);             // 拷贝

// 字符串
new TextDecoder("utf-8").decode(new Uint8Array(memory.buffer, ptr, len));
```

## ⚠️ 五个必背的坑

1. **`memory.grow()` 会 detach 旧 ArrayBuffer** —— 所有缓存的 TypedArray 视图失效
2. **`--target bundler` 产物不能裸跑浏览器** —— 无打包器要用 `--target web`
3. **服务器不设 `application/wasm`** —— `instantiateStreaming` 直接失败
4. **i64 参数必须传 BigInt** —— 传 Number 抛 TypeError
5. **多线程需要 COOP/COEP** —— 会拦第三方跨域资源、断 `window.opener`

## WAT 语法速记

```wat
(module
  (import "console" "log" (func $log (param i32)))   ;; 两级命名空间
  (import "js" "mem" (memory 1))
  (global $g (mut i32) (i32.const 0))
  (data (i32.const 0) "Hi")

  (func $f1 (result i32) i32.const 42)
  (func $f2 (result i32) i32.const 13)
  (table 2 funcref)
  (elem (i32.const 0) $f1 $f2)                       ;; elem 之于 table = data 之于 memory
  (type $sig (func (result i32)))

  (func $add (param $a i32) (param $b i32) (result i32)
    local.get $a
    local.get $b
    i32.add)
  (func (export "callByIndex") (param $i i32) (result i32)
    local.get $i
    call_indirect (type $sig))
  (export "add" (func $add)))
```

**记忆点**：
- `br` 跳到 `block` = **break**（往后跳）；跳到 `loop` = **continue**（往前跳）
- 指令命名：`<type>.<op>[_s|_u]`，如 `i32.div_s`、`i64.extend_i32_s`
- `add`/`sub`/`mul` 溢出**回绕不 trap**；`div`/`rem` 除零**会 trap**
- 移位次数**取模**：`i32.shl` 移 32 位 = 不移

## 版本里程碑

| 版本 | 关键特性 |
| --- | --- |
| **2.0** | 多返回值、`externref`、bulk memory、定宽 SIMD、i64↔BigInt |
| **3.0** | **GC**、**异常处理**、memory64、多内存、尾调用、Relaxed SIMD、JS String Builtins、Branch Hinting |
| Phase 5 | JSPI、Web CSP |
| Phase 4 | Threads |
| Phase 3 | **ESM Integration**、Stack Switching、Custom Page Sizes |
| Phase 1 | **Component Model**、Shared-Everything Threads、Stringref |

## 工具链

```bash
# wabt
wat2wasm a.wat -o a.wasm
wasm2wat a.wasm --fold-exprs
wasm-objdump -h a.wasm         # section 体积分布
wasm-decompile a.wasm          # 最可读的反编译

# binaryen
wasm-opt -Oz a.wasm -o a.min.wasm
wasm-opt --strip-debug --strip-producers

# Rust
wasm-pack build --target web
twiggy top pkg/app_bg.wasm     # 体积分析

# Emscripten
emcc -O3 --closure 1 -sWASM_BIGINT -sENVIRONMENT=web a.c -o a.js
```

## 性能第一原则

```text
把边界画在粗粒度的地方：传大块数据 → 做大块工作 → 返回大块结果
```

```js
// ❌ 一百万次跨边界
for (let i = 0; i < 1e6; i++) sum += wasm.add(arr[i], 1);

// ✅ 一次跨边界
const ptr = wasm.alloc(arr.length * 4);
new Int32Array(wasm.memory.buffer, ptr, arr.length).set(arr);
const sum = wasm.sumAll(ptr, arr.length);
```

## 该不该用 Wasm：一句话版

> 先 profile 确认瓶颈是 CPU 计算；看有没有现成原生库可移植；
> 看接口能不能做成粗粒度大块传递；算总账（下载+编译+编解码 vs 节省的执行时间）；
> **先确认浏览器原生 API（WebCrypto / CompressionStream / WebGPU）能不能解决**。
> 另有一条独立路径：需要沙箱执行不可信代码时，用 Wasm 的理由是**能力安全，不是性能**。

## 三句话打消误解

- "Wasm 快 10 倍" → 稳态计算通常 **1–3 倍**，跨边界密集时可能**更慢**
- "Wasm 能直接操作 DOM" → **不能**，必须绕道 JS
- "编译成 Wasm 就内存安全了" → 沙箱保护的是**宿主**，模块**自己的堆照样能写坏**

## 相关页面

- 题库：[[interview/question-bank]]
- 路线：[[interview/roadmap]]
- 全库导航：[[index]]
