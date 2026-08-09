---
title: "垃圾回收：引用计数 + 分代标记清除"
date: 2026-08-07
tags: [GC, 垃圾回收, 引用计数, 循环引用, 分代, weakref, 内存泄漏]
sources: ["interview-python-cn.md"]
---

# 垃圾回收：引用计数 + 分代标记清除

「Python 的垃圾回收机制」是中文面试题库的**第 24 题**，几乎必考。完整答案分三层。

## 一句话总纲

> **CPython 以引用计数为主、分代标记-清除为辅**：引用计数负责绝大多数对象的即时回收，
> 循环 GC 专门解决引用计数无法处理的**循环引用**，并用**分代假设**降低扫描成本。

## 1. 第一层：引用计数（主力）

见 [[internals/cpython-object-model]]。计数归零立刻 `tp_dealloc`。

**致命缺陷——循环引用**：

```python
import gc, sys

class Node:
    def __init__(self): self.ref = None
    def __del__(self): print("collected")

a, b = Node(), Node()
a.ref = b
b.ref = a          # 互相引用
del a, b           # 外部引用没了，但 a.refcnt=1（被 b 引用），b.refcnt=1
                   # → 什么都不打印，内存泄漏！

gc.collect()       #=> collected / collected   循环 GC 出手才回收
```

自引用同样如此：

```python
lst = []
lst.append(lst)    # lst.refcnt = 2
del lst            # refcnt = 1，永远不归零
```

## 2. 第二层：标记-清除（mark & sweep）

循环 GC 只跟踪**容器类对象**（list/dict/set/tuple/实例/类等可能包含引用的对象）。
`int`/`str`/`float` 这类**原子对象不被 GC 跟踪**（它们不可能构成循环）。

```python
import gc
gc.is_tracked([])        #=> True
gc.is_tracked(1)         #=> False
gc.is_tracked("x")       #=> False

t = (1, 2)
gc.is_tracked(t)         #=> True    ❗ 刚创建时【仍被跟踪】
gc.collect()
gc.is_tracked(t)         #=> False   ← 一次 GC 之后才被"去跟踪"
gc.is_tracked((1, [2]))  #=> True    含可变元素的 tuple 永远被跟踪
```

⚠️ **"只含不可变原子的 tuple 不被跟踪"这个优化是惰性的**——
GC 在扫描时才判断并解除跟踪，不是创建时就判断。这个细节容易记错。

算法（CPython 用的是**引用计数差值法**，不是从根集遍历）：

```text
1. 对当前代的每个容器对象，拷贝一份 gc_refs = ob_refcnt
2. 遍历每个容器内部的引用，把被指向对象的 gc_refs 减 1
   → 减完后 gc_refs > 0 的对象，说明存在【来自该代之外】的引用 = 存活根
3. 从这些存活根出发做可达性标记，标记到的全部存活
4. 剩下 gc_refs == 0 且未被标记的 = 只被彼此引用的垃圾环 → 回收
```

这个设计的巧妙之处：**不需要知道"根集"是什么**，靠计数差就能区分内外引用。

## 3. 第三层：分代（generational）

**分代假设**：绝大多数对象"朝生夕死"，活得越久的越可能继续活着。

CPython 分 **3 代（gen0/gen1/gen2）**：

- 新创建的容器对象进 **gen0**。
- 某代 GC 后**幸存**的对象晋升到下一代。
- **gen0 扫描频繁、gen2 扫描罕见**——用扫描频率换性能。

```python
import gc
gc.get_threshold()    #=> (700, 10, 10)   ← Python ≤ 3.12
                      #=> (2000, 10, 10)  ← Python 3.13 ★ 已改
                      #=> (2000, 10, 0)   ← Python 3.14（增量式 GC，第三个阈值不再使用）
```

含义：

- **第一个数**：`分配的容器对象数 - 释放数` 超过它时触发 gen0 回收。
  **3.13 起从 700 提高到 2000**（减少 GC 频率，因为小对象分配变快了）。
