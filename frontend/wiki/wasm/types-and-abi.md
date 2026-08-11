---
title: 类型系统与跨边界 ABI
date: 2026-08-10
tags: [wasm, 类型, abi, externref, 面试]
sources: [mdn-wasm/reference/value_types.md, mdn-wasm/guides/exported_functions.md, mdn-wasm/guides/understanding_the_text_format.md, mdn-wasm/guides/imported_string_constants.md, wasm-design/Rationale.md, wasm-design/CAndC++.md]
---

# 类型系统与跨边界 ABI

## 全部值类型

Wasm 的类型系统小得惊人。这不是能力不足，而是**刻意的设计**：Wasm 把自己定位成
"虚拟 ISA"，复杂类型由源语言编译器自己用基本类型拼出来。

### 数值类型（number types）

| 类型 | 含义 |
| --- | --- |
| `i32` | 32 位整数，**有符号/无符号由指令决定**，类型本身不带符号信息 |
| `i64` | 64 位整数，同上 |
| `f32` | IEEE 754 单精度 |
| `f64` | IEEE 754 双精度 |

### 向量类型

| 类型 | 含义 |
| --- | --- |
| `v128` | 128 位向量，由 SIMD 指令按 `i8x16`/`i16x8`/`i32x4`/`f32x4`/`f64x2` 等形态解释 |

### 引用类型（reference types）

| 类型 | 含义 |
| --- | --- |
| `funcref` | Wasm 函数引用 |
| `externref` | **宿主值的不透明引用**（DOM 节点、JS 对象、字符串……） |
| `exnref` | 一个被抛出的异常（Wasm 3.0 异常处理） |

Wasm 3.0 的 GC proposal 还引入了 `anyref`/`eqref`/`structref`/`arrayref` 等一整套堆类型。

### 为什么没有 i8 / i16 / f16 / i128？

官方 Rationale：

- `i8`/`i16` 在算术上会被提升到 `i32`，只在内存访问时有语义意义——所以用
  `i32.load8_u` 这类**带宽度的 load/store 指令**解决就够了，不必单列类型。
- `f16`/`i128` 硬件支持参差不齐（有的只支持 load/store，有的只支持 SIMD 运算），
  留给运行库或后续 proposal（FP16 目前在 Phase 2）。
- 所有列出的类型都能**直接映射到现代 CPU 的原生类型**，这是"接近原生速度"的前提。

> **面试落点**："Wasm 只有 4 个数值类型"背后是一条设计原则：
> **Wasm 提供原语，复杂类型由源语言编译器构造**。字符串、结构体、类、闭包——
> 在 core wasm 里全都不存在，全靠线性内存里的字节布局 + 编译器约定实现。

### 为什么没有字符串、没有对象？

**核心理由：这两样东西不存在一个"中立"的定义。**

| 语言 | "字符串"是什么 |
| --- | --- |
| C | NUL 结尾的 `char*`，不带长度 |
| C++ | `std::string`，带 SSO 的堆对象 |
| Rust | `String` = UTF-8 的 `Vec<u8>`；`&str` 是胖指针 (ptr, len) |
| Java / C# / JS | UTF-16 码元序列，**不可变** |
| Go | 不可变字节切片 (ptr, len) |
| Swift | UTF-8，但索引语义是 grapheme cluster |

编码不同、可变性不同、带不带长度前缀不同、索引单位不同（字节 / 码元 / 字素）。
Wasm 只要内建**一种**，就等于选边站：其它语言要么在每次跨边界时付编码转换的代价，
要么干脆绕开不用——那这个内建类型就成了死重量。

对象更极端：C++ 多继承 + vtable、Java 单继承 + 接口 + 反射、JS 原型链、
Rust trait object 胖指针、Go 结构嵌入 + 接口表——内存布局和方法分派规则互不兼容，
**没有哪一种能当公约数**。而 High-Level Goals 明确要求**不偏向任何语言族**。

另外三条同样成立的理由：

1. **Wasm 的定位是虚拟 ISA，不是运行时。** x86 也没有字符串类型。ISA 给的是原语，
   "字符串"属于库和编译器约定的层面——和上面"为什么没有 i8/i16"是同一条原则。
2. **有对象就必须有 GC，而 MVP 阶段 GC 是被明确排除的。** 真正的对象需要生命周期管理：
   要么手动（那就退化成线性内存里的指针，等于没内建），要么引擎 GC。MVP 就内建 GC 意味着
   **每个 embedder**（含嵌入式、服务端的极小运行时）都得实现一个 GC，实现门槛暴涨；
   而且 GC 语义（终结器、弱引用、循环收集）本身就是个巨大的设计空间。
