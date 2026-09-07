---
title: "弱引用（weakref）的实现原理"
date: 2026-08-27
tags: [weakref, 弱引用, GC, 循环引用, tp_weaklistoffset, finalize, 缓存]
sources: ["cpython-doc/internaldocs-gc-3.14.6.md", "interview-python-cn.md"]
---

# 弱引用（weakref）的实现原理

弱引用是**打破循环引用的正规武器**。它的价值不是"让 GC 更快回收"，而是
**让对象根本不需要 GC 介入** —— 环被打断后引用计数就能归零，当场析构。

> 用法层面见 [[internals/garbage-collection]] §6；本页讲实现、边界和坑。

## 1. 前提：类型必须预留一个槽位

弱引用不是魔法。被引用对象的**类型**里必须有 `tp_weaklistoffset`，
表示实例内存中哪个偏移量存放"弱引用链表头"。Python 层能直接看到：

```python
class C: pass

C.__weakrefoffset__       #=> 16    普通类：实例第 16 字节是弱引用链表头
list.__weakrefoffset__    #=> 0     ← 没有槽位
object.__weakrefoffset__  #=> 0
```

**offset == 0 就不能被弱引用**。实测（3.11.9）：

| 类型 | 能否弱引用 |
|---|---|
| `list` / `dict` / `tuple` | ❌ `cannot create weak reference to 'list' object` |
| `int` / `str` / `object` | ❌ |
| **`set`** | ✅ |
| 普通自定义类 | ✅ |

> ❗ **`list`/`dict` 不能被弱引用**这条最常被记错（很多人以为只有 int/str 不行）。
> 原因是省内存：给每个 list 多加 8 字节的链表头不值得。`set` 反而有。

`__slots__` 同理——加了 `__slots__` 就不再自动获得 `__weakref__` 槽：

```python
class S:  __slots__ = ()                    # weakref.ref(S()) → TypeError
class S2: __slots__ = ('__weakref__',)      # ✅ 显式加回来
```

参见 [[language/descriptors-properties]] 里 `__slots__` 那一节。

## 2. 生命周期四步

```text
① weakref.ref(o) 创建
   弱引用对象内部存一个裸指针 wr_object → o，【不做 Py_INCREF】
   并把自己挂进 o 的弱引用链表

② o 的引用计数归零 → tp_dealloc 被调用

③ tp_dealloc 第一件事：PyObject_ClearWeakRefs(o)
   遍历 o 的弱引用链表，把每个 wr_object 改写成 Py_None，摘链
   → 此后 r() 返回 None（这就是"弱引用死亡"的实现）

④ 逐个调用 callback，参数是【弱引用对象自己】，不是被引用对象
```

第 ① 步实测——引用计数确实没动：

```python
c = C()
sys.getrefcount(c)        #=> 2
r = weakref.ref(c)
sys.getrefcount(c)        #=> 2    ← 没变
```

第 ④ 步实测——**callback 拿不到原对象**，它已经在析构中了：

```text
callback! w = <weakref at ...; dead>   w() = None
```

所以 `lambda w: cleanup(w())` 这种写法必然拿到 `None`。**需要用到对象内容的清理逻辑，
必须在创建弱引用时就把值捕获出来**（或者直接用 `weakref.finalize`）。

## 3. 三个必踩的坑

### 3.1 无 callback 的弱引用会被缓存复用

```python
weakref.ref(c) is weakref.ref(c)               #=> True    ← 同一个对象！
weakref.ref(c) is weakref.ref(c, lambda w: 0)  #=> False   ← 带 callback 的不复用
c.__weakref__ is weakref.ref(c)                #=> True
```

所以别拿 `is` 判断"是不是我创建的那个弱引用"，也别给弱引用对象挂属性。

### 3.2 弱引用自己没被持有 → callback 永不触发

```python
weakref.ref(e, lambda w: print("never"))   # ❌ 返回值扔掉了
del e                                       # → 什么都不打印
```

