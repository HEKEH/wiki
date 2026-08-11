---
title: WebAssembly 高频面试题与参考答案
date: 2026-08-10
tags: [wasm, 面试, 题库]
sources: []
---

# WebAssembly 面试题库

按难度分三档。**每题的参考答案都控制在"口述 30–60 秒"的长度**——
面试里回答太长反而扣分，讲清核心 + 留钩子让对方追问才是最优策略。

---

## 一档：基础（一面必问）

### Q1. WebAssembly 是什么？

**参考答案**：
一种为编译目标设计的、可移植的二进制指令格式，跑在安全沙箱里的栈式虚拟机上。
它不是给人手写的，而是 C/C++/Rust 这类语言的编译产物，
目标是让这些语言以接近原生的速度运行在 Web 上。

要强调：它**不是机器码**，浏览器拿到后仍然要编译成本机代码——
只是这个过程比解析 JS 快得多，而且结果可预测（没有 deopt）。

→ [[wasm/what-is-wasm]]

### Q2. Wasm 会取代 JavaScript 吗？

**参考答案**：
不会，官方 FAQ 明确说是**互补**关系。JS 是高层、动态、生态庞大的语言，
适合 UI 和业务逻辑；Wasm 是低层、静态、性能可预测的编译目标，适合计算密集内核。
而且 Wasm **不能直接访问 DOM**，必须通过导入的 JS 函数间接调用——
它在架构上就依赖 JS。

### Q3. Wasm 为什么比 JS 快？

**参考答案**（分三层讲，层次感是加分点）：

1. **加载快**：二进制格式解码比解析等价的 asm.js 源码快约 23 倍，
   名字都编码成整数索引，还能边下载边编译。
2. **执行可预测**：类型静态确定且编译前验证过，引擎直接生成机器码，
   **没有类型推测和反优化（deopt）**。
3. **贴近硬件**：只有 i32/i64/f32/f64/v128，全是 CPU 原生类型。

但要主动补一句：**稳态计算上 Wasm 通常只是优化过的 JS 的 1–3 倍，
不存在"快 10 倍"这种普适说法**。涉及大量跨边界调用或字符串操作时可能更慢。

→ [[wasm/performance]]

### Q4. Wasm 有哪些数据类型？

**参考答案**：
数值类型 `i32` `i64` `f32` `f64`，向量类型 `v128`，
引用类型 `funcref` `externref` `exnref`（Wasm 3.0 的 GC 还加了 `structref`/`arrayref` 等）。

**没有字符串、没有对象、没有数组**——这是刻意的：Wasm 定位是"虚拟 ISA"，
复杂类型由源语言编译器自己用基本类型和线性内存拼出来。

→ [[wasm/types-and-abi]]

### Q5. 怎么在网页里加载一个 Wasm 模块？

```js
const { instance } = await WebAssembly.instantiateStreaming(
  fetch("app.wasm"),
  importObject,
);
instance.exports.foo();
```

**要点**：`instantiateStreaming` 在网络字节流上**边下载边编译**，
省掉"等下载完 → 转 ArrayBuffer → 再编译"的串行等待。
前提是服务器返回 `Content-Type: application/wasm`。

→ [[wasm/js-interop]]

---

## 二档：进阶（二面拉开差距）

### Q6. Wasm 怎么和 JS 传字符串？

**参考答案**：
Wasm 没有字符串类型，所以传的是**线性内存里的偏移 + 长度**。
JS 侧用 `new Uint8Array(memory.buffer, ptr, len)` 建一个**视图**（不拷贝），
再用 `TextDecoder` 解码。反方向就是 `TextEncoder` 编码后写进内存。

wasm-bindgen、Emscripten 的胶水代码自动化的就是这件事：
调 Wasm 导出的 `malloc` 分配、写内存、传 `(ptr, len)`、调用完再 `free`。

→ [[wasm/types-and-abi]]、[[wasm/text-format-wat]]

### Q7. `memory.grow()` 有什么坑？

**参考答案**（**最高频的实战题**）：
`ArrayBuffer.byteLength` 不可变，所以 grow 成功后 `memory.buffer` 会返回一个
**全新的 ArrayBuffer**，旧的变成 **detached**，
**所有基于旧 buffer 建的 TypedArray 视图全部失效**。

