---
title: "源：CPython 官方文档与 PEP"
date: 2026-08-07
tags: [源导读, 官方文档, PEP, 权威]
sources: ["cpython-doc/datamodel.rst", "cpython-doc/descriptor-howto.rst", "cpython-doc/functional-howto.rst", "cpython-doc/faq-programming.rst", "pep-0008-style-guide.rst", "pep-0492-async-await.rst", "pep-0703-free-threading.rst"]
---

# 源：CPython 官方文档与 PEP

从 `python/cpython` 与 `python/peps` 仓库直接抓取的**一手权威材料**。
**抓取日期**：2026-08-07。

## 抓取清单

| 文件 | 原路径 | 大小 | 内容 |
|---|---|---|---|
| `raw/cpython-doc/datamodel.rst` | `Doc/reference/datamodel.rst` | 157 KB | **数据模型语言参考**：对象/类型/值、所有 dunder 方法的精确语义 |
| `raw/cpython-doc/descriptor-howto.rst` | `Doc/howto/descriptor.rst` | 55 KB | **描述符 HOWTO**：从零推导 property/classmethod/slots 的实现 |
| `raw/cpython-doc/functional-howto.rst` | `Doc/howto/functional.rst` | 50 KB | **函数式编程 HOWTO**：迭代器、生成器、itertools、functools |
| `raw/cpython-doc/faq-programming.rst` | `Doc/faq/programming.rst` | 79 KB | **编程 FAQ**：大量"为什么 Python 这样设计"的官方回答 |
| `raw/pep-0008-style-guide.rst` | PEP 8 | 51 KB | 代码风格指南 |
| `raw/pep-0492-async-await.rst` | PEP 492 | 49 KB | **async/await 语法的设计文档** |
| `raw/pep-0703-free-threading.rst` | PEP 703 | 86 KB | **移除 GIL 的完整方案** |

## 各文档的价值

### `datamodel.rst` —— 语言的"宪法"

**如果只读一份官方文档，就读这份**。它精确定义了：

- 对象的 identity / type / value 三要素
- 标准类型层级
- **所有特殊方法的精确调用语义**（包括本库反复强调的
  "隐式调用的 dunder 在**类型**上查找，不在实例上查找"）
- 属性访问的完整流程、描述符的优先级
- `__slots__`、`__init_subclass__`、`__set_name__`、元类的类创建流程
- 协程与异步迭代协议

消化进：[[language/data-model]]、[[language/classes-mro]]、[[language/metaclasses]]、
[[language/context-managers]]。

### `descriptor-howto.rst` —— 最好的进阶教材之一

Raymond Hettinger 写的，**用纯 Python 重新实现 `property`/`classmethod`/`staticmethod`/
`__slots__`**，读完就彻底理解了"方法为什么能自动绑定 self"。
面试里能讲清描述符 = 直接进阶。

消化进：[[language/descriptors-properties]]。

### `functional-howto.rst`

迭代器协议、生成器、生成器表达式、`itertools`、`functools` 的系统讲解，
比 API 参考多了"什么时候用"。

消化进：[[language/iterators-generators]]、[[stdlib/collections-itertools]]、
[[language/comprehensions-functional]]。

### `faq-programming.rst` —— 被低估的宝藏

**官方对"为什么"的直接回答**，很多正是面试题：

- 为什么可变默认参数会共享？
- 为什么 lambda 捕获的是变量不是值？怎么修？
- 为什么方法要显式写 self？
- 怎么在函数间共享全局变量？
- 为什么 `a += b` 对 list 和 tuple 行为不同？
- 怎么复制对象？浅拷贝深拷贝
- 为什么 `id()` 会被复用？

消化进：[[language/objects-mutability]]、[[language/scope-closure]]、
[[language/functions-arguments]]。

### PEP 492（async/await）

**理解 asyncio 的设计意图**：为什么要引入新语法而不继续用 `yield from`、
`__await__` 协议、原生协程与生成器协程的区别、`async for` / `async with` 的语义。

消化进：[[concurrency/asyncio-fundamentals]]。

### PEP 703（free-threading）★

**2026 年最值得读的 PEP**。它不只讲"怎么移除 GIL"，还完整论述了
**GIL 为什么存在**——这正是面试标准答案的一手来源：

- 引用计数的线程安全问题与三种解法的权衡
- **偏向引用计数（biased reference counting）**
- **不朽对象（immortal objects，PEP 683）**
- 每对象锁、线程安全的内存分配器
- 对 C 扩展生态的影响与迁移路径
- 性能回退的量化分析

消化进：[[internals/gil]]。

### PEP 8

代码风格。**但实践中不靠人记，靠 `ruff format` + `ruff check` 强制**——
见 [[engineering/tooling-quality]]。

## 官方文档的正确用法

```text
docs.python.org 的四个部分，用途完全不同：
├── Tutorial        —— 入门，转语言者可以快速扫一遍
├── Library Reference —— 查 API（最常用）
├── Language Reference —— 查语义（datamodel.rst 在这里）★ 最深
└── HOWTOs          —— 主题教程（descriptor / functional / logging / sorting 都很好）
```

**面试准备的建议顺序**：先读本库的原理页建立框架 →
遇到不确定的细节回来查 `datamodel.rst` →
对 GIL/GC 这类必考题去读对应 PEP 拿到一手论述。

## 未抓取但值得读的

```text
Doc/howto/logging.rst          日志的正确用法
Doc/howto/sorting.rst          排序指南（Timsort、key、稳定性）
Doc/library/asyncio-*.rst      asyncio 全套 API
PEP 20   Zen of Python（很短，值得背）
PEP 484  类型注解
PEP 572  海象运算符（含设计争议，很有意思）
PEP 634  结构化模式匹配
PEP 684  每解释器 GIL
PEP 709  推导式内联
devguide.python.org            CPython 开发者指南（想读源码从这里开始）
```

## 相关

- [[language/data-model]] / [[language/descriptors-properties]] —— 主要消化页
- [[internals/gil]] —— PEP 703 的消化
- [[concurrency/asyncio-fundamentals]] —— PEP 492 的消化
- [[engineering/tooling-quality]] —— PEP 8 的工程化落地
- [[sources/python-cheatsheet]] —— 速查用的二手材料
