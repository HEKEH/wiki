---
title: "垃圾回收：引用计数 + 分代标记清除"
date: 2026-08-07
tags: [GC, 垃圾回收, 引用计数, 循环引用, 分代, 浮动垃圾, weakref, 内存泄漏]
sources: ["interview-python-cn.md", "cpython-doc/internaldocs-gc-3.14.6.md", "cpython-doc/internaldocs-gc-3.14.0-incremental.md", "cpython-doc/whatsnew-3.14.rst"]
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
   → 减完后 gc_refs > 0 的对象，说明存在【来自被扫描集合之外】的引用 = 存活根
3. 从这些存活根出发做可达性标记，标记到的全部存活
4. 剩下 gc_refs == 0 且未被标记的 = 只被彼此引用的垃圾环 → 回收
```

这个设计的巧妙之处：**不需要知道"根集"是什么**，靠计数差就能区分内外引用。

> ⚠️ 第 2 步里的"被扫描集合"**不等于"这一代"**：收 gen1 时 gen0 会被合并进来一起扫，
> 而 gen2 的引用算外部引用。这正是分代 GC 安全的前提，也是 §3.2 / §3.3 两个坑的根源。

### 2.1 减法到底在算什么

对任何被扫描的对象都有这个恒等式：

```text
ob_refcnt  =  来自【被扫描集合内部】的引用数  +  来自【集合外部】的引用数
```

外部引用 = 栈帧局部变量、模块全局、老一代对象、C 扩展持有的引用……
第 2 步做的就是把右边第一项**精确减掉**，于是**减完剩下的 `gc_refs` 恰好等于外部引用数**：

- `gc_refs > 0` → 集合外面还有人拿着它 → 必然存活（**存活根**）
- `gc_refs == 0` → 只有集合内部在引用它 → **嫌疑犯，还不是定罪**

### 2.2 为什么必须有第 3 步

**例一：纯垃圾环 —— 减法就够了**

```python
a = []; b = []
a.append(b); b.append(a)
del a, b
```

```text
        ┌──────┐        ┌──────┐
        │  A   │───────▶│  B   │
        │ rc=1 │◀───────│ rc=1 │
        └──────┘        └──────┘

gc_refs 初始:   A=1        B=1
B 内部指向 A → A 减 1 → A=0
A 内部指向 B → B 减 1 → B=0
→ 都是 0，且没有任何根能标记到它们 → 回收 ✅
```

**例二：减法会误判 —— 这才是第 3 步存在的理由**

```python
keep = []                    # 模块全局，来自集合外部的引用
x = []; y = []
x.append(y); y.append(x)     # x、y 互相引用
keep.append(x)               # keep 也引用 x
del x, y
```

```text
 [全局 keep] ──▶ ┌──────┐         ┌──────┐        ┌──────┐
                 │ KEEP │────────▶│  X   │───────▶│  Y   │
                 │ rc=1 │         │ rc=2 │◀───────│ rc=1 │
                 └──────┘         └──────┘        └──────┘

第 1、2 步：
  KEEP: gc_refs 1 → 集合内没人指它 → 1  ✔ 存活根
  X   : gc_refs 2 → 被 KEEP 减 1、被 Y 减 1 → 0   ← 误判为嫌疑犯！
  Y   : gc_refs 1 → 被 X 减 1 → 0                ← 误判为嫌疑犯！
