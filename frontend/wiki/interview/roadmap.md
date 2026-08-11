---
title: WebAssembly 速成路线（面试导向）
date: 2026-08-10
tags: [wasm, 面试, 路线图]
sources: []
---

# WebAssembly 速成路线

目标：**从"听说过"到"能在中高级前端面试里把 Wasm 这块聊透"**。

按投入时间给三条路径，从上到下递进。

## 路径 A：2 小时应急（明天就面）

只读这些，只背这些：

1. **[[wasm/what-is-wasm]]** —— 全文。重点：一句话定义、四个设计目标、
   为什么不用 asm.js（23× 解码速度）、版本时间线、三个常见误解
2. **[[wasm/core-model]]** 的前半 —— Module/Instance/Memory/Table 四件套，
   Module 无状态可 postMessage
3. **[[wasm/js-interop]]** 的加载部分 —— `instantiateStreaming` 为什么最快
4. **[[wasm/memory-model]]** 的 grow/detach 部分 —— 这是最实用的一个坑
5. **[[analysis/when-to-use-wasm]]** —— 面试怎么答那一节，**背下来**
6. **[[interview/cheatsheet]]** —— 全文扫一遍

能答出的问题：什么是 Wasm、和 JS 什么关系、什么时候用、怎么加载、
Wasm 传字符串怎么传。

## 路径 B：一周扎实（正常准备）

### Day 1：定位与核心模型

- 读 [[wasm/what-is-wasm]] + [[wasm/core-model]]
- **动手**：装 wabt，把一个 `add.wat` 编译成 `.wasm`，在浏览器里跑起来

```bash
brew install wabt   # 或 npm i -g wabt
cat > add.wat <<'EOF'
(module
  (func $add (param $a i32) (param $b i32) (result i32)
    local.get $a
    local.get $b
    i32.add)
  (export "add" (func $add)))
EOF
wat2wasm add.wat -o add.wasm
```

```js
const { instance } = await WebAssembly.instantiateStreaming(fetch("add.wasm"));
console.log(instance.exports.add(1, 2));  // 3
```

### Day 2：文本格式与类型系统

- 读 [[wasm/text-format-wat]] + [[wasm/types-and-abi]]
- **动手**：写一个 `call_indirect` 的例子，故意用错签名，观察 `RuntimeError`
- **重点理解**：为什么函数引用不能存进线性内存

### Day 3：内存模型与 JS 互操作

- 读 [[wasm/memory-model]] + [[wasm/js-interop]]
- **动手**：写一个"Wasm 里存字符串，JS 用 TextDecoder 读出来"的完整例子
- **动手**：故意触发一次 `memory.grow`，观察缓存的 TypedArray 视图失效

### Day 4：工具链

- 读 [[wasm/toolchain-rust]]（重点）+ [[wasm/toolchain-emscripten]]（了解）
- **动手**：跑通 `wasm-pack build --target web` 的 hello world
- **重点理解**：`--target bundler` 和 `--target web` 的区别

### Day 5：性能与决策

- 读 [[wasm/performance]] + [[analysis/when-to-use-wasm]]
- **动手**：写一个"JS 版 vs Wasm 版"的数组求和对比，
  分别测**细粒度调用**和**粗粒度调用**两种接口设计 —— 亲眼看到边界开销

### Day 6：工程化

- 读 [[wasm/build-integration]] + [[wasm/threads-and-workers]] + [[wasm/debugging]]
- **动手**：在一个 Vite 项目里集成上面的 Wasm 模块
- **重点理解**：COOP/COEP 的部署代价

### Day 7：查漏 + 演练

- 读 [[wasm/security-sandbox]] + [[wasm/proposals-and-versions]] + [[wasm/use-cases]]
- 过一遍 [[interview/question-bank]]，**自己口述**答案，别只在心里想
- 扫 [[interview/cheatsheet]]

## 路径 C：真正掌握（有项目要做）

在路径 B 之上：

1. **做一个完整的小项目**——推荐：图片格式转换器（canvas → Wasm 编码 → 下载）。
   它把内存传递、零拷贝视图、体积优化、按需加载全都涵盖了。
2. **读一遍真实项目的产物**：`wasm-decompile` 一个 npm 上的 Wasm 包，看它长什么样
3. **读 spec 的 core/exec 部分**：<https://webassembly.github.io/spec/core/>
4. **看 raw/ 里的原始文档**：`raw/wasm-design/Rationale.md` 是理解"为什么这么设计"的最佳材料
5. **关注 proposal 进展**：<https://webassembly.org/features/> 看引擎支持矩阵

## 按面试轮次准备什么

| 轮次 | 大概率问什么 | 重点准备 |
| --- | --- | --- |
| **一面（基础）** | 是什么、和 JS 关系、用过吗 | [[wasm/what-is-wasm]] |
| **二面（深入）** | 内存模型、怎么传数据、性能特征 | [[wasm/memory-model]]、[[wasm/types-and-abi]]、[[wasm/performance]] |
| **三面（架构）** | 什么时候该用、怎么落地、有什么坑 | [[analysis/when-to-use-wasm]]、[[wasm/build-integration]] |
| **加分项** | Wasm 3.0 有什么、GC 意味着什么、服务端 Wasm | [[wasm/proposals-and-versions]]、[[wasm/beyond-browser]] |

## 最容易被问倒的五个点

按被问倒的概率排序，**这五个一定要能脱口而出**：

1. **`memory.grow` 会 detach ArrayBuffer** —— 缓存的视图全失效
2. **为什么函数引用要放 table，不能放线性内存** —— 安全（地址泄露 + 伪造跳转）
3. **跨边界开销和粗粒度接口设计** —— Wasm 优化的第一原则
4. **Wasm 不能直接访问 DOM** —— 必须绕道 JS
5. **"编译成 Wasm 就内存安全了"是错的** —— 沙箱保护的是宿主，不是模块自己的堆

## 相关页面

- 题库：[[interview/question-bank]]
- 速查：[[interview/cheatsheet]]
- 全库导航：[[index]]
