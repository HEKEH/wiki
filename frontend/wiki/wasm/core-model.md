---
title: 核心执行模型 —— Module / Instance / 栈机 / 验证 / Trap
date: 2026-08-10
tags: [wasm, 执行模型, 栈机, 面试]
sources: [mdn-wasm/guides/concepts.md, mdn-wasm/guides/using_the_javascript_api.md, mdn-wasm/guides/understanding_the_text_format.md, wasm-design/Rationale.md, wasm-design/Security.md]
---

# 核心执行模型

Wasm 的整个概念体系只有**四个一等实体**，而且它们在 JS API 里 1:1 有对应对象。
这是最值得先记牢的一张表：

| 概念 | 是什么 | 状态 | JS 对应 |
| --- | --- | --- | --- |
| **Module** | 已编译的二进制代码，声明 imports 和 exports | **无状态** | `WebAssembly.Module` |
| **Instance** | Module + 它运行时用到的全部状态（memory、table、imports） | 有状态 | `WebAssembly.Instance` |
| **Memory** | 一段可增长的线性字节数组 | 有状态 | `WebAssembly.Memory` |
| **Table** | 可增长的**引用**数组（主要放函数引用） | 有状态 | `WebAssembly.Table` |

再加一个 **Global**（可跨模块 import/export 的全局变量，`WebAssembly.Global`）。

类比记忆：**Module ≈ ES module 的代码本身，Instance ≈ 这个 module 被载入某个 global
并绑定了一组 import 之后的产物**。就像"一个函数字面量可以产生 N 个闭包"，
一个 Module 可以实例化出 N 个 Instance。

> **面试落点**：Module 是无状态的，所以它像 `Blob` 一样能被 `postMessage()` 传给 Worker
> 共享——**编译一次，多个 Worker 复用**，这是多线程 Wasm 的关键优化点。
> 而 Instance 有状态，不能这样共享。

## 多重性（multiplicity）

- 1 个 Module → N 个 Instance
- 1 个 Instance → 0~1 个 Memory（Wasm 3.0 起可以 0~N，multi-memory）
- 1 个 Instance → 0~1 个 Table（现在也可以多个）
- 1 个 Memory/Table → 被 0~N 个 Instance 共用 ← **这是动态链接的基础**

多个 instance 共享同一 Memory + 同一 Table，就等于共享同一个"地址空间"，
类似原生程序里多个 `.dll` 共享一个进程地址空间。见 [[wasm/js-interop]] 里的动态链接示例。

## 栈式虚拟机（stack machine）

Wasm 的执行语义定义在一台**栈机**上：每条指令从操作数栈上弹出若干值、压回若干值。

```wat
(func (param $p i32) (result i32)
  local.get $p     ;; 压入 $p
  local.get $p     ;; 再压入 $p
  i32.add)         ;; 弹出 2 个 i32，压回它们的和
```

函数开始时栈为空，结束时栈上剩下的值就是返回值。

**为什么选栈机而不是 AST / 寄存器 / SSA？** 官方 Rationale 的三条理由：

1. 栈机编码比寄存器/SSA **更小**（不需要显式编码操作数位置）。
2. **结构化控制流**让验证可以单遍线性完成，不需要像早期 JVM 那样做不动点计算。
3. 可以直接解码进编译器的 SSA 内部表示，也便于基线编译器/解释器快速处理。

注意是**"结构化"栈机**，不是完全通用的栈机——控制流必须是 `block`/`loop`/`if` 嵌套结构，
跳转只能跳到外层 label，不能任意 goto。任何控制流（包括不可归约的）都能用
**Relooper 算法**转成结构化形式，代价可控。

> **面试落点**：栈机 + 结构化控制流 = **单遍线性验证**。这是 Wasm 能"边下载边编译"
> （streaming compilation）的前提，也是它比 JVM 字节码更容易被快速验证的原因。

### 局部变量不在栈上

Wasm 有**两套"栈"**，这是一个经典考点：

