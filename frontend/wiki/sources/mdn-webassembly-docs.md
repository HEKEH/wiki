---
title: 源材料 —— MDN WebAssembly 文档
date: 2026-08-10
tags: [源材料, mdn]
sources: [mdn-wasm/index.md, mdn-wasm/guides/concepts.md, mdn-wasm/guides/loading_and_running.md, mdn-wasm/guides/using_the_javascript_api.md, mdn-wasm/guides/understanding_the_text_format.md, mdn-wasm/reference/javascript_interface.md]
---

# 源材料：MDN WebAssembly 文档

**位置**：`raw/mdn-wasm/`（22 个文件）
**来源**：`github.com/mdn/content` → `files/en-us/webassembly/`
**抓取日期**：2026-08-10

## 这份材料是什么

MDN 的 WebAssembly 章节，分 Guides（教程）和 Reference（参考）两部分。
**这是本期最主要的源材料**——它是唯一一份既讲原理又讲实操、且持续更新的官方级文档。

### Guides（`raw/mdn-wasm/guides/`）

| 文件 | 覆盖内容 | 落到哪一页 |
| --- | --- | --- |
| `concepts.md` | Wasm 是什么、设计目标、四个核心概念、四条入门路径 | [[wasm/what-is-wasm]]、[[wasm/core-model]] |
| `loading_and_running.md` | fetch + compile/instantiate 的各种组合、streaming 为什么快 | [[wasm/js-interop]] |
| `using_the_javascript_api.md` | Memory / Table / Global 的完整用法、**多重性（multiplicity）** | [[wasm/core-model]]、[[wasm/js-interop]] |
| `understanding_the_text_format.md` | **最有价值的一篇**（931 行）：S-表达式、栈机、import/export、memory、data、table、call_indirect、动态链接、bulk memory、多内存、共享内存 | [[wasm/text-format-wat]]、[[wasm/memory-model]] |
| `exported_functions.md` | 导出函数在 JS 里的真实形态（length/name/toString） | [[wasm/types-and-abi]] |
| `imported_string_constants.md` | Wasm 3.0 的字符串常量导入优化 | [[wasm/types-and-abi]] |
| `rust_to_wasm.md` | wasm-pack 完整流程 + webpack 集成 | [[wasm/toolchain-rust]] |
| `c_to_wasm.md` / `existing_c_to_wasm.md` | Emscripten 基础 + **libwebp 编码完整案例** | [[wasm/toolchain-emscripten]]、[[wasm/use-cases]] |
| `text_format_to_wasm.md` | wat2wasm 工具用法 | [[wasm/debugging]] |

### Reference（`raw/mdn-wasm/reference/`）

`javascript_interface.md` 是 JS API 的完整清单（含 `Suspending`、`promising`、
`Tag`、`Exception`、`JSTag` 这些新成员）。其余是指令分类索引：
`control_flow` / `numeric` / `memory` / `table` / `value_types` / `simd` /
`exception_handling` / `variables`。

这些索引页**只有指令名和一句话说明**，没有语义细节——
细节要看 spec（`webassembly.github.io/spec/core/`）。

## 时效性评估

✅ **很新**。这批文档已经覆盖了 Wasm 3.0 的内容：
`exnref`、`try_table`/`catch_ref`、多内存、JS String Builtins、
`WebAssembly.Suspending`/`promising`（JSPI）都有。

⚠️ 局部陈旧：

- `understanding_the_text_format.md` 里"当前最多 1 个返回值""multi-value 处于早期阶段
  （2020 年 6 月）"——**已过时**，multi-value 在 Wasm 2.0 就进 spec 了
- 同文件里"目前每个模块实例只允许一个 table"——已过时，reference-types 之后支持多 table
- `loading_and_running.md` 里的 XMLHttpRequest 章节纯属历史包袱，实践中不用
- 「Firefox 58 新增」这类版本注记是老文案残留

本库在写页面时已经**显式修正**了这些表述。

## 覆盖的考点

- 一档基础题全覆盖
- 二档的字符串传递、memory grow、table/call_indirect 全覆盖
- ❌ **不覆盖**：性能量化数据、安全模型的深层分析、
  proposal 阶段、非 Web 场景——这些要看 design 仓库和 proposals 仓库

## 局限

MDN 是**面向使用者**的文档，讲"怎么用"很好，讲"为什么这么设计"很少。
面试里的深度题（为什么选栈机、为什么 table 不能放线性内存、NaN 为什么不确定）
必须补 `raw/wasm-design/Rationale.md`——见 [[sources/wasm-design-repo]]。

## 相关页面

- [[sources/wasm-design-repo]]
- [[index]]
