---
title: "源：gto76/python-cheatsheet"
date: 2026-08-07
tags: [源导读, 速查表, 标准库]
sources: ["python-cheatsheet.md"]
---

# 源：gto76/python-cheatsheet

- **原始路径**：`raw/python-cheatsheet.md`（约 150 KB）
- **来源**：<https://github.com/gto76/python-cheatsheet>（GitHub 上 star 数最高的 Python 速查表之一）
- **抓取日期**：2026-08-07

## 这份材料是什么

一份**极度密集的单页 Python 速查表**，按主题罗列语法与标准库用法，几乎全是代码、极少散文。
覆盖范围异常广：

```text
Collections（list/dict/set/tuple/range/enumerate/iterator/generator）
Types（type/abc/str/regex/format/numbers/combinatorics/datetime）
Syntax（args/inplace_operators/decorator/class/duck_types/enum/except/exit）
System（print/input/argparse/open/paths/os_commands）
Data（json/pickle/csv/sqlite/bytes/struct/array/memoryview/deque）
Advanced（threading/operator/match/logging/introspection/coroutines）
Libraries（progress_bar/plot/table/console_app/gui/scraping/web/profile/numpy/image/audio/games/pandas/pygame）
```

## 在本库里的用途

它是**语法与标准库层面的事实核对来源**，不适合作为学习材料（没有解释"为什么"）。
本库引用了它的以下部分：

| 本库页面 | 引用内容 |
|---|---|
| [[stdlib/stdlib-essentials]] | pathlib / datetime / json / logging / argparse / subprocess 的完整 API |
| [[stdlib/collections-itertools]] | itertools 与 collections 的全量函数签名 |
| [[language/strings-encoding]] | 格式化说明符（对齐、进制、千分位、精度）的完整表 |
| [[language/functions-arguments]] | 参数与解包的各种形式 |
| [[language/data-model]] | dunder 方法清单 |
| [[interview/coding-patterns]] | 常用惯用法 |

## 优点与局限

**优点**：

- 信息密度极高，**查语法比翻官方文档快**。
- 持续维护，跟进新版本特性（match、walrus、dataclass 等）。
- 有 PDF 版可打印，适合面试前一天速览。

**局限**：

- **只讲"怎么写"，不讲"为什么"和"什么时候用"**——不能替代原理性学习。
- 缺少 CPython 内部机制（GIL/GC/内存）——这恰恰是面试重点，
  由本库的 [[internals/gil]] 等页面补足。
- 缺少 Web 框架和工程实践。
- 排版是单页巨型文档，不利于按主题精读。

## 推荐用法

**面试前一周每天扫一遍语法部分**，确保不会因为"想不起 `itertools.groupby` 怎么写"而卡壳。
理解层面的准备走本库的 `language/` 和 `internals/`。

## 相关

- [[sources/wtfpython]] —— 互补：那份讲边界情况
- [[stdlib/stdlib-essentials]] —— 本库的标准库整理
- [[interview/roadmap]] —— 何时用这份材料