所以不能缓存视图：

```js
// ❌ 危险
const heap = new Uint8Array(wasm.memory.buffer);
wasm.doSomethingThatAllocates();   // 内部触发 grow
heap[0];                            // 💥 detached

// ✅ 每次重新取
const u8 = () => new Uint8Array(wasm.memory.buffer);
```

补充加分点：**共享内存（`shared: true`）grow 时不会 detach**——
因为它不能搬家，所以必须提前指定 `maximum` 预留地址空间。

→ [[wasm/memory-model]]

### Q8. 为什么 Wasm 需要 Table？函数引用为什么不能放线性内存？

**参考答案**：
`call` 指令的函数索引是静态立即数，只能调固定函数。但 C 有函数指针、
C++ 有虚函数，需要运行时决定调谁，所以有了 `call_indirect`。

那为什么不直接传函数引用作为操作数？因为**引用不能存进线性内存**——
线性内存的字节对模块完全可见可改，把真实函数地址放进去，
既泄露地址信息，又能被伪造成任意跳转目标，沙箱就破了。

**解法**：引用存 table 里，代码里传 **i32 索引**。`call_indirect` 做两次检查：
**边界检查 + 签名检查**，任一失败就 trap。这就是 Wasm 控制流完整性的核心。

→ [[wasm/text-format-wat]]、[[wasm/security-sandbox]]

### Q9. Wasm 的三种错误类型分别什么时候抛？

| 错误 | 阶段 | 原因 |
| --- | --- | --- |
| `CompileError` | 解码/验证 | 二进制损坏、类型不匹配、用了不支持的 proposal |
| `LinkError` | 实例化 | importObject 缺字段、类型对不上 |
| `RuntimeError` | 运行 | trap：越界、除零、签名不匹配、栈溢出 |

**加分**：把它们对应到"编译 → 实例化 → 运行"三个生命周期阶段来讲。

→ [[wasm/core-model]]

### Q10. i64 跨边界有什么问题？

**参考答案**：
JS 的 Number 是 f64，无法精确表示全部 64 位整数。
所以 Wasm 的 JS-BigInt-integration（Wasm 2.0 进 spec）规定 **i64 映射为 BigInt**：

```js
instance.exports.f(123);    // ❌ TypeError
instance.exports.f(123n);   // ✅
```

**加分**：更早的引擎直接抛错，Emscripten 为此有"legalization"——
把 i64 拆成两个 i32 传。`-sWASM_BIGINT` 可以关掉它，链接更快。
另外 **BigInt 有装箱成本，热路径上尽量用 i32/f64**。

### Q11. Module 和 Instance 有什么区别？为什么这个区别重要？

**参考答案**：
`Module` 是**编译后的无状态代码**，`Instance` 是 Module + 它运行时的全部状态
（memory、table、imports）。类比：Module ≈ 函数字面量，Instance ≈ 闭包。

**为什么重要**：因为无状态，Module 可以像 Blob 一样 `postMessage()` 给 Worker——
**编译一次，N 个 Worker 复用**。这是多线程 Wasm 最重要的启动优化。
Instance 有状态，不能这样共享。

→ [[wasm/core-model]]、[[wasm/js-interop]]

### Q12. Wasm 是怎么做多线程的？

**参考答案**：
Wasm 自己不创建线程，线程是宿主的能力（浏览器里就是 Web Worker）。
Wasm 提供的是三样东西的组合：

1. **共享内存**：`new WebAssembly.Memory({ shared: true })`，buffer 是 SharedArrayBuffer
2. **原子指令**：`i32.atomic.rmw.*`、`memory.atomic.wait32`/`notify`
3. **Web Worker** 作为执行单元

**必须补的一句**（体现工程经验）：用 SharedArrayBuffer 需要
**COOP/COEP 响应头**。代价很大——COEP 会拦所有没标 CORP/CORS 的跨域资源，
COOP 会把 `window.opener` 置 null（OAuth 弹窗登录会挂）。
很多项目权衡后放弃了多线程 Wasm。

→ [[wasm/threads-and-workers]]

### Q13. wasm-pack 和 wasm-bindgen 是什么关系？

**参考答案**：三层：