3. **类型系统越小，单遍验证越快、安全性越容易证明。**

> **面试落点**：**"没有对象"和"Wasm 3.0 加了 GC"是同一个问题的两端。**
> MVP 拒绝选任何语言的对象模型 → GC 语言只能把整个 GC 运行时编译进线性内存 →
> 产物暴涨、两个堆互相看不见（官方 FAQ：这些对象活在 *a walled-off world of their own*）、
> 跨堆循环引用回收不掉。而 GC proposal 的解法**并没有背弃语言中立**——它没内建
> "Java 对象"，而是给了一组更低层的原语（struct / array / i31 / 子类型），
> 让各语言编译器在这之上拼出自己的对象模型。
> **它交出去的是"内存管理"这一层，不是"对象模型"这一层。**
> 字符串同理：JS String Builtins 不是"Wasm 有字符串类型了"，字符串仍然是 `externref`，
> 只是多了一组操作它的内建导入——规范层面 Wasm 至今没有字符串类型。
> 详见 [[wasm/proposals-and-versions]]。

## externref：唯一能安全持有宿主对象的方式

`externref` 从 Wasm 的角度是**完全不透明**的：模块能接收它、存它、传回去，
但**不能读它的内容、不能伪造它**。

```wat
(module
  (global $obj (import "js" "someObject") externref)
  (func (export "passBack") (result externref)
    global.get $obj))
```

为什么必须不透明？和 table 的理由一样：如果宿主对象的真实指针能被 Wasm 代码读到，
沙箱就破了。`externref` 只能存在**值栈、局部变量、global、table** 里，
**永远不能存进线性内存**。

在 `externref` 之前，传 DOM 引用只能靠一张 JS 侧的对象表 + 传整数 handle，
外加手工引用计数（wasm-bindgen 早年就是这么干的）。`externref` 让引擎的 GC 直接接管生命周期。

## i64 ↔ BigInt：最经典的边界陷阱

```js
// 导出函数签名里含 i64 时
instance.exports.takesI64(123);      // ❌ TypeError：需要 BigInt
instance.exports.takesI64(123n);     // ✅
const r = instance.exports.returnsI64(); // 得到 BigInt，不是 Number
```

**JS 的 Number 是 f64，无法精确表示全部 64 位整数**，所以 Wasm 的
JS-BigInt-integration（Wasm 2.0 已并入 spec）规定：i64 在边界上映射为 **BigInt**。

历史包袱：更早的引擎在遇到 i64 参数/返回值时直接抛错。Emscripten 为此提供了
"legalization"——把一个 i64 拆成两个 i32 传递，代价是链接期要改写 Wasm。
用 `-sWASM_BIGINT` 可以关掉 legalization，链接更快、也没有拆分开销。

> **面试落点**：i64 跨边界要用 BigInt，而 BigInt 装箱/拆箱有成本。
> **高频调用的热路径上尽量用 i32/f64 而不是 i64**，这是真实可测的优化点。

## JS ↔ Wasm 的类型映射全表

| Wasm 类型 | 传入（JS → Wasm） | 传出（Wasm → JS） |
| --- | --- | --- |
| `i32` | `ToInt32(value)`，即等价 `value \| 0` | Number |
| `i64` | 必须是 BigInt，`ToBigInt64` | BigInt |
| `f32` | ToNumber 后收窄到单精度 | Number（可能有精度损失） |
| `f64` | ToNumber | Number |
| `externref` | 任意 JS 值原样传入 | 原样传出 |
| `funcref` | 必须是 exported wasm function 或 null | 一个 JS 函数包装 |
| `v128` | **不能跨边界** | 不能跨边界 |

> **⚠️ 常见误答**："SIMD 的 v128 可以传给 JS" —— 不能。`v128` 在 JS API 边界上是非法的，
> 必须通过线性内存交换向量数据。

## 导出函数在 JS 里是什么

`instance.exports.foo` 是一个**真正的 JS 函数**（`typeof === "function"`，能 `.call()`/`.bind()`），
只不过是对 Wasm 函数的包装：

```js
const f = instance.exports.tbl.get(0);
typeof f;         // "function"
f.toString();     // "function 0() { [native code] }"
f.length;         // Wasm 签名里声明的参数个数
f.name;           // 该函数在模块里的索引的字符串形式，如 "0"
```