- 第一个 **10**：每 10 次 gen0 回收触发一次 gen1 回收。
- 第二个 **10**：每 10 次 gen1 回收触发一次 gen2 **全量**回收（最贵）。
  **Python 3.14 改用增量式 GC**，不再有"停顿很久的全量回收"，该阈值退化为 0（未使用）。

> ⚠️ **实测于各版本**：3.8 / 3.12 → `(700, 10, 10)`；3.13 → `(2000, 10, 10)`；
> 3.14 → `(2000, 10, 0)`。**面试时说 `(700, 10, 10)` 仍然是绝大多数面试官期待的答案**
> （题库和博客都是这个数），但**主动补一句"3.13 起改成了 2000、3.14 换成了增量式 GC"
> 是很强的加分信号**——它证明你在跟进而不是背了十年前的文章。

```python
gc.get_count()        #=> (593, 3, 1)     各代当前计数
gc.get_stats()        # 每代的 collections/collected/uncollectable
gc.collect()          # 手动全量回收，返回回收的对象数
gc.collect(0)         # 只回收 gen0
gc.set_threshold(2000, 20, 20)    # 调大 → GC 更少但每次更久
gc.disable() / gc.enable() / gc.isenabled()
gc.freeze()           # 3.7+：把当前所有对象移出 GC 跟踪（fork 前调用，避免 CoW 内存爆炸）
```

> **面试落点**：这个默认阈值和它的三层含义，是"背过没背过"的分水岭。
> 顺带说清「第一个数是**净增的容器对象数**，不是内存大小、不是所有对象数」更好；
> 再补一句版本变化（≤3.12 是 700，3.13 起是 2000）就更稳。

## 4. `__del__` 与不可回收对象

Python 3.4（PEP 442）之前，**带 `__del__` 的对象若在循环里，GC 不敢回收**（不知道调用顺序），
会被丢进 `gc.garbage` 永久泄漏。

**3.4 起已修复**：循环里的 `__del__` 会被调用（顺序不保证），对象能被回收。

```python
gc.garbage      #=> []   现代 Python 里通常永远是空的
```

但 `__del__` 依然**不该用来做资源清理**：

```python
class Bad:
    def __del__(self):
        self.conn.close()    # ❌ 调用时机不确定；解释器退出时可能全局已被清空；
                             #    异常会被忽略；PyPy 上可能永远不调用
```

正确做法：**上下文管理器**（[[language/context-managers]]）或 `weakref.finalize`。

```python
import weakref
class Good:
    def __init__(self):
        self.conn = connect()
        self._fin = weakref.finalize(self, self.conn.close)   # 更可控的终结器
```

## 5. 弱引用（weakref）—— 打破循环的正规武器

```python
import weakref

class Parent:
    def __init__(self):
        self.children = []

class Child:
    def __init__(self, parent):
        self._parent = weakref.ref(parent)      # 弱引用，不增加引用计数
    @property
    def parent(self):
        return self._parent()                    # 调用它取回对象，已回收则返回 None

p = Parent()
c = Child(p)
p.children.append(c)
del p            # ✅ 立刻回收，无需等 GC（引用计数就够了）
c.parent         #=> None
```

弱引用容器：

```python
weakref.WeakValueDictionary()    # 值被回收时自动删键 —— 缓存的标准做法
weakref.WeakKeyDictionary()      # 给对象挂元数据而不阻止其回收
weakref.WeakSet()
weakref.proxy(obj)               # 像普通引用一样用，但对象没了就抛 ReferenceError
```

> ⚠️ 不是所有对象都能被弱引用：`int`/`str`/`tuple`/`list`/`dict` 不行，
> 自定义类默认可以（用了 `__slots__` 的要加 `"__weakref__"`）。

## 6. 真实世界的内存泄漏来源

引用计数 + GC 之后，Python **仍然会泄漏**，原因几乎都是"你还持有引用"：

