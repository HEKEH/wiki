---
title: 源材料 —— WebAssembly/design 仓库
date: 2026-08-10
tags: [源材料, 设计文档, rationale]
sources: [wasm-design/HighLevelGoals.md, wasm-design/Rationale.md, wasm-design/Security.md, wasm-design/Nondeterminism.md, wasm-design/Portability.md, wasm-design/FAQ.md, wasm-design/CAndC++.md, wasm-design/DynamicLinking.md, wasm-design/UseCases.md, wasm-design/Tooling.md]
---

# 源材料：WebAssembly/design 仓库

**位置**：`raw/wasm-design/`（21 个文件）
**来源**：`github.com/WebAssembly/design` + `github.com/WebAssembly/proposals`
**抓取日期**：2026-08-10

## 这份材料是什么

Wasm 的**原始设计文档仓库**。它不是规范（规范在 `WebAssembly/spec`），
而是记录**"为什么这样设计"**的地方。

> **这是本库深度题的主要来源。** MDN 教你怎么用，design 仓库告诉你为什么。

## 最有价值的四篇

### `Rationale.md`（612 行）—— 必读

逐条解释设计决策：

- 为什么选栈机（不是 AST / 寄存器 / SSA）
- 为什么是**结构化**栈机（单遍验证、避免不动点计算）
- 为什么只有少数基本类型（i8/i16/f16/i128 为什么不要）
- **为什么用 table 做间接调用**（函数指针要能存进线性内存 + 安全）
- 为什么页大小固定、maximum 为什么可选（四条互相冲突的约束）
- 为什么二进制编码（23× 解码速度、20–30% 体积）
- **为什么 NaN 位模式不确定**（x86 vs ARM 行为差异的具体举例）
- 为什么整数除法向零取整、移位次数取模
- **为什么跳转之后是"多态栈类型"**（可组合性 + 编译器优化不会产生非法代码）

### `Security.md`（186 行）—— 必读

安全模型的完整论述：两层目标（保护用户 / 帮助开发者）、
**控制流完整性的三类边**（forward-edge ×2、back-edge）、
变量落位的两种情况、内存安全**保护了什么和没保护什么**、
Clang/LLVM CFI（`-fsanitize=cfi`）。

**"Wasm 没消灭什么"那一段是面试深度题的金矿**：
函数粒度的代码重用攻击、TOCTOU、侧信道，都在这里。

### `Nondeterminism.md`（56 行）

列全了 Wasm 允许不确定性的**七个来源**，以及 "limited, local" 的设计原则。
短但信息密度极高。

### `FAQ.md`（397 行）

回答了一堆经典问题：为什么不用 asm.js、为什么不用 LLVM bitcode、
是不是要取代 JS、为什么没有 fast-math、mmap 怎么办、
**为什么分 wasm32/wasm64 而不是统一 64 位指针**（Knuth 的"64 位指针之怒"都引用了）。

## 其余文件

| 文件 | 内容 | 价值 |
| --- | --- | --- |
| `HighLevelGoals.md` | 五条官方目标 | 高，简短必读 |
| `Portability.md` | 对宿主平台的硬性假设清单 | 高（小端序、无锁原子、前进保证） |
| `CAndC++.md` | wasm32/wasm64 数据模型、**UB 在 Wasm 里依然是 UB** | 高 |
| `DynamicLinking.md` | 共享 memory/table 实现 dlopen/dlsym | 中 |
| `UseCases.md` | 官方设想的用例清单 | 中 |
| `Tooling.md` | 期望的工具生态（**调试信息应按需提供**） | 中 |
| `FeatureTest.md` | 特性检测的动机场景 | 中 |
| `NonWeb.md` | 非浏览器嵌入 | 中 |
| `MVP.md` `Semantics.md` `Modules.md` `BinaryEncoding.md` `JS.md` `TextFormat.md` | **已被掏空**，只剩一句"见规范文档" | 无 |

`proposals-README.md` 和 `proposals-finished.md` 来自 `WebAssembly/proposals` 仓库，
是**proposal 阶段的权威名单**——[[wasm/proposals-and-versions]] 整页基于它。

## ⚠️ 时效性：这是最需要注意的一点

**design 仓库的多数文档停留在 MVP 时代（2015–2017）**，
里面大量"未来会有 :unicorn:"的表述指的是**现在早就有了**的特性：

| 文档里写的 | 今天的现实 |
| --- | --- |
| "MVP 不支持多返回值，但很快会加" | Wasm **2.0 已有** |
| "未来会加 SIMD" | Wasm **2.0 已有**（定宽），3.0 有 Relaxed SIMD |
| "未来会加线程和共享内存" | 浏览器早已实现，proposal 在 **Phase 4** |
| "未来会加零成本 C++ 异常处理" | Wasm **3.0 已有** |
| "MVP 只支持 wasm32，未来加 wasm64" | Wasm **3.0 的 memory64 已进 spec** |
| "未来会加 GC" | Wasm **3.0 已有** |
| "MVP 阶段还没有稳定 ABI" | 仍然基本成立（工具链约定为主） |

**只有 `proposals-finished.md` 是最新的**（记录到 2025-07-23 的 WG 会议）。

> 使用规则：**读 design 仓库是为了理解"为什么"，不是为了了解"现在有什么"。**
> 现状一律以 `proposals-finished.md` + `webassembly.org/features` 为准。

本库在引用这些文档时已经全部做了现状修正。

## 覆盖的考点

- 三档深度题的绝大部分：栈机选型、CFI、不确定性、页大小、类型系统裁剪理由
- ❌ 不覆盖：具体 API 用法、工具链、工程实践

## 相关页面

- [[sources/mdn-webassembly-docs]]
- [[wasm/proposals-and-versions]]
- [[wasm/security-sandbox]]
