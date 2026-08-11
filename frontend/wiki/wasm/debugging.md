---
title: 调试与工具链
date: 2026-08-10
tags: [wasm, 调试, devtools, wabt, binaryen]
sources: [mdn-wasm/guides/using_the_javascript_api.md, mdn-wasm/guides/text_format_to_wasm.md, emscripten-doc/Optimizing-Code.rst, wasm-bindgen-doc/reference-debug-info.md, wasm-design/Tooling.md]
---

# 调试与工具链

## 三个层次的调试体验

| 层次 | 看到的东西 | 需要什么 |
| --- | --- | --- |
| **裸 Wasm** | WAT 反汇编，函数索引 | 什么都不需要 |
| **带 name section** | WAT + 真实函数名 | 编译时保留 name section |
| **源码级** | 原始 C/C++/Rust 源码，断点、变量 | **DWARF 调试信息** + DevTools 扩展 |

## 浏览器 DevTools

**Chrome**：安装 **C/C++ DevTools Support (DWARF)** 扩展后，
带 DWARF 信息的 Wasm 可以做到：

- 在原始 `.c` / `.rs` 源码上打断点
- 单步、查看调用栈
- 检查变量值（含结构体展开）
- 在 Console 里对源码表达式求值

**Firefox**：Debugger 面板里有 `wasm://` 条目，展示文本格式，可以在 WAT 层面打断点、
单步、看调用栈。

没有调试信息时，你看到的是这样：

```wat
(func $func23 (param i32 i32) (result i32)
  local.get 0
  ...
```

有 name section 时至少能看到 `$my_function`。所以：

> **面试落点**：**开发构建保留 name section 和 DWARF，生产构建 strip 掉**。
> name section 在大型项目里可能占产物体积相当大的比例。
> 更好的做法是**把调试信息拆成独立文件**按需加载（Wasm 支持外部 DWARF），
> 这正是官方 Tooling 文档说的"调试信息应该按需提供，而不是内建在模块里"。

## 生成调试信息

```bash
# Emscripten
emcc -g file.c              # 完整 DWARF
emcc -g2 file.c             # 只保留 name section
emcc -O2 --profiling file.c # 优化 + 保留函数名（给 profiler 用）

# 拆分调试信息到单独文件
emcc -g --emit-symbol-map -gseparate-dwarf=app.debug.wasm file.c
```

```bash
# Rust
cargo build --target wasm32-unknown-unknown   # debug 模式默认带调试信息
# release 想保留：Cargo.toml 里 [profile.release] debug = true
```

Rust 侧务必加 panic hook，否则 panic 只表现为一个 `unreachable` trap：

```rust
#[wasm_bindgen(start)]
pub fn start() {
    console_error_panic_hook::set_once();   // panic 变成可读的 console.error
}
```

## wabt —— WebAssembly Binary Toolkit

```bash
wat2wasm add.wat -o add.wasm            # 文本 → 二进制
wat2wasm --enable-multi-memory a.wat    # 某些 proposal 需显式开启

wasm2wat add.wasm --fold-exprs          # 二进制 → 文本（折叠形式更可读）
wasm2wat add.wasm --generate-names      # 没有 name section 时生成占位名

wasm-objdump -x app.wasm                # 全部 section 详情
wasm-objdump -h app.wasm                # 只看 section 头（快速看体积分布）
wasm-objdump -d app.wasm                # 反汇编

wasm-validate app.wasm                  # 验证
wasm-strip app.wasm                     # 去掉自定义 section
wasm-decompile app.wasm                 # 反编译成类 C 伪代码（可读性最好）
wasm-interp app.wasm                    # 命令行解释执行
```

`wasm-decompile` 值得单独提——它输出的伪代码可读性远好于 WAT，
分析第三方 `.wasm` 时很好用。

## Binaryen

Binaryen 是 **Wasm 层面的编译器基础设施**（Emscripten 和 wasm-pack 内部都在用）：

```bash
wasm-opt -Oz input.wasm -o output.wasm       # 极致压体积
wasm-opt -O3 input.wasm -o output.wasm       # 极致优化速度
wasm-opt --strip-debug --strip-producers input.wasm -o output.wasm

wasm-metadce ...      # 跨 JS/Wasm 边界的死代码消除
wasm-as / wasm-dis    # Binaryen 自己的汇编/反汇编
```

`wasm-opt` 做的是 **LLVM 做不到的全程序 Wasm 级优化**——它看到的是完整的 Wasm 模块，
可以跨编译单元内联、消除 LLVM 阶段丢失信息后残留的冗余。

> **⚠️ 注意**：Binaryen 的全程序优化可能做出让人意外的内联——
> LLVM IR 上的 `noinline` 属性到这个阶段已经丢失了。

## 体积分析

```bash
# Rust
twiggy top pkg/app_bg.wasm          # 谁最占体积
twiggy dominators pkg/app_bg.wasm   # 支配树：删掉某个东西能省多少
twiggy garbage pkg/app_bg.wasm      # 找到进不去的死代码

# 通用
wasm-objdump -h app.wasm            # section 级别的体积分布
```

典型的体积分布诊断：如果 `Custom` section（name/DWARF）占了一半，
说明忘了 strip；如果 `Code` section 里某几个函数占大头，去看是不是格式化/异常机制被链进来了。

## Emscripten 专属

```bash
EMCC_DEBUG=1 emcc ...       # 输出每个编译阶段的中间文件
emcc -sASSERTIONS=1 ...     # 运行时断言（越界、类型错误等，开发期必开）
emcc -sSAFE_HEAP=1 ...      # 更严格的内存访问检查（很慢，但能抓住细微越界）
emcc -sSTACK_OVERFLOW_CHECK=2 ...
emcc -fsanitize=address ... # ASan
emcc -fsanitize=undefined ... # UBSan
```

`-sASSERTIONS` 和 `-sSAFE_HEAP` 是排查"神秘内存问题"的第一梯队工具。

## Profiling

```bash
emcc -O2 --profiling file.cpp    # 优化 + 保留符号名
```

浏览器 Performance 面板的火焰图里能看到 Wasm 函数（前提是保留了符号）。

**必须多浏览器测**——各引擎的 Wasm 实现差异比 JS 大得多。
Emscripten 官方文档明确写着"在多个浏览器上 profile 是强烈建议的"，
且"如果一个浏览器上性能可接受、另一个上明显差，请提 bug"。

## 常见问题的排查路径

| 症状 | 先查什么 |
| --- | --- |
| `CompileError` | 二进制是否损坏；是否用了引擎不支持的 proposal（`wasm-validate` 一下） |
| `LinkError` | importObject 是否缺字段、类型对不对、memory 初始大小够不够 |
| `RuntimeError: unreachable` | Rust panic 或 C 的 `abort()`；上 panic hook / `-sASSERTIONS` |
| `RuntimeError: memory access out of bounds` | 指针算错或缓冲区大小算错；上 `-sSAFE_HEAP` |
| 数据读出来是 0 或乱码 | **`memory.grow` 之后视图 detached**，见 [[wasm/memory-model]] |
| `instantiateStreaming` 失败 | 服务器 `Content-Type` 不是 `application/wasm` |
| 多线程代码完全跑不起来 | 没配 COOP/COEP，`self.crossOriginIsolated` 是 false |
| 产物比预期大很多 | name/DWARF 没 strip；异常/RTTI/格式化机制被链进来 |

## 相关页面

- 文本格式：[[wasm/text-format-wat]]
- 体积与性能优化：[[wasm/performance]]
- 构建集成：[[wasm/build-integration]]