```text
rustc --target wasm32-unknown-unknown  → 原始 wasm
wasm-bindgen（crate 宏 + CLI）          → 剥离冗余 + 生成 JS 胶水 + .d.ts
wasm-pack                               → 串起流水线 + 生成 package.json + wasm-opt
```

`wasm-bindgen` 的宏在编译期把类型信息写进 wasm 的 **custom section**，
CLI 在链接后读取并生成对应的 JS 编解码代码。它是**构建期工具，不是运行时库**。

**加分**：`--target bundler`（默认）的产物**不能直接跑在浏览器里**，
因为它假设"wasm 是原生 ES module"，而 ESM Integration 还在 Phase 3。
不用打包器就必须选 `--target web`。

→ [[wasm/toolchain-rust]]

---

## 三档：深度（三面 / 架构面）

### Q14. 什么时候该用 Wasm，什么时候不该？

见 [[analysis/when-to-use-wasm]] 里的"面试怎么答这道题"，那一段是照着面试场景写的。

核心结构：**先 profile → 看有没有现成原生库可移植 → 看接口能不能做成粗粒度 →
算总账（下载+编译+编解码 vs 节省的执行时间）→ 先确认浏览器原生 API 能不能解决。**

再补一条独立路径：**需要沙箱执行不可信代码时，用 Wasm 的理由是能力安全，不是性能**。

### Q15. Wasm 的沙箱到底保护了什么？没保护什么？

**参考答案**（**区分度最高的一题**）：

**硬保证（保护用户）**：

- 模块逃不出沙箱，越界一定 trap
- **控制流完整性免费获得**：代码不可变不可观测，返回地址在 VM 保护的调用栈上
  （不在可寻址的线性内存里），间接调用有签名检查 → **天然免疫代码注入和传统 ROP**
- 所以 DEP、栈 canary 这类经典缓解措施在 Wasm 里是不需要的

**没保护的（尽力而为）**：

- **线性内存内部**的越界覆写——边界检查只在整块内存边界上做，不是上下文敏感的
- 函数粒度的代码重用攻击（篡改 table 索引跳到另一个同签名函数）
- 竞态、TOCTOU、侧信道
- **源码层的 UB 依然是 UB**

结论：**"把 C 代码编译成 Wasm 就内存安全了"是错的**。
一个有堆溢出的 C 库编译到 Wasm 后漏洞依然存在，只是利用它拿不到 RCE，
最多把这个模块自己的数据搞坏。

**极高加分**：Wasm 里**没有 ASLR、没有栈 canary、堆布局高度确定**，
所以在"只污染模块自己内存"这个层面，Wasm 里的堆溢出反而**比原生更容易稳定利用**。

→ [[wasm/security-sandbox]]

### Q16. Wasm 3.0 带来了什么？GC 为什么重要？

**参考答案**：
Wasm 3.0（2025）进 spec 的主要有：**GC、异常处理、memory64、多内存、尾调用、
Relaxed SIMD、JS String Builtins、Branch Hinting**。

**GC 最重要**。在它之前，Java/Kotlin/Dart 这类 GC 语言编译到 Wasm
只能**把整个 GC 运行时也编译进线性内存**，后果是：产物体积暴涨、
**和 JS 对象无法共享**、跨堆循环引用无法回收、引擎的 GC 优化全用不上。

有了 GC proposal，Wasm 可以声明 struct/array 类型交给引擎的 GC 管理，
和 JS 对象活在同一个堆里。

**必须补的边界**：**Rust/C++ 不用 GC**，它们仍然用线性内存，
GC proposal 对它们没有影响。

→ [[wasm/proposals-and-versions]]

### Q17. Wasm 为什么选栈机而不是寄存器机或 AST？

**参考答案**（三条官方理由）：

1. **编码更小**——不需要显式编码操作数位置
2. **结构化控制流让验证可以单遍线性完成**，不需要像早期 JVM 那样做不动点计算 →
   这是**流式编译**的前提
3. 可以直接解码进编译器的 SSA 内部表示

注意是**结构化**栈机：控制流必须是 `block`/`loop`/`if` 嵌套，不能任意 goto。
任何控制流（包括不可归约的）都能用 **Relooper 算法**转成结构化形式。

**加分**：`br`/`return`/`unreachable` 之后的代码是**多态栈类型**的——
验证器允许假设栈是任意类型。这不是偷懒，是为了保证类型系统的**可组合性**：
不然编译器把 `x/0` 优化成一条 `br` 就会产出无法验证的代码。

