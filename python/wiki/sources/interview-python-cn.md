---
title: "源：中文 Python 面试题库（interview_python / Python面试宝典）"
date: 2026-08-07
tags: [源导读, 面试题, 中文, 过时内容]
sources: ["interview-python-cn.md", "interview-bible-cn.md"]
---

# 源：中文 Python 面试题库

两份材料：

| 文件 | 来源 | 大小 |
|---|---|---|
| `raw/interview-python-cn.md` | `taizilongxu/interview_python`（17k+ star，中文 Python 面试题的"祖师爷"） | 73 KB |
| `raw/interview-bible-cn.md` | `jackfrued/Python-Interview-Bible`（Python面试宝典-2020） | 74 KB |

**抓取日期**：2026-08-07

## 覆盖的题目

`interview_python` 的 30 道 Python 题（这就是中文面试圈的"标准题库"）：

```text
1 函数参数传递          2 元类 metaclass        3 staticmethod/classmethod
4 类变量和实例变量       5 自省                  6 字典推导式
7 单下划线和双下划线      8 字符串格式化           9 迭代器和生成器
10 *args/**kwargs      11 AOP 和装饰器         12 鸭子类型
13 重载                14 新式类和旧式类 ⚠️     15 __new__ 和 __init__
16 单例模式            17 作用域               18 GIL ★
19 协程                20 闭包                 21 lambda
22 函数式编程          23 拷贝                 24 垃圾回收 ★
25 List                26 is                   27 read/readline/readlines
28 Python2 和 3 的区别  29 super init           30 range/xrange ⚠️
```

外加操作系统（select/poll/epoll、调度、死锁、分页分段）、数据库（事务、索引、MVCC、
InnoDB vs MyISAM）、网络（三次握手、HTTP/HTTPS、Cookie/Session、CGI/WSGI、C10K）三大块。

`Python-Interview-Bible` 覆盖 Python 2/3 差异、随机数、线程池、PEP 8 等，
以及数据分析/SQL/机器学习的独立篇章。

## ⚠️ 重要：过时内容标注

这两份材料成型于 **2015~2020 年**，有相当比例的内容**已经过时**，
照搬会在面试中暴露知识陈旧：

| 原材料内容 | 2026 年的现状 |
|---|---|
| 新式类 vs 旧式类 | **Python 3 里全是新式类**，这个区分已无意义（只在维护 Py2 代码时才有）|
| `range` vs `xrange` | Python 3 只有惰性的 `range`，`xrange` 已移除 |
| `%` 和 `.format` 格式化 | 现在首选 **f-string**（3.6+），且性能最好 |
| Python 2/3 差异（大量篇幅）| Python 2 已于 **2020-01-01 EOL**，只需知道字符串模型的差异 |
| GIL 的描述 | 缺少 **PEP 703 free-threading（3.13+）** 和 3.2 的时间片切换改进 |
| 垃圾回收 | 缺少 **PEP 442（3.4 修复了带 `__del__` 的循环回收）** |
| 字典无序 | **3.7 起有序已写进语言规范** |
| `dict.has_key()` | 已移除，用 `in` |
| 单例的 4 种实现 | 现在优先模块级变量或 `functools.cache`，元类方案是过度设计 |
| 协程用 `yield from` | 3.5 起用 `async`/`await`，asyncio API 已大改 |
| 缺失的现代主题 | 类型注解、dataclass、pattern matching、asyncio 生态、pyproject/uv、Pydantic/FastAPI |

**本库的处理方式**：吸收题目框架（这些题确实还在被问），
但**答案全部按 Python 3.12+ 的现状重写**，并显式标注版本演进。

## 在本库里的用途

| 本库页面 | 对应源题目 |
|---|---|
| [[interview/question-bank-language]] | 题 1、3-13、15、17、20-23、25-27、29 |
| [[internals/gil]] | **题 18**（并补充 PEP 703） |
| [[internals/garbage-collection]] | **题 24**（并补充 PEP 442、weakref、泄漏排查） |
| [[language/metaclasses]] | 题 2、14、15、16 |
| [[language/objects-mutability]] | 题 1、23、26 |
| [[language/strings-encoding]] | 题 8、28 |
| [[language/decorators]] | 题 11 |
| [[language/iterators-generators]] | 题 9、19 |
| [[concurrency/io-multiplexing]] | **操作系统篇的 select/poll/epoll**、网络篇的 C10K |
| [[web/wsgi-asgi]] | 网络篇的 CGI/WSGI |
| [[interview/question-bank-web]] | 数据库篇（事务/索引/MVCC）与网络篇（HTTP/Cookie-Session/RESTful）的部分内容 |

> ⚠️ **本库未覆盖的源材料部分**：操作系统篇的调度算法/死锁/分页分段/虚拟内存、
> 网络篇的三次握手四次挥手/ARP/HTTPS 细节、数据库篇的 MVCC/InnoDB 内部。
> 这些属于通用计算机基础而非 Python 知识，本库只吸收了与 Python 直接相关的部分
> （epoll → asyncio、CGI/WSGI → ASGI、死锁 → threading）。
> 完整的数据库与网络基础见 [[home]] 的开放问题。

## 价值判断

**仍然值得看**：因为**中文面试圈的题目分布确实来自这里**——
GIL、垃圾回收、装饰器、深浅拷贝、闭包这几道题被问了十年，至今还在问。

**但不能只看它**：按它的答案回答只能拿到"及格分"。
拿高分要在这些题上答出**版本演进、实现细节 vs 语言规范的区分、工程权衡**——
这正是本库各原理页想补足的部分。

## 相关

- [[interview/question-bank-language]] —— 现代化重写的语言题库
- [[interview/question-bank-internals-concurrency]] —— GIL/GC 的完整答法
- [[interview/roadmap]] —— 复习优先级
- [[sources/wtfpython]] —— 补充陷阱题