调用时发生的事：JS 值 → 按签名转换成 Wasm 值 → 进入 Wasm → 返回值转换回 JS 值。

## 复合数据怎么跨边界：三种模式

core wasm 只能传数值和引用，所以所有复杂数据都要编解码。三种主流做法：

### 1. 指针 + 长度（最底层，最快）

```js
// Wasm 侧 export 一个 alloc(size) -> ptr
const ptr = instance.exports.alloc(bytes.length);
new Uint8Array(instance.exports.memory.buffer, ptr, bytes.length).set(bytes);
instance.exports.process(ptr, bytes.length);
instance.exports.dealloc(ptr, bytes.length);
```

**零拷贝的关键**：`new Uint8Array(buffer, ptr, len)` 是**视图**（不拷贝），
而 `new Uint8Array(typedArray)` 是**拷贝**。这一个字之差是高频面试题。

### 2. 胶水代码自动生成（wasm-bindgen / Embind）

你写 `#[wasm_bindgen] fn greet(name: &str)`，工具生成的胶水在 JS 侧做
`TextEncoder` 编码 → 调 Wasm 的 `__wbindgen_malloc` → 写内存 → 传 `(ptr, len)` → 调用 →
释放。见 [[wasm/toolchain-rust]]。

### 3. JS String Builtins（Wasm 3.0 新增，面向 GC 语言）

给 Wasm 一组内建的字符串操作导入，让编译到 Wasm 的 Java/Kotlin/Dart
可以**直接操作 JS 字符串**（表现为 `externref`），不必在线性内存里重造一套字符串实现。

配套的还有 **imported global string constants**：以前每个字符串常量都要在 Wasm 里
声明一个 import、在 JS importObject 里写一份值，成千上万个常量会显著撑大体积。
现在只要在编译时传一个命名空间：

```js
WebAssembly.instantiate(bytes, importObject, {
  builtins: ["js-string"],
  importedStringConstants: "",  // 生产环境用空字符串省体积
});
```

引擎会自动把该命名空间下每个 import 的**名字本身**当作字符串值填进去。

```wat
(global $h (import "" "hello ") externref)
(global $w (import "" "world!") externref)
```

> **面试落点**：JS String Builtins + imported string constants 是 Wasm 3.0 里
> **专为 GC 语言（Java/Kotlin/Dart/Scala.js）铺路**的一环——目标是让这些语言编译到 Wasm 时，
> 字符串直接复用 JS 引擎的实现，而不是自带一份。

## C/C++ 的数据模型

| 目标 | 数据模型 | `int` | `long` | 指针 |
| --- | --- | --- | --- | --- |
| wasm32 | ILP32 | 32 | 32 | 32 |
| wasm64 | LP64 | 32 | 64 | 64 |

- `float`/`double` 原生对应 `f32`/`f64`。
- `long double`（IEEE 四精度）**没有原生支持**，靠软件模拟，慢且撑大体积——移植时要留意。
- **wasm32 上指针是 32 位**，所以线性内存上限 4GiB。需要更大就要 memory64（Wasm 3.0 已进 spec），
  代价是每个指针 8 字节，缓存和内存带宽利用率下降（与 Linux x32 ABI 的权衡同理）。

> **面试落点**：为什么不干脆全用 64 位指针？因为**绝大多数应用用不到 4GiB**，
> 强制 8 字节指针会显著增加内存占用、降低 cache 命中率。这就是 wasm32/wasm64 分开的理由。

## Undefined behavior 仍然存在

一个高区分度的点：**Wasm 定义了行为，不代表 C/C++ 层面的 UB 消失了**。

例如非对齐访问在 **Wasm 层面是完全定义的**，但 C/C++ 编译器仍然假设对齐规则被遵守，
会基于这个假设做优化——所以源码里的非对齐访问依然是 bug，依然可能产生意外行为。

Wasm 保证的只是：**UB 不会突破沙箱**。不会执行任意代码，不会腐蚀调用栈。
但它完全可能腐蚀自己的线性内存、用任意参数调 import、挂死、trap、耗尽资源。

## 相关页面

- 线性内存与零拷贝：[[wasm/memory-model]]
- JS API 全貌：[[wasm/js-interop]]
- Rust 工具链如何自动化这些：[[wasm/toolchain-rust]]
- 版本与 proposal：[[wasm/proposals-and-versions]]
