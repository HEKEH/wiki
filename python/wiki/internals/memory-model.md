---
title: "内存模型与分配器"
date: 2026-08-07
tags: [内存, pymalloc, arena, 内存池, 内存分析, RSS, fork, copy-on-write]
sources: ["interview-python-cn.md"]
---

# 内存模型与分配器

「为什么我 `del` 了对象，进程内存（RSS）却没降？」——这是生产环境高频问题，
也是能拉开差距的面试题。答案在 CPython 的**三层内存分配器**。

## 1. 三层分配器

先分清本页两个容易混淆的「三」——它们是**互相垂直**的两件事：

- **纵向的三层分配器**：一次内存请求依次经过谁，是**调用链**（本节 1.1）
- **横向的 arena → pool → block**：第 2 层 pymalloc **自己内部**怎么切内存（本节 1.2）

（CPython 源码 `Objects/obmalloc.c` 的注释里其实编号到 Layer 0，
但 Layer 0 是操作系统、不属于 Python，所以习惯说「三层」。）

### 1.1 纵向：一次分配的调用链

规则只有一条：**上层要不到，才向下层要；下层不知道上层在干什么。**

```text
              需要为对象分配内存
                     │
                     ▼
  Layer 3  对象特化 freelist（float/list/dict/tuple/frame …）
           命中？ ──是──► 返回，纳秒级，完全不碰分配器
                     │否
                     ▼
              这次请求 ≤ 512 字节？
           ──否──► Layer 1  通用分配器  PyMem_RawMalloc → libc malloc
                     │是                                    │
                     ▼                                      │
  Layer 2  pymalloc（PyObject_Malloc）                       │
           从对应大小类的 pool 摘一个 block，O(1) 指针操作      │
           arena 用尽？ ──是──► 申请新 arena ────────────────┤
                     │否                                    │
                     ▼                                      ▼
                 返回 block              Layer 0  操作系统 mmap / brk
```

| 层 | 是谁 | C 接口 | 负责什么 |
|---|---|---|---|
| **Layer 3** | 对象特化分配器 | 各类型自己的代码 | 「我这类对象刚死的空壳，我自己留着复用」 |
| **Layer 2** | pymalloc | `PyObject_Malloc` | **≤ 512 字节**的小块 |
| **Layer 1** | 通用分配器 | `PyMem_RawMalloc` | 直接转发给 libc `malloc` |
| **Layer 0** | 操作系统 | `mmap` / `brk` | 真正的物理内存 |

**具体走一遍**——`x = 3.5`，一个 float 对象需要 24 字节：

1. **Layer 3**：float 有自己的 freelist，缓存着之前 `del` 掉的 float 空壳。
   命中就改个值直接返回，**一次分配器调用都没有**。
2. 未命中 → **Layer 2**：24 字节向上取整到 **32 字节的大小类**，
   找到正在服务 32 字节的 pool，从它的空闲链表 pop 一个 block 返回。
3. 没有可用 pool → pymalloc 从某个 arena 里切一个新 pool 出来，标记为「32 字节专用」。
4. 所有 arena 都满 → **Layer 0**：`mmap` 一块新的 1 MB arena。

而 `y = [0] * 1000`（约 8 KB > 512 字节）**直接跳过 Layer 2**，走 Layer 1 的 `malloc`。
**大对象和小对象走的是两条完全不同的路**——这是后面所有内存现象的前提。

### 1.2 横向：pymalloc 内部的 arena → pool → block

这一级跟上面的分层没有从属关系，它只是 **Layer 2 自己的实现细节**。类比一栋公寓楼：

| 单位 | 大小 | 类比 | 说明 |
|---|---|---|---|
| **arena** | **1 MB**（3.12 起；早期 256 KB） | 一整栋楼 | 向 OS 批发的最大单位，用 `mmap` |
| **pool** | 一个内存页（常见 4 KB / 16 KB） | 一层楼 | 一个 pool 只服务**一种**大小类 |
| **block** | 16, 32, 48, …, **512 字节**，共 **32 个大小类** | 一个房间 | 实际交给对象的单元 |

核心规矩只有一条，但决定了 pymalloc 的全部性能与全部缺点：

> **一个 pool 一旦被定为「32 字节专用」，它整层就只能切成 32 字节的 block。**

- **好处**：同一 pool 内所有房间尺寸相同 → 分配就是摘链表头，
  不用搜索、不用合并相邻空闲块、每个 block 也不需要 size 头部。
  **这是 O(1) 的纯指针操作，比 `malloc` 快得多**——Python 能承受「万物皆对象」全靠它。