```

X 和 Y 明明活着（`keep[0]` 就能访问到），`gc_refs` 却归零了——因为**指向它们的引用恰好都在被扫描集合内部**。
第 3 步从 KEEP 出发传播标记，X 被标记、再由 X 标记 Y，两者获得赦免。

| 步骤 | 作用 |
|---|---|
| 1–2（减法） | 找出**入口**：哪些对象被集合外部**直接**引用 |
| 3（可达性传播） | 顺着入口**往下捞**：被存活对象**间接**引用的也活 |
| 4 | 减法归零 **且** 没被捞到 = 真正的孤岛环 → 回收 |

> **一句话记**：减法只能证明"谁被外面**直接**引用"，证明不了"谁被外面**间接**引用"，
> 后者必须靠第 3 步的传播。

实现上 CPython 把 `gc_refs > 0` 的对象移入 `reachable` 链表，再遍历该链表把新标记到的也搬进去，
最后剩在 `unreachable` 链表里的即垃圾——`subtract_refs()` / `move_unreachable()`，
位于 `Modules/gcmodule.c`（≤3.12）、`Python/gc.c`（3.13+，`gcmodule.c` 只剩模块壳）。

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
                      #=> (2000, 10, 0)   ← Python 3.14.0–3.14.4（增量式 GC，第三位不再使用）
                      #=> (2000, 10, 10)  ← Python 3.14.5+ ★ 增量式已被回滚，退回 3.13 的分代 GC
```

含义：

- **第一个数**：`分配的容器对象数 - 释放数` 超过它时触发 gen0 回收。
  **3.13 起从 700 提高到 2000**（减少 GC 频率，因为小对象分配变快了）。
- 第一个 **10**：每 10 次 gen0 回收触发一次 gen1 回收。
- 第二个 **10**：每 10 次 gen1 回收触发一次 gen2 **全量**回收（最贵）。
  但真正触发还要过 §3.4 那道 25% 门槛。

### 3.1 版本演进：增量式 GC 上线又被回滚 ★

| 版本 | GC 实现 | `get_threshold()` |
|---|---|---|
| ≤ 3.12 | 三代分代 | `(700, 10, 10)` |
| 3.13 | 三代分代 | `(2000, 10, 10)` |
| **3.14.0 – 3.14.4** | **增量式（young + old 两代）** | `(2000, 10, 0)` |
| **3.14.5 +** | **回滚成三代分代** | `(2000, 10, 10)` |

3.14 官方 What's New 原话（`raw/cpython-doc/whatsnew-3.14.rst`）：

> **From Python 3.14.5 onwards:** Python 3.14.0-3.14.4 shipped with a new incremental GC.
> However, due to a number of reports of **significant memory pressure in production
> environments**, it has been **reverted back to the generational GC from 3.13**.

