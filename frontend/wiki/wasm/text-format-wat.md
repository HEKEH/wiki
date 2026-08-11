---
title: 文本格式 WAT —— 读懂与手写
date: 2026-08-10
tags: [wasm, wat, 文本格式, 面试]
sources: [mdn-wasm/guides/understanding_the_text_format.md, mdn-wasm/guides/text_format_to_wasm.md, mdn-wasm/reference/control_flow.md, mdn-wasm/reference/numeric.md, wasm-design/FAQ.md]
---

# 文本格式 WAT

WAT（WebAssembly Text format）与二进制格式**1:1 对应**，用 S-表达式书写。
它存在的意义是 View Source、devtools 展示、手写测试用例、以及写编译器时肉眼验证输出。

面试里不会让你手写复杂 Wasm，但**能读懂一段 WAT 并解释它在干什么**是明确的加分项。

## 最小模块

```wat
(module)
```

编译出来就是 8 字节文件头：

```plain
0000000: 0061 736d    ; magic "\0asm"
0000004: 0100 0000    ; version 1
```

> **面试落点**：Wasm 二进制的 magic number 是 `\0asm`（`0x00 0x61 0x73 0x6D`），
> 后跟 4 字节版本号 `1`。**至今仍是 1**——Wasm 靠 feature detection 演进，不靠版本号，
> 所以 "Wasm 3.0" 指的是 spec 版本，不是二进制头里的版本。

## 函数：签名、局部变量、函数体

```wat
( func <signature> <locals> <body> )
```

```wat
(module
  (func $add (param $lhs i32) (param $rhs i32) (result i32)
    local.get $lhs
    local.get $rhs
    i32.add)
  (export "add" (func $add)))
```

要点：

- 参数本质就是"被调用方实参初始化了的局部变量"。`local.get 0` 先数参数、再数 locals。
- `$name` 只是可读性糖，二进制里只有整数索引。
- 没有 `(result)` 就是无返回值。**Wasm 2.0 起支持多返回值**（multi-value），
  写成 `(result i32 i32)`。早期文档里"最多 1 个返回值"的说法已过时。
- `(export "add" (func $add))` 可以简写成内联形式 `(func (export "add") ...)`。

JS 侧调用：

```js
const { instance } = await WebAssembly.instantiateStreaming(fetch("add.wasm"));
console.log(instance.exports.add(1, 2)); // 3
```

## 两种等价写法：线性 vs 折叠

同一段代码有两种写法，读别人的 WAT 时两种都会碰到：

```wat
;; 线性（栈式）写法——更贴近二进制
i32.const 0
i32.const 42
i32.store

;; 折叠（S-表达式）写法——更像常规语言
(i32.store (i32.const 0) (i32.const 42))
```

折叠写法里，**子表达式先求值并压栈**，外层指令再消费它们。工具（wabt 的 `wasm2wat`）
默认输出折叠形式，可读性更好。

## Import：两级命名空间

```wat
(module
  (import "console" "log" (func $log (param i32)))
  (func (export "logIt")
    i32.const 13
    call $log))
```

Wasm 的 import 是**两级命名空间** `模块名.字段名`，对应 JS 里的嵌套对象：

```js
const importObject = { console: { log: (arg) => console.log(arg) } };
await WebAssembly.instantiateStreaming(fetch("logger.wasm"), importObject);
```

> **⚠️ 常见误答**："导入的 JS 函数会被类型检查" —— **不会**。JS 函数没有签名概念，
> 任何 JS 函数都能满足任何 import 声明。Wasm 只在**内部**静态检查 import 的签名，
> 实际传进来的 JS 函数不做校验；参数按声明的类型做转换。

## Global

```wat
(module
  (global $g (import "js" "global") (mut i32))
  (func (export "getGlobal") (result i32)
    (global.get $g))
  (func (export "incGlobal")
    (global.set $g (i32.add (global.get $g) (i32.const 1)))))
```

`(mut i32)` 表示可变。JS 侧：

```js
const global = new WebAssembly.Global({ value: "i32", mutable: true }, 0);
// 双向可见：JS 改 global.value，wasm 能读到；wasm 改，JS 也能读到
```

## Memory 与 data 段

```wat
(module
  (import "console" "log" (func $log (param i32 i32)))  ;; offset, length
  (import "js" "mem" (memory 1))                        ;; 至少 1 页 = 64KiB
  (data (i32.const 0) "Hi")                             ;; 实例化时写入偏移 0
  (func (export "writeHi")
    i32.const 0    ;; offset
    i32.const 2    ;; length
    call $log))
```

JS 侧把 `(offset, length)` 还原成字符串——**这就是 Wasm 传字符串的本质**：

```js
const memory = new WebAssembly.Memory({ initial: 1 });
function consoleLogString(offset, length) {
  const bytes = new Uint8Array(memory.buffer, offset, length);
  console.log(new TextDecoder("utf8").decode(bytes));
}
await WebAssembly.instantiateStreaming(fetch("logger2.wasm"), {
  console: { log: consoleLogString },
  js: { mem: memory },
});
```

> **面试落点**：**Wasm 没有字符串类型**。跨边界传字符串 = 传"线性内存里的偏移 + 长度"，
> 由 JS 侧用 `TextDecoder` 解码。这是所有 Wasm 胶水代码（wasm-bindgen、Emscripten）
> 都在自动化的事情。详见 [[wasm/types-and-abi]]。