- **代价**：内部碎片（`sys.getsizeof(1) == 28`，实际占 32 字节，
  多出的 4 字节就是取整到大小类的浪费），以及下一节要讲的 arena 无法归还。

```console
$ PYTHONMALLOCSTATS=1 python3.13 -c "pass"
Small block threshold = 512, in 32 size classes.
1 arenas * 1048576 bytes/arena     =            1,048,576
```

> ⚠️ 常见的错误说法是「8 字节对齐、64 个大小类」——那是 **32 位**时代的数字。
> **64 位构建上是 16 字节对齐、32 个大小类**（512 ÷ 16 = 32），可用上面的命令直接验证。

## 2. 为什么内存不还给操作系统

释放走的是 §1.1 调用链**完全对称的反向路径**，而**每一层都会把内存截留下来**：

```text
del x
  → 对象进 Layer 3 的 freelist                  ← 被该类型截留，等下一个同类对象复用
  → freelist 满了，block 还给 Layer 2 的 pool    ← 被 pool 截留，等下一个同大小类的对象
  → pool 全空，还给它所属的 arena
  → 【只有整个 arena 的所有 pool 全空】才 munmap 还给 OS   ← 很难发生
```

**关键规则：只有一个 arena 内的所有 pool 全部空闲，这个 arena 才会被释放回 OS。**

最后一步是致命的：1 MB 的 arena 里只要**残留一个**存活对象（比如一个长命的字符串），
这 1 MB 就永远回不去。你 `del` 掉的内存只是从「Python 在用」变成「Python 留着以后用」，
在 OS 眼里进程占用一点没变。

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

三个截留点，正好对应上面三层：

1. **freelist 缓存**（Layer 3）：float/list/tuple/frame 等类型主动扣着空闲对象不放。
2. **arena 碎片**（Layer 2）：arena 里残留一个存活对象，整个 1 MB 就不能释放。
3. **libc 也不还**（Layer 1）：大对象走 `malloc`，glibc 自己也会缓存不还给内核
   （`M_TRIM_THRESHOLD`）。所以就算绕过 pymalloc（`PYTHONMALLOC=malloc`）也一样不降。

> **面试落点**：「Python 释放的内存去哪了？」标准答案：
> **对象被回收后内存回到 pymalloc 的池里，供后续 Python 对象复用，但通常不归还给操作系统**；
> 只有整个 arena 完全空闲时才会释放。因此 RSS 是"历史峰值"的近似，不是当前使用量。

**工程对策**——注意里面没有一条是「手动释放」，因为那不管用。
只有两条路：把峰值关进一个用完就退出的进程（**进程退出是唯一能保证归还的手段**），
或者从源头上不制造峰值：

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
list of 普通类实例        ~ 130 MB   （已吃到共享键字典 3.3 + 内联值 3.11 两轮优化）
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

## 8. fork 与 copy-on-write：为什么 `gc.freeze()` 能省内存

pre-fork 服务器（gunicorn/uWSGI 的 preload 模式）指望 **copy-on-write**：父进程加载的代码、
配置、模型权重在 fork 后由所有 worker **共享物理页**，只有被写的页才复制。

Python 有两个东西会系统性地把这些页写脏：

| 元凶 | 机制 |
|---|---|
| **引用计数** | 只读地访问一个对象也要 `ob_refcnt++/--`，对象头就在页里 → 页被写脏 |
| **循环 GC** | 每轮扫描都要**就地修改**被跟踪对象的 `gc_refs`（引用计数差值法，见 [[internals/garbage-collection]] §2） |

第二个是可以彻底消掉的：

```python
# 父进程加载完所有代码和数据、fork 之前
gc.freeze()        # 把当前所有被跟踪对象移出三代，进"永久代"，此后永不扫描
```

不 freeze 的话，worker 里一次 GC 遍历就会把继承来的几百 MB 页全部写脏，CoW 全废。
这就是 Instagram 那篇经典优化的核心（配合 `gc.disable()`）。

第一个（引用计数）在 3.12+ 由**不朽对象**部分缓解——最热的 `None`/`True`/小整数不再改计数，
见 [[internals/cpython-object-model]]。

> ⚠️ 只对 `fork` 有效。Windows 和 `spawn` 启动方式没有 CoW，这套优化完全不适用。

## 相关

- [[internals/cpython-object-model]] —— 对象头与 getsizeof
- [[internals/garbage-collection]] —— 回收时机与泄漏排查
- [[stdlib/builtin-data-structures]] —— list/dict 的底层布局
- [[language/descriptors-properties]] —— `__slots__` 的内存收益
- [[engineering/performance]] —— 性能与内存的权衡
- [[interview/question-bank-internals-concurrency]] —— 内存相关面试题