弱引用对象**自己先被回收了**（它也是普通对象，也要靠引用计数活着），链已经摘掉。
正解是用 `finalize`，它内部自持引用：

```python
weakref.finalize(f, lambda: print("ran"))  # ✅ 实测会执行
del f
```

### 3.3 循环 GC 里的 callback 不保证执行

`raw/cpython-doc/internaldocs-gc-3.14.6.md`「Destroying unreachable objects」第 1 步原文：

> Weak references to unreachable objects are set to `None`. If the weak reference has an
> associated callback, the callback is enqueued... **We only invoke callbacks for weak
> references that are themselves reachable.** If both the weak reference and the pointed-to
> object are unreachable **we do not execute the callback**. This is partly for historical
> reasons: the callback could resurrect an unreachable object...

翻译成结论：

- 弱引用被清空（`r()` 返回 `None`）是**一定发生**的；
- **callback 只在"弱引用自身可达"时才调用**。如果弱引用和被引用对象一起落进垃圾环，
  callback 被直接丢弃 —— 拒绝在半死状态下执行用户代码，也避免 callback 复活对象。

> **所以别把关键逻辑放进弱引用 callback**。这条和 [[internals/garbage-collection]] §5
> 的"`__del__` 不能用来做资源清理"是同一类结论。

## 4. 为什么它能治 GC 的病

```text
强引用：  parent(gen2) ⇄ child(gen0)
          环成立 → 要等扫到 gen2 的全量回收，且有 25% 门槛 → 等待时间没有上界

弱引用：  parent(gen2) → child(gen0) → weakref(parent)
          环被打断 → child 的 refcnt 能归零 → 【当场析构，循环 GC 完全不参与】
```

典型改法（父子双向引用是最常见的跨代环来源）：

```python
# ❌ 跨代环
class Node:
    def __init__(self, parent=None):
        self.parent = parent
        if parent: parent.children.append(self)

# ✅ 打断
import weakref
class Node:
    def __init__(self, parent=None):
        self._parent = weakref.ref(parent) if parent else None
        if parent: parent.children.append(self)
    @property
    def parent(self):
        return self._parent() if self._parent else None
```

## 5. 派生工具选型

| 工具 | 场景 | 注意 |
|---|---|---|
| `WeakValueDictionary` | 缓存：value 没别人用了自动出表 | value 必须可弱引用（不能是 list/dict！） |
| `WeakKeyDictionary` | 给对象挂外部元数据而不阻止回收 | key 必须可弱引用且可哈希 |
| `WeakSet` | observer / 监听器列表 | 最典型的泄漏源，用它 |
| `weakref.proxy` | 透明代理，用法像普通引用 | 对象死后访问抛 `ReferenceError`；不可哈希 |
| `weakref.finalize` | "对象死了跑清理"的**唯一正确姿势** | 自持引用，不需要你保存返回值 |

```python
_CACHE = {}                              # ❌ 长跑进程里就是永久内存
@lru_cache(maxsize=None)                 # ❌ 同上
_CACHE = weakref.WeakValueDictionary()   # ✅
@lru_cache(maxsize=1024)                 # ✅ 有界
```

> ⚠️ `WeakValueDictionary` 不是万能缓存：value 一旦没有别的强引用就**立刻**出表，
> 想要"至少留住 N 个"得配合 LRU 之类的强引用层。

## 相关

- ★ [[internals/garbage-collection]] —— 循环 GC、跨代环、浮动垃圾（本页的上文）
- [[internals/cpython-object-model]] —— 引用计数与 `tp_*` 槽位
- [[language/descriptors-properties]] —— `__slots__` 与 `__weakref__`
- [[language/context-managers]] —— 资源清理的首选方案
- [[language/decorators]] —— `lru_cache` 的泄漏陷阱
- [[sources/cpython-docs]] —— 来源：CPython `InternalDocs/garbage_collector.md`
