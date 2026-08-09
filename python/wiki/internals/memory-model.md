---
title: "内存模型与分配器"
date: 2026-08-07
tags: [内存, pymalloc, arena, 内存池, 内存分析, RSS]
sources: ["interview-python-cn.md"]
---

# 内存模型与分配器

「为什么我 `del` 了对象，进程内存（RSS）却没降？」——这是生产环境高频问题，
也是能拉开差距的面试题。答案在 CPython 的**三层内存分配器**。

## 1. 三层结构

```text
┌─────────────────────────────────────────────┐
│ Layer 3: 对象特化分配器                       │  int/list/dict/tuple 的 freelist
├─────────────────────────────────────────────┤
│ Layer 2: pymalloc（对象分配器）                │  ≤ 512 字节的小对象
│   arena(1MB) → pool(4KB/16KB) → block(8~512B) │
├─────────────────────────────────────────────┤
│ Layer 1: 通用分配器                           │  > 512 字节 → 直接 malloc
├─────────────────────────────────────────────┤
│ Layer 0: 操作系统 malloc / mmap / brk         │
└─────────────────────────────────────────────┘
```

### pymalloc 的三级结构

| 单位 | 大小 | 说明 |
|---|---|---|
| **arena** | **1 MB**（3.12 起；早期 256 KB） | 向 OS 申请的最大单位，用 mmap |
| **pool** | 一个系统内存页（常见 4 KB / 16 KB） | 一个 pool 只服务**一种大小类**的 block |
| **block** | 16, 32, 48, …, **512 字节**上限，**32 个大小类** | 实际分给对象的单元 |

```console
$ PYTHONMALLOCSTATS=1 python3.13 -c "pass"
Small block threshold = 512, in 32 size classes.
1 arenas * 1048576 bytes/arena     =            1,048,576
```

> ⚠️ 常见的错误说法是"8 字节对齐、64 个大小类"——那是 **32 位**时代的数字。
> **64 位构建上是 16 字节对齐、32 个大小类**（512 ÷ 16 = 32），可用上面的命令直接验证。

分配 `x = SomeSmallObject()` 时：找到对应大小类的 pool → 从 free list 取一个 block。
**这是 O(1) 的指针操作，比 `malloc` 快得多**——这是 Python 能承受"万物皆对象"的关键。

## 2. 为什么内存不还给操作系统

**关键规则：只有一个 arena 内的所有 pool 全部空闲，这个 arena 才会被释放回 OS。**

```python
import os, psutil
p = psutil.Process()
rss = lambda: p.memory_info().rss // 1024**2

rss()                    #=> 20 MB
big = [object() for _ in range(10**7)]
rss()                    #=> 700 MB
del big
import gc; gc.collect()
rss()                    #=> 仍然 300+ MB   ← 内存"没还回去"
```

三个原因：

1. **内存碎片**：只要 arena 里还有一个存活对象，整个 1MB arena 就不能释放。
2. **大对象走 malloc**：glibc 的 malloc 对小块内存也会缓存不还给内核（`M_TRIM_THRESHOLD`）。
3. **freelist 缓存**：float/list/tuple/frame 等类型维护自己的空闲对象链表，主动保留。

> **面试落点**：「Python 释放的内存去哪了？」标准答案：
> **对象被回收后内存回到 pymalloc 的池里，供后续 Python 对象复用，但通常不归还给操作系统**；
> 只有整个 arena 完全空闲时才会释放。因此 RSS 是"历史峰值"的近似，不是当前使用量。

**工程对策**：

```python
# ① 峰值内存高的任务放到子进程 —— 进程退出，内存必然归还
from concurrent.futures import ProcessPoolExecutor
with ProcessPoolExecutor(max_workers=1) as ex:
    result = ex.submit(memory_heavy_task, data).result()

# ② 流式处理，避免一次性物化
for chunk in pd.read_csv(f, chunksize=100_000):    # 而不是一次读完
    process(chunk)

# ③ 用生成器代替列表
sum(x for x in huge)      # 而不是 sum([x for x in huge])

# ④ gunicorn 的 --max-requests 让 worker 定期重启（业界标准做法）
gunicorn app:app --max-requests 1000 --max-requests-jitter 100
```

## 3. 对象大小与内存效率

```python
import sys
sys.getsizeof(1)                #=> 28    小整数
sys.getsizeof([])               #=> 56
sys.getsizeof({})               #=> 64
sys.getsizeof(set())            #=> 216   set 起步就大
sys.getsizeof(())               #=> 40
```

**容器只存指针**：`sys.getsizeof` 不递归。真实占用要自己算或用工具：

```python
from pympler import asizeof
asizeof.asizeof(obj)            # 递归计算

# 或用标准库
import tracemalloc
tracemalloc.start()
# ...
current, peak = tracemalloc.get_traced_memory()
```

### 省内存的具体手段（面试实战题）