回归 issue：[gh-142516](https://github.com/python/cpython/issues/142516)。本页实测于 **3.14.6**：

```text
VER 3.14.6
get_threshold (2000, 10, 10)      ← 第三位是 10，不是 0
n stats gens 3                     ← 三代，不是增量式的两代
```

**增量式当时的设计**（`raw/cpython-doc/internaldocs-gc-3.14.0-incremental.md`，值得留档）：

- 只有 **两代**：young + old；old 拆成 `pending`（本轮未扫）和 `visited`（本轮已扫）两个链表。
- 一次完整堆扫描叫 **full scavenge**，切成若干 **increment**。每个 increment =
  ① 整个 young 代 ② old 里最久没被扫的一批 ③ **从这些对象可达、且本轮还没扫过的所有对象（传递闭包）**。
- 幸存者挪到 `visited` 尾部；`pending` 空了就**翻转 GCState 里一个 bit** 让两链表互换身份，不用遍历。
- ③ 是**正确性命门**：减法算法"will not find any cycles that are even **partly outside** of that list"，
  所以必须做传递闭包，保证 increment 里不含"半个环"。代价是 increment 实际大小不可控，
  取决于对象图连通性——大概也是内存压力回归的来源之一。
- 参数语义全变：`threshold0` 不变；`threshold1` 变成"每个 increment 取 old 代的比例"（**成反比**，
  值越大扫得越慢）；`threshold2` 被忽略、`get_threshold()` 第三位恒 0；
  **`gc.collect(1)` 语义改成"执行一个 increment"**，不再是"收 gen1"。
- 收益：大堆最大停顿降低一个数量级以上，并顺带治好了 §3.3 的跨代环问题。

> ⚠️ **面试落点**：说 `(700, 10, 10)` 仍然是绝大多数面试官期待的答案（题库和博客都是这个数），
> 主动补"3.13 起改成 2000"是加分信号。但**不要再说"3.14 是增量式 GC"**——那只对 3.14.0–3.14.4 成立，
> 3.14.5 起已经回滚。真要展示深度，就说"3.14 试过增量式，因生产环境内存压力在 3.14.5 回滚了"。

### 3.2 什么对象在哪一代

"永远不回收"的对象归宿完全不同——**不在任何代 / 赖在 gen2 / 永久代**三种，别混成一类：

| 情况 | 在哪一代 | 说明 |
|---|---|---|
| 原子对象（`int`/`str`/`float`/`bytes`） | **不在任何代** | 不被 GC 跟踪，`gc.is_tracked(1) == False`，只靠引用计数 |
| **不朽对象**（静态小整数、interned 字符串、`()`、静态类型） | **不在任何代** | 引用计数被初始化成天文数字，永不归零——见 [[internals/cpython-object-model]] |
| 普通长寿容器（module `__dict__`、类对象、函数、闭包、`sys.modules`） | **gen2**，被反复空扫 | 每次全量回收都完整遍历一遍且每次都判存活，纯浪费——§3.4 的启发式就是为它而生 |
| `gc.freeze()` 之后 | **永久代**（第 4 个链表） | 独立于 gen0/1/2，**永不扫描**——见 §3.5 |
| `gc.garbage` | gen2 挂着 | 现代 Python 基本恒空，只有 `DEBUG_SAVEALL` 才填 |

### 3.3 分代的代价（一）：跨代环要等最老那一代

老对象引用新对象（`old.append(new)`）时，环会横跨两代。**收年轻代时，来自老代的引用算"外部引用"**，
于是年轻成员被判存活、还被**晋升**上去；只有扫到最老那一代，减法才第一次算对。

```python
old = N("OLD"); gc.collect(); gc.collect()   # 推到 gen2
young = N("YOUNG")                            # gen0
old.ref = young; young.ref = old               # 跨代环
del young
```

实测（3.11.9）：

```text
gc.collect(0)  → 回收 0 个，young 被【晋升】到 gen1     ← 误判存活
gc.collect(1)  → young 晋升到 gen2
（断开所有外部引用后）
gc.collect(0) → 0     gc.collect(1) → 0
gc.collect(2) → [freed] OLD / [freed] YOUNG   回收 2 个   ✅
```

**注意方向**：没有"降级"，是 gen0 那个对象被晋升到 gen2 去等。要等多久？
gen2 全量回收约每 100 次 gen0，数量级上是 **~7 万次净容器分配**（≤3.12）/ **~20 万次**（3.13+），
再叠加 §3.4 的门槛，长跑服务里可能非常久。

这就是 tracing GC 的经典 **old→young 指针问题**，别的语言用 write barrier + remembered set 解决
（老对象指向新对象时记一笔，收年轻代时把记录当额外根）。CPython 没有 write barrier，
只能靠"扫到最老那一代"这条笨路——3.14 的增量式 GC 正是冲着它去的，但被回滚了。

### 3.4 为什么全量回收越来越罕见

除了阈值，gen2 全量回收还有一道**硬门槛**（`raw/cpython-doc/internaldocs-gc-3.14.6.md`）：

> the GC only triggers a full collection of the oldest generation if the ratio
> `long_lived_pending / long_lived_total` is above a given value (**hardwired to 25%**).
> ... doing a full collection every \<constant number\> of object creations entails a dramatic
> performance degradation ... (**building a large list of GC-tracked objects would show
> quadratic performance**)

理由：非全量回收每次检查的对象数大致恒定，而**全量回收的成本正比于长寿对象总数**，几乎无上界。
用比例代替常数才能摊销成线性。效果总结成一句就是：

> **老对象越多，每次全量回收越贵，所以做得越少。**

所以 §3.3 的"要等多久"没有上界——**越到进程后期等得越久**。

### 3.5 `gc.freeze()` 与永久代

`gc.freeze()`（3.7+）把当前所有被跟踪对象移出三代，进一个**永久代**，此后**永不扫描**。实测：

```text
freeze 前：  各代对象数 [182, 4821, 0]   freeze_count = 0
gc.freeze()
freeze 后：  各代对象数 [2, 0, 0]        freeze_count = 5001   ← 三代被搬空
新建一个 []：gen0 = 1                     ← 新对象照常进 gen0
gc.unfreeze()
解冻后：     各代对象数 [3, 0, 5001]     ← 全部落回 gen2，不是 gen0
```

**真实用途是 pre-fork 服务器**（gunicorn/uWSGI preload）：

```python
# 父进程加载完所有代码和数据、fork 之前
gc.freeze()
# worker 里 GC 再也不会去读写这些对象的 GC 头 → 内存页不被写脏 → copy-on-write 共享得以保持
```

不 freeze 的话，子进程里一次 GC 遍历就会把继承来的几百 MB 页全部写脏（`gc_refs` 要被就地修改），
CoW 全废。这就是 Instagram 那篇经典优化的核心。参见 [[internals/memory-model]]。

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

## 4. 环一定能被回收吗？三个条件 ★

"环整体在被扫描的那一代内"**不是**充分条件。精确的条件是三条，缺一不可：

```text
环能被回收  ⟺  ① 环的每个成员都在本次被扫描的集合内，
                 且【没有任何来自集合外部的引用——哪怕那个引用来自一个同样已经死掉的对象】
              ② 每个成员的类型都正确实现了 GC 协议（被跟踪 + tp_traverse 完整）
              ③ 没有 finalizer 把它复活（见 §5）
```

### 4.1 反例：浮动垃圾（floating garbage）

条件 ① 的后半句最容易漏。**环整体在 gen0，却因为被一个"已经死了的 gen2 对象"引用而收不掉**：

```python
holder = N("HOLDER"); holder.self = holder   # ★ 自环 → 引用计数永不归零
gc.collect(0); gc.collect(1)                 # 推到 gen2

x, y = N("X"), N("Y")
x.ref = y; y.ref = x                         # 环【完全在 gen0】
holder.ref = x
del x, y, holder                             # 三个全是垃圾，holder 靠自环赖在 gen2
```

实测（3.11.9）：

```text
holder 在 gen2
gc.collect(0) → 0    ← X/Y 的环整体在 gen0，仍然收不掉
gc.collect(1) → 0
gc.collect(2) → 3    ← [freed] HOLDER / X / Y   只有扫到 gen2 才一起收
```

```text
      gen2                      gen0
   ┌─────────┐            ┌────┐     ┌────┐
   │ HOLDER  │───────────▶│ X  │────▶│ Y  │
   │  ↺ 自环  │            │    │◀────│    │
   └─────────┘            └────┘     └────┘
   它自己也是垃圾！          环整体在 gen0

收 gen0 时：X.gc_refs = 2(来自 Y、来自 HOLDER) − 1(只减得到 Y) = 1 > 0
            → 判为"存活根" → 标记 Y → 整个环逃过 ❌
```

> **关键**：减法只区分"集合内 / 集合外"，**完全不区分"活 / 死"**。
> 一个已死的老对象，它的引用在扫年轻代时照样算外部引用。这就是 **浮动垃圾**——垃圾互相担保存活。

所以 §3.3 的结论要扩写：不只"环跨代"会拖延，**环在本代、但被任何老代对象（死的也算）引用**，
同样得等扫到那一代。

### 4.2 反例：C 扩展的 GC 协议不完整

这类在纯 Python 层完全看不见，是真实的永久泄漏源：

| 缺陷 | 后果 |
|---|---|
| 类型没设 `Py_TPFLAGS_HAVE_GC` | 对象**不进任何代**，穿过它的环对 GC 完全不可见 → **永久泄漏** |
| 有 `tp_traverse` 但漏访问某个字段 | 该字段指向的对象 `gc_refs` 没被减 → 误判为存活根 → 环泄漏 |
| `tp_traverse` 报告了不该报告的引用 | **过度减计数 → 误回收 → 段错误**（比泄漏更糟） |

自查：`gc.is_tracked(obj)` 看类型有没有被跟踪；`gc.set_debug(gc.DEBUG_LEAK)` 看 GC 是否真的看见了它。

> 机制性补充（3.12+）：不朽对象的引用计数是天文数字，减法后仍然巨大，**永远是存活根**，
> 从它可达的一切都被标活。这是设计使然（都是解释器静态对象），但足以说明"减法判存活"的边界。

### 4.3 特例：全局对象之间的环，运行期永远收不掉

```python
A = N("GLOBAL_A"); B = N("GLOBAL_B")
A.ref = B; B.ref = A          # 全局之间的环，不 del
```

实测：

```text
refcnt(A) = 3          （module.__dict__ 一份 + B 一份 + getrefcount 自己一份）
gc.collect() → 0       ← 收不掉
sys.modules['__main__'].__dict__ is globals()  → True
```

引用链条：`A` ← `__main__.__dict__` ← module 对象 ← `sys.modules` ← 解释器。
这份引用来自被扫集合之外，`gc_refs` 恒 > 0 → **永远是存活根**。

但最值得记住的是：**GC 眼里没有"全局对象"这个类别**，只有"被 `module.__dict__` 引用的对象"。
那份引用一撤，立刻可收：

```text
--- del A, B ---
gc.collect() → 2      ← [freed] GLOBAL_A / GLOBAL_B
```

所以"全局环泄漏"从来不是 GC 的锅——是你自己还攥着入口。工程含义见 §7。

## 5. `__del__` 与不可回收对象

Python 3.4（PEP 442）之前，**带 `__del__` 的对象若在循环里，GC 不敢回收**（不知道调用顺序），
会被丢进 `gc.garbage` 永久泄漏。

**3.4 起已修复**：循环里的 `__del__` 会被调用（顺序不保证），对象能被回收。

```python
gc.garbage      #=> []   现代 Python 里通常永远是空的
```

### 5.1 销毁不可达对象的五步

GC 拿到 `unreachable` 链表之后的完整流程（`raw/cpython-doc/internaldocs-gc-3.14.6.md`）：

```text
1. 处理并清空弱引用：指向不可达对象的弱引用被置为 None；
   callback 入队 —— 但【只调用那些自身可达的弱引用的 callback】。
   若弱引用和被引用对象都不可达 → 不执行 callback（历史原因 + callback 可能复活对象）
2. 有 legacy finalizer（tp_del）的对象 → 丢进 gc.garbage
3. 调用 tp_finalize（即 __del__），并【打上 finalized 标记，避免复活后被调第二次】
4. 处理复活的对象：重跑一遍环检测，找出仍然不可达的子集，继续
5. 调用每个对象的 tp_clear 打断内部链接 → 引用计数归零 → 真正销毁
```

第 1 步就是"循环里的弱引用 callback 不保证执行"的出处；第 3、4 步是下面的复活语义。

### 5.2 `__del__` 里复活自己：只生效一轮

```python
saved = []
class R:
    def __del__(s):
        print("__del__ of", s.t)
        saved.append(s)          # ★ 复活
a, b = R("A"), R("B"); a.ref = b; b.ref = a; del a, b
```

实测（3.11.9）：

```text
__del__ of A
__del__ of B
collect() = 0   saved = ['A', 'B']   ← 被复活，一个都没回收
saved.clear()
collect() = 2                        ← 回收了，但 __del__ 【没有再被调用】
gc.garbage = []
```

对应上面第 3 步的 finalized 标记：**`__del__` 一辈子只调一次**。
所以复活只能推迟**一轮**，第二轮对象被静默回收——想靠 `__del__` 永久续命做不到，
但"这一轮收不掉"是真的（这就是 §4 的条件 ③）。

### 5.3 解释器退出时：会收，但不保证

```python
C = N("SHUTDOWN_C"); D = N("SHUTDOWN_D")
C.ref = D; D.ref = C          # 全局环，靠退出清理
```

实测（3.11.9）——不但收了，`__del__` 里**模块全局还活着**：

```text
[freed] SHUTDOWN_C / [freed] SHUTDOWN_D
__del__ C | CONST = I am a module global | helper = helper() called | sys ok = True
```

3.4 起 CPython 不再在关闭时把模块全局强行置 `None`（老 Python 里"`__del__` 看到一堆 None"
的著名坑），而是在清理模块前后跑多轮 `gc.collect()`。**但文档明确不保证**：

> It is not guaranteed that `__del__()` methods are called for objects that still exist
> when the interpreter exits.

已知失效场合：daemon 线程被强杀（它栈帧里持有的对象）、`os._exit()`、C 静态变量持有、
模块清理顺序导致依赖的模块先被清空。

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

## 6. 弱引用（weakref）—— 打破循环的正规武器

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

关键是**它不是"让 GC 更快回收"，而是让对象根本不进 GC**：

```text
强引用：  parent(gen2) ⇄ child(gen0)   环成立 → 要等扫到 gen2（§3.3 / §4.1）
弱引用：  parent(gen2) → child(gen0) → weakref(parent)
          环被打断 → child 的 refcnt 能归零 → 【当场析构，循环 GC 完全不参与】
```

弱引用容器：

```python
weakref.WeakValueDictionary()    # 值被回收时自动删键 —— 缓存的标准做法
weakref.WeakKeyDictionary()      # 给对象挂元数据而不阻止其回收
weakref.WeakSet()                # observer / 监听器列表（最典型的泄漏源）
weakref.proxy(obj)               # 像普通引用一样用，但对象没了就抛 ReferenceError
weakref.finalize(obj, fn)        # 唯一正确的"对象死了跑清理"姿势
```

> ⚠️ 不是所有对象都能被弱引用：`int`/`str`/`tuple`/**`list`/`dict`** 都不行（`set` 反而可以），
> 自定义类默认可以（用了 `__slots__` 的要加 `"__weakref__"`）。

**实现原理、三个必踩的坑、与循环 GC 的配合** → 独立页 [[internals/weakref]]。

## 7. 真实世界的内存泄漏来源

引用计数 + GC 之后，Python **仍然会泄漏**，原因几乎都是"你还持有引用"：

| 泄漏源 | 说明 | 解法 |
|---|---|---|
| **全局缓存/字典无界增长** | 最常见 | `functools.lru_cache(maxsize=N)`、`WeakValueDictionary`、TTL 缓存 |
| **`lru_cache` 装饰实例方法** | 缓存 key 含 `self`，实例永不释放 | 用 `cached_property` 或每实例缓存 |
| **未取消的 asyncio Task** | Task 持有协程帧和闭包 | 保存引用并在关闭时 cancel |
| **闭包捕获大对象** | 回调持有整个上下文 | 只捕获需要的字段 |
| **模块级列表 append** | 日志缓冲、事件收集器 | 有界队列 `collections.deque(maxlen=N)` |
| **异常 traceback 持有帧** | `except X as e:` 后 e 在块外自动删除，但存到别处会拖住整个栈帧 | 只存 `str(e)` |
| **父子/双向强引用成环** | parent 在 gen2、child 在 gen0，跨代环要等全量回收（§3.3） | `weakref.ref(parent)` |
| **模块级对象之间的环** | 运行期永远收不掉，`gc.collect()` 也没用（§4.3） | 显式 `del` / 解引用，别靠 GC |
| **C 扩展的引用泄漏 / GC 协议不完整** | 缺 `Py_TPFLAGS_HAVE_GC` 或 `tp_traverse` 漏字段（§4.2） | `gc.is_tracked` + `tracemalloc` + 原生工具 |

## 8. 排查工具

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

## 9. 生产调优实践

```python
# ① 长期运行的服务：适度调大阈值，减少 GC 频率
gc.set_threshold(50000, 50, 50)

# ② 请求处理型服务（如 uWSGI/gunicorn worker）：
#    有些团队 gc.freeze() + 关闭自动 GC，改为每 N 个请求手动 collect
gc.disable()
# ...每处理 1000 个请求
gc.collect()

# ③ fork 前 freeze，避免 CoW 页被 GC 就地修改 gc_refs 写脏（gunicorn preload 场景）
#    原理见 §3.5 与 [[internals/memory-model]] §8
gc.freeze()

# ④ 数据处理脚本：主动 del 大对象 + gc.collect()
del huge_df
gc.collect()
```

> ⚠️ 关 GC 是有风险的高级操作——只有在**确认没有循环引用**或**能定期手动回收**时才做。
> 而"确认没有循环引用"比听起来难：§4 的三个条件里，浮动垃圾和全局环都会让"我 del 了就该释放"的
> 直觉失效。
> 面试里能说出"Instagram 曾通过 `gc.freeze()` + 禁用 GC 把内存降了 x%"这类案例会很出彩。

## 10. 标准答题模板

> **问：说说 Python 的垃圾回收机制。**
>
> 分三部分答：
>
> 1. **引用计数是主力**。每个对象有 `ob_refcnt`，归零立即释放。优点是即时、无停顿；
>    缺点是有计数开销、且**无法处理循环引用**。
> 2. **循环 GC 是补充**。用标记-清除处理容器对象之间的引用环。CPython 的实现用"引用计数差值法"
>    找出只被内部引用的对象组。原子对象（int/str）不被跟踪。
> 3. **分代是优化**。三代，基于"新对象更容易死"的分代假设，让 gen0 扫得勤、gen2 扫得少。
>    阈值默认 `(700, 10, 10)`，**3.13 起改为 `(2000, 10, 10)`**；3.14.0–3.14.4 试过增量式 GC，
   **因生产环境内存压力在 3.14.5 又回滚了**。
>
> 补一句实践：**Python 仍会因"持有引用"而泄漏**，常见于全局缓存、`lru_cache` 装饰实例方法、
> 未取消的 Task；排查用 `tracemalloc` / `objgraph` / `memray`；打破循环用 `weakref`。
>
> **想再深一层**（三个都是"背过 vs 真懂"的分水岭）：
>
> - 减法只能找"被外部**直接**引用"的入口，所以还需要第 3 步做可达性传播（§2.2）。
> - **环在本代也不一定能收**：被老代对象引用就得等扫到那一代，哪怕那个老对象自己也是垃圾
>   ——浮动垃圾（§4.1）。
> - 全量回收还有 **25% 硬门槛**：老对象越多做得越少，所以跨代环的等待时间没有上界（§3.4）。

## 相关

- ★ [[internals/weakref]] —— 弱引用的实现原理与三个坑（本页 §6 的展开）
- [[internals/cpython-object-model]] —— 引用计数的实现、不朽对象
- [[internals/gil]] —— 引用计数与 GIL 的因果
- [[internals/memory-model]] —— 回收后内存是否还给操作系统
- [[language/context-managers]] —— 为什么不用 `__del__` 做清理
- [[language/decorators]] —— `lru_cache` 的泄漏陷阱
- [[interview/question-bank-internals-concurrency]] —— GC 面试题
- [[sources/cpython-docs]] —— 来源：CPython `InternalDocs/garbage_collector.md`、What's New in 3.14
- [[sources/interview-python-cn]] —— 来源：中文面试题库第 24 题