→ [[wasm/core-model]]

### Q18. Wasm 里有哪些不确定性？

**参考答案**（能列全就非常突出）：

1. 不同引擎支持的 proposal 不同
2. 导出函数被调用的顺序、外部传入的参数值（宿主决定）
3. **共享内存**上的 load / RMW / wait / notify
4. **NaN 的位模式**
5. 无 NaN 输入却产生 NaN 时的**符号位**
6. **Relaxed SIMD** 指令的结果
7. **资源耗尽**（内存分配、物理页分配、栈、句柄）

设计原则是 **"limited, local"**：只在少数明确定义的场景出现，且影响是局部的——
不像 C++ 的 UB 那样让整个程序失去意义。

**NaN 为什么不确定**：硬件行为本来就不统一——x86 在无 NaN 输入产生 NaN 时置符号位，
ARM 不置；多 NaN 输入时选哪个也不同。强制规范化要在每次浮点运算后加检查，
开销不可接受、收益边际。

→ [[wasm/security-sandbox]]

### Q19. 为什么 Wasm 页大小是 64KiB？

**参考答案**：
64KiB 是**大量平台和 CPU 页大小的最小公倍数**。定死一个值才能保证
程序在所有引擎上行为一致（可移植性目标）。同时页对齐让引擎能用
**虚拟内存机制做边界检查**（把整块线性内存映射到一段受保护的虚拟地址区间，
越界访问触发硬件页错误，不需要每条指令都插入软件检查）。

**加分**：Custom Page Sizes proposal（Phase 3）正在做可变页大小，
主要是为了 IoT/嵌入式——64KiB 对某些设备来说粒度太粗了。

### Q20. Wasm 在服务端有什么价值？

**参考答案**：
**主要卖点不是性能，是隔离性和启动速度**。
Wasm 实例的冷启动比容器快好几个数量级。加上"没导入的能力就绝对拿不到"
的能力安全模型，可以在同一进程里安全地跑成千上万个租户的代码。

（**说"数量级"就够了，别报具体数字**——微秒/毫秒的具体值是厂商口径，
面试官如果追问"你测过吗"就很被动。）

典型场景：Fastly Compute@Edge、Cloudflare Workers、Shopify Functions（多租户 serverless），
Envoy/Istio WasmPlugin（插件系统），CosmWasm/NEAR（智能合约，看中的是**确定性**）。

**WASI** 就是给非浏览器宿主的标准 import 集合。0.1 是扁平接口，
0.2 基于 Component Model 做了模块化和**可虚拟化**，0.3 引入原生 async。
**注意 Component Model 本身还在 Phase 1**，别说成现状。

→ [[wasm/beyond-browser]]

### Q21. 你在项目里怎么集成 Wasm？会踩什么坑？

**参考答案**（照着真实经验讲，列 5 个具体的坑）：

1. **服务器 MIME 类型**——不设 `application/wasm`，`instantiateStreaming` 直接失败
2. **`--target bundler` 产物不能裸跑**——用 Vite 无打包器场景要选 `--target web`
3. **打包器可能把小 wasm 内联成 base64**——破坏流式编译，大模块要确认是独立文件
4. **`memory.grow` 之后视图失效**——所有缓存的 TypedArray 都要重建
5. **产物体积**——生产构建必须 strip name/DWARF section，跑 `wasm-opt -Oz`，开 brotli

加载策略上：**不要在入口就 `await` 一个大 Wasm**，用动态 import 按需加载，
或者 `requestIdleCallback` 空闲预热。

→ [[wasm/build-integration]]、[[wasm/debugging]]

---

## 反问环节可以问的

（面试尾声反问能体现你真的懂）

- "你们的 Wasm 模块是移植的既有库还是新写的？体积和加载时间怎么控制的？"
- "有没有上多线程？COOP/COEP 对页面上的第三方资源有影响吗？"
- "Wasm 3.0 的 GC 出来之后，有没有考虑过换语言或者换方案？"

## 相关页面

- 学习路线：[[interview/roadmap]]
- 速查卡：[[interview/cheatsheet]]
- 决策框架：[[analysis/when-to-use-wasm]]