```python
# ① __slots__ —— 百万级小对象场景，能省 40~60%
class P:
    __slots__ = ("x", "y")

# ② array / numpy —— 同质数值数据
import array
array.array("i", range(10**6))       # 4 MB   vs  list 约 40 MB（8B 指针 + 28B int 对象）
import numpy as np
np.arange(10**6, dtype=np.int32)     # 4 MB，且支持向量化运算

# ③ 生成器代替列表
# ④ sys.intern 大量重复字符串
# ⑤ tuple / frozenset 代替 list / set（更紧凑）
# ⑥ dataclass(slots=True)
# ⑦ 大数据用 pyarrow / polars（列式，零拷贝）
```

对比实测（100 万个二维点，CPython 3.13 / 64 位，`getsizeof` 递归计入实例的 `__dict__`，
不含承载它们的 list 本身的 8 MB 指针数组）：

```text
list of dict            ~ 175 MB
list of 普通类实例        ~ 130 MB   （3.11+ 共享键字典已优化过一轮）
list of NamedTuple       ~  53 MB
list of __slots__ 类     ~  46 MB   ← 最省
numpy 结构化数组          ~   8 MB   （1M × 2 × int32，连续内存无对象头）
```

> ⚠️ 注意 **`__slots__` 类比 NamedTuple 还略省**（两个槽位 vs tuple 的头部 + 长度字段）。
> 直觉上容易以为 tuple 最紧凑，实测不是。选 NamedTuple 的理由应该是**不可变和可解包**，
> 而不是省内存。

## 4. list / dict 的扩容策略（常考）

**list 的过度分配（over-allocation）**：

```python
import sys
lst = []
prev = 0
for i in range(20):
    lst.append(i)
    cur = sys.getsizeof(lst)
    if cur != prev:
        print(i + 1, cur)
        prev = cur
#=> 1 88 / 5 120 / 9 184 / 17 248 / ...   （CPython 3.13 实测）
```

增长公式（CPython `list_resize`）：`new = n + (n >> 3) + 6` 左右，
即**约 1.125 倍 + 常数**（不是 2 倍！）。

- 好处：`append` 的**摊还复杂度 O(1)**。
- 增长因子小 → 内存更省，但重分配更频繁（Python 的取舍偏向省内存）。
- **已知大小时预分配更快**：`[None] * n` 或直接 `list(iterable)`。

**dict 的扩容**：负载因子超过 **2/3** 时扩容到 `used * 3`（3.4+ 是 `used*2` 到 `used*3` 之间，
具体版本相关），容量始终是 2 的幂。

```python
d = {}
sys.getsizeof(d)          #=> 64
for i in range(6): d[i] = i
sys.getsizeof(d)          #=> 352   扩容发生（3.13 实测）
```

> **面试落点**：「list append 是 O(1) 吗？」——**摊还 O(1)**。
> 因为过度分配，n 次 append 的总重分配代价是 O(n)。能说出增长因子约 1.125 而非 2 更好。

## 5. freelist 与对象复用

CPython 给高频创建/销毁的类型保留了空闲对象链表：

```python
# float、list、dict、tuple(按长度)、frame、method 等都有 freelist
a = 1.5
id_a = id(a)
del a
b = 2.5
id(b) == id_a        #=> 常常是 True —— 复用了同一块内存
```

这是 `id()` 会"重复"的原因，也是为什么**不能用 `id()` 做长期唯一标识**：

```python
# ❌ 危险：对象被回收后 id 会被复用
seen = {id(obj) for obj in objs}
# ✅
seen = {obj.uuid for obj in objs}
```

## 6. 排查工具速查

```python
# 标准库
import tracemalloc                    # 分配来源统计，可比较快照
tracemalloc.start(25)                 # 保留 25 层调用栈
sys.getsizeof(obj)
gc.get_objects()                      # 所有被 GC 跟踪的对象

# 第三方
memray run app.py                     # Bloomberg，最强的内存剖析器（火焰图、实时模式）
memray flamegraph output.bin
py-spy dump --pid 1234                # 无侵入看栈
objgraph.show_growth()                # 哪类对象在涨
pympler.asizeof                       # 递归大小
psutil.Process().memory_info()        # RSS / VMS
```

**生产环境排查内存增长的标准流程**：

```text
1. psutil / 监控确认 RSS 确实在涨（区分"真泄漏"和"pymalloc 不归还"）
2. objgraph.show_growth() 找出增长最快的对象类型
3. gc.get_referrers() 或 objgraph.show_backrefs() 找持有引用的根
4. tracemalloc / memray 定位分配代码位置
5. 确认是不是全局缓存 / lru_cache / 未取消 Task（见 GC 页的泄漏清单）
```

## 7. 环境变量与调试构建

```bash
PYTHONMALLOC=malloc python app.py      # 绕过 pymalloc，用系统 malloc（配合 valgrind）
PYTHONMALLOC=debug python app.py       # 开启内存调试（越界检测、未初始化填充）
PYTHONMALLOCSTATS=1 python -c "pass"   # 退出时打印 arena/pool 统计
```

## 相关

- [[internals/cpython-object-model]] —— 对象头与 getsizeof
- [[internals/garbage-collection]] —— 回收时机与泄漏排查
- [[stdlib/builtin-data-structures]] —— list/dict 的底层布局
- [[language/descriptors-properties]] —— `__slots__` 的内存收益
- [[engineering/performance]] —— 性能与内存的权衡
- [[interview/question-bank-internals-concurrency]] —— 内存相关面试题