| 泄漏源 | 说明 | 解法 |
|---|---|---|
| **全局缓存/字典无界增长** | 最常见 | `functools.lru_cache(maxsize=N)`、`WeakValueDictionary`、TTL 缓存 |
| **`lru_cache` 装饰实例方法** | 缓存 key 含 `self`，实例永不释放 | 用 `cached_property` 或每实例缓存 |
| **未取消的 asyncio Task** | Task 持有协程帧和闭包 | 保存引用并在关闭时 cancel |
| **闭包捕获大对象** | 回调持有整个上下文 | 只捕获需要的字段 |
| **模块级列表 append** | 日志缓冲、事件收集器 | 有界队列 `collections.deque(maxlen=N)` |
| **异常 traceback 持有帧** | `except X as e:` 后 e 在块外自动删除，但存到别处会拖住整个栈帧 | 只存 `str(e)` |
| **C 扩展的引用泄漏** | numpy/pandas/自写扩展 | `tracemalloc` + 原生工具 |

## 7. 排查工具

```python
# ① tracemalloc —— 标准库，定位分配来源
import tracemalloc
tracemalloc.start()
snap1 = tracemalloc.take_snapshot()
run_workload()
snap2 = tracemalloc.take_snapshot()
for stat in snap2.compare_to(snap1, "lineno")[:10]:
    print(stat)      #=> file.py:42: size=12.3 MiB (+12.3 MiB), count=100000

# ② gc 模块直接查
gc.set_debug(gc.DEBUG_LEAK)
len(gc.get_objects())            # 当前被跟踪的对象数
gc.get_referrers(obj)            # 谁引用了它（找泄漏根源）
gc.get_referents(obj)            # 它引用了谁

# ③ objgraph（第三方）—— 画引用图，最直观
import objgraph
objgraph.show_growth()           # 两次调用之间哪类对象增长最多
objgraph.show_backrefs([obj], max_depth=5, filename="refs.png")

# ④ memray（Bloomberg 出品，现代首选）/ py-spy（无侵入采样）
```

## 8. 生产调优实践

```python
# ① 长期运行的服务：适度调大阈值，减少 GC 频率
gc.set_threshold(50000, 50, 50)

# ② 请求处理型服务（如 uWSGI/gunicorn worker）：
#    有些团队 gc.freeze() + 关闭自动 GC，改为每 N 个请求手动 collect
gc.disable()
# ...每处理 1000 个请求
gc.collect()

# ③ fork 前 freeze，避免 CoW 页被 GC 的标记位写脏（gunicorn preload 场景）
gc.freeze()

# ④ 数据处理脚本：主动 del 大对象 + gc.collect()
del huge_df
gc.collect()
```

> ⚠️ 关 GC 是有风险的高级操作——只有在**确认没有循环引用**或**能定期手动回收**时才做。
> 面试里能说出"Instagram 曾通过 `gc.freeze()` + 禁用 GC 把内存降了 x%"这类案例会很出彩。

## 9. 标准答题模板

> **问：说说 Python 的垃圾回收机制。**
>
> 分三部分答：
>
> 1. **引用计数是主力**。每个对象有 `ob_refcnt`，归零立即释放。优点是即时、无停顿；
>    缺点是有计数开销、且**无法处理循环引用**。
> 2. **循环 GC 是补充**。用标记-清除处理容器对象之间的引用环。CPython 的实现用"引用计数差值法"
>    找出只被内部引用的对象组。原子对象（int/str）不被跟踪。
> 3. **分代是优化**。三代，基于"新对象更容易死"的分代假设，让 gen0 扫得勤、gen2 扫得少。
>    阈值默认 `(700, 10, 10)`，**3.13 起改为 `(2000, 10, 10)`，3.14 换成增量式 GC**。
>
> 补一句实践：**Python 仍会因"持有引用"而泄漏**，常见于全局缓存、`lru_cache` 装饰实例方法、
> 未取消的 Task；排查用 `tracemalloc` / `objgraph` / `memray`；打破循环用 `weakref`。

## 相关

- [[internals/cpython-object-model]] —— 引用计数的实现
- [[internals/gil]] —— 引用计数与 GIL 的因果
- [[internals/memory-model]] —— 回收后内存是否还给操作系统
- [[language/context-managers]] —— 为什么不用 `__del__` 做清理
- [[language/decorators]] —— `lru_cache` 的泄漏陷阱
- [[interview/question-bank-internals-concurrency]] —— GC 面试题
- [[sources/interview-python-cn]] —— 来源：中文面试题库第 24 题