| | 位置 | 谁能访问 | 用途 |
| --- | --- | --- | --- |
| **操作数栈 + 调用栈** | VM 内部，**不在线性内存里** | 只有 VM | 保存返回地址、溢出的局部变量 |
| **"aliased stack"** | 线性内存里，由编译器（LLVM）维护 | Wasm 代码可读写 | 被取地址的变量、返回值为 struct 的变量 |

局部变量（`local.get`/`local.set`）按索引访问，**位于地址空间之外**，
所以缓冲区溢出无法覆盖它们，返回地址也无法被篡改——这是 Wasm 天然免疫 ROP 攻击的根本原因。
但 C 里对局部变量取地址（`&x`）的情况，编译器必须把它放进线性内存的那个"影子栈"上，
那部分就**不再受保护**了。

> **面试落点**：Wasm 里"栈溢出攻击"打不穿控制流，因为**返回地址在 VM 保护的调用栈上，
> 不在程序可寻址的线性内存里**。但线性内存内部的对象之间仍然可以互相越界覆写。

## 验证（validation）

模块在编译前必须通过**静态验证**，失败抛 `WebAssembly.CompileError`。验证保证：

- 每条指令的操作数类型匹配，函数结束时栈上恰好是声明的返回类型
- 所有分支目标都是有效的、在当前函数内的 label
- 所有索引（函数、局部变量、global、table、memory）都在各自 index space 范围内

一个细节考点：**跳转之后的代码是"多态栈类型"的**。
`br`、`return`、`unreachable` 之后的指令不可达，此时验证器允许假设栈是任意类型。
这不是偷懒，而是为了保证类型系统的**可组合性**——不然编译器做个简单优化
（把 `x/0` 直接编成 `br $div_zero`）就会产出无法验证的代码。

## Trap

**Trap = 立即终止执行并向宿主报错**。在浏览器里表现为抛出 `WebAssembly.RuntimeError`。

会 trap 的操作：

- 线性内存越界访问
- `call_indirect` 的签名与 table 里实际函数不匹配
- table 索引越界
- 整数除以 0、有符号除法溢出（`INT_MIN / -1`）
- 调用栈耗尽

> **⚠️ 常见误答**："Wasm 越界会读到随机内存" —— 不会。**越界一定 trap**，
> 这是沙箱的硬保证。真正的风险是**线性内存内部**的越界：模块自己的堆里，
> 对象 A 可以覆写相邻对象 B，因为边界检查只在整块线性内存的边界上做。

## 三类错误对象

| 错误 | 何时抛 | 典型原因 |
| --- | --- | --- |
| `CompileError` | 解码 / 验证阶段 | 二进制损坏、类型不匹配、用了引擎不支持的 proposal |
| `LinkError` | 实例化阶段 | importObject 里缺字段、类型对不上、memory 初始大小不满足 |
| `RuntimeError` | 运行阶段 | trap（越界、除零、签名不匹配、栈溢出） |

> **面试落点**：能把这三个错误按**编译 → 实例化 → 运行**三个阶段对应上，
> 就说明你真的理解了 Wasm 的生命周期。调试时看到 `LinkError` 就该去查 importObject。

## Index space（索引空间）

Wasm 里一切都靠**索引**引用，不靠名字。每类实体有独立的索引空间：
function / table / memory / global / type / elem / data。

**导入的实体排在前面**，然后才是模块内定义的。例如模块导入了 2 个函数、自己定义了 3 个，
那么函数索引 0、1 是导入的，2、3、4 是自己的。

文本格式里可以用 `$name` 起名字，但那只是给人看的语法糖——**编译成二进制后只剩整数索引**。
这也是"二进制格式比文本快 23 倍解码"的原因之一：数组下标 vs 字典查找。

## 相关页面

- 线性内存细节：[[wasm/memory-model]]
- 类型系统：[[wasm/types-and-abi]]
- 文本格式语法：[[wasm/text-format-wat]]
- JS 侧 API：[[wasm/js-interop]]
- 安全模型：[[wasm/security-sandbox]]
