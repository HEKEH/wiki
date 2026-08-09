---
title: "源：satwikkansal/wtfpython"
date: 2026-08-07
tags: [源导读, 陷阱, 边界情况]
sources: ["wtfpython.md"]
---

# 源：satwikkansal/wtfpython

- **原始路径**：`raw/wtfpython.md`（约 125 KB）
- **来源**：<https://github.com/satwikkansal/wtfpython>（有中文版 wtfpython-cn）
- **抓取日期**：2026-08-07

## 这份材料是什么

**一本"Python 惊奇代码片段集"**。每一节都是：

```text
① 给出一段看起来正常的代码
② 展示令人意外的输出
③ 用 "💡 Explanation" 解释底层机制
```

约 60+ 个片段，分成 "Strain your brain"、"Slippery Slopes"、"The Hidden treasures"、
"Miscellaneous" 等章节。

## 为什么它对面试特别有价值

**面试官非常喜欢从这里出题**——因为每道题都精确对应一个底层机制：

| 陷阱 | 实际考的是 |
|---|---|
| 可变默认参数 | 函数对象的 `__defaults__` 与求值时机 |
| `is` 与小整数 | CPython 的对象缓存与驻留 |
| 循环变量延迟绑定 | 闭包的 cell 机制、无块级作用域 |
| `tuple` 里的 list `+=` | `__iadd__` 与 `STORE_SUBSCR` 的两步语义 |
| dict 里 `5` 与 `5.0` | `__hash__`/`__eq__` 契约 |
| 类作用域不参与闭包 | LEGB 中 E 只含函数 |
| `finally` 里 return | 异常传播与控制流 |
| 遍历时修改容器 | 迭代器的内部指针 |
| `round(0.5) == 0` | banker's rounding |
| NaN 的自反性 | IEEE 754 与 set 的 `is` 短路 |

**答题时不能只说结果，要说清机制**——这正是本库 [[interview/traps]] 页的组织方式。

## 在本库里的用途

| 本库页面 | 引用内容 |
|---|---|
| [[interview/traps]] | **主要消化页**，精选 22 个高频陷阱并逐一给出机制解释与解法 |
| [[language/objects-mutability]] | `is` / 驻留 / 拷贝相关片段 |
| [[language/scope-closure]] | 延迟绑定、类作用域、变量泄漏 |
| [[language/functions-arguments]] | 可变默认参数 |
| [[stdlib/builtin-data-structures]] | hash/eq 契约、字典键合并 |

**本库对源材料的处理**：所有引用的片段都在 **CPython 3.13 上重新实测**过，
输出以实测为准（原文部分示例基于较老版本）。

## 局限

- **只讲"意外"，不成体系**——不能当教材，只能当"体检"。
- 部分片段涉及非常边缘的实现细节（如 `dict` 的迭代顺序内部行为），
  面试价值低，实际编码中永远不会遇到。
- 有些示例的输出**依赖 Python 版本和运行方式**（REPL vs 脚本），
  阅读时要注意上下文——这也正是"实现细节 vs 语言规范"的绝佳教学素材。

## 推荐用法

**面试前把 [[interview/traps]] 通读两遍**，对每道题练习"结果 → 机制 → 解法 → 加分句"的四段式回答。
原始材料可以在准备充分后通读一遍，作为查漏补缺。

## 相关

- [[interview/traps]] —— 本库消化后的精选陷阱题
- [[sources/python-cheatsheet]] —— 互补：那份讲正常用法
- [[interview/question-bank-language]] —— 系统的语言题库