## Table 与 call_indirect —— 最容易被问倒的部分

**为什么需要 table？** `call` 指令的函数索引是**静态立即数**，只能调固定的函数。
但 C 有函数指针、C++ 有虚函数、JS 有一等函数，都需要**运行时决定调谁**。

那为什么不直接给一个 `funcref` 类型的操作数？因为**引用不能存进线性内存**——
线性内存的字节内容对模块完全可见可改，把真实函数地址暴露进去，
既泄露地址信息，又能被伪造成任意跳转目标，沙箱就破了。

**解法**：函数引用存在一张 table 里，代码里传递的是 **i32 索引**。索引可以安全地放进线性内存。

```wat
(module
  (table 2 funcref)
  (func $f1 (result i32) i32.const 42)
  (func $f2 (result i32) i32.const 13)
  (elem (i32.const 0) $f1 $f2)              ;; 从索引 0 开始填入两个函数
  (type $return_i32 (func (result i32)))
  (func (export "callByIndex") (param $i i32) (result i32)
    local.get $i
    call_indirect (type $return_i32)))
```

```js
const { instance } = await WebAssembly.instantiateStreaming(fetch("wasm-table.wasm"));
instance.exports.callByIndex(0); // 42
instance.exports.callByIndex(1); // 13
instance.exports.callByIndex(2); // RuntimeError：越界
```

`call_indirect` 每次调用做**两次动态检查**：

1. **边界检查**：索引是否在 table 范围内 → 越界则 trap
2. **签名检查**：table 里那个函数的类型是否与调用点声明的 `(type ...)` 一致 → 不一致则 trap

> **面试落点**：`call_indirect` 的两次检查是 Wasm **控制流完整性（CFI）** 的核心。
> 它让"用错误签名调函数"无法破坏沙箱安全性，代价是每次间接调用有固定开销。
> 这也是为什么大量虚函数调用的 C++ 代码在 Wasm 上相对原生有可见的性能损失。

`elem` 段之于 table，就像 `data` 段之于 memory：实例化时初始化一段区域。
未初始化的 table 元素默认是"调用即 trap"。

> **⚠️ 注意源文档的过时说法**：MDN 里写着"目前每个模块实例只允许一张 table，
> 所以 `call_indirect` 隐式调用它"——**已过时**。reference-types（Wasm 2.0）之后支持多 table，
> `call_indirect` 可以显式指定：`call_indirect $my_table (type $sig)`。
> 省略时默认 table 0，所以老代码不受影响。

## 结构化控制流

```wat
block   ;; 建立一个 label，br 跳到它 = 跳到 block 末尾（向前跳，等于 break）
loop    ;; 建立一个 label，br 跳到它 = 跳回 loop 开头（等于 continue）
if / else / end
br      / br_if / br_table     ;; 无条件 / 有条件 / 跳转表（等价 switch）
return  / call / call_indirect
drop    ;; 丢弃栈顶
select  ;; 三元运算符
nop     / unreachable
```

关键直觉：**`br $label` 跳到 `block` 是往后跳（break），跳到 `loop` 是往前跳（continue）**。
初学者最容易在这里搞反。

`br` 及其变体可以**携带栈上的值**，这样 `if` 就能当表达式用，
省掉大量 `local.set`/`local.get` 对——早期实验里这类指令占了总字节数的 30–40%。

## 数值指令的命名规律

记住命名模式就不用背指令表：

```text
<type>.<op>[_<signedness>]

i32.add          有符号无关（补码魔法：add/sub/mul 不区分符号）
i32.div_s        有符号除
i32.div_u        无符号除
i32.lt_s         有符号小于
f64.sqrt         浮点专属，无符号后缀
i64.extend_i32_s 类型转换：i32 符号扩展成 i64
i32.wrap_i64     截断：i64 → i32
i32.trunc_f64_s  浮点转有符号整数（会 trap；非陷阱版本是 i32.trunc_sat_f64_s）
```

几个语义陷阱（都来自官方 Rationale）：

- **有符号除法向零取整**（跟 C 一致，不是向下取整）。
- **移位次数会被取模**：`i32.shl` 移 32 位等于不移。所以左移**不等价于**乘 2 的幂。
- `add`/`sub`/`mul` 溢出时**回绕不 trap**；`div`/`rem` 除零和 `INT_MIN/-1` 会 **trap**。
- **NaN 的位模式是不确定的**（x86 和 ARM 行为不同，规范不强制规范化）。

## 工具链

```bash
# wabt: 文本 ↔ 二进制
wat2wasm add.wat -o add.wasm
wasm2wat add.wasm -o add.wat --fold-exprs
wasm-objdump -x add.wasm       # 查看各 section
wasm-validate add.wasm

# binaryen: 优化
wasm-opt -Oz input.wasm -o output.wasm
```

某些 proposal 在 wabt 里还需显式开关，例如多内存要 `wat2wasm --enable-multi-memory`。

## 相关页面

- 执行模型：[[wasm/core-model]]
- 类型与跨边界转换：[[wasm/types-and-abi]]
- 线性内存：[[wasm/memory-model]]
- 调试与工具：[[wasm/debugging]]
