---
title: "collections / itertools / functools 工具箱"
date: 2026-08-07
tags: [collections, itertools, functools, heapq, bisect, 标准库]
sources: ["python-cheatsheet.md", "cpython-doc/functional-howto.rst"]
---

# collections / itertools / functools 工具箱

这三个模块 + `heapq` / `bisect` 是**「写出地道 Python」的核心弹药库**。
面试时能直接甩出 `Counter`、`defaultdict`、`itertools.groupby`，比手写循环加分得多。

## 1. collections

```python
from collections import (
    defaultdict, Counter, deque, namedtuple, OrderedDict, ChainMap, UserDict,
)
```

### defaultdict —— 消灭 "key 不存在" 判断

```python
d = defaultdict(list)
for word in words:
    d[word[0]].append(word)      # 不需要先检查 key 是否存在

defaultdict(int)                 # 计数器
defaultdict(set)                 # 去重分组
defaultdict(lambda: defaultdict(list))   # 嵌套

# ⚠️ 陷阱：读取不存在的键会【创建】它
d = defaultdict(list)
if d["missing"]: ...             # 副作用：'missing' 被插入了！
"missing" in d                   #=> True
# 只读判断用 d.get(k) 或 k in d
```

### Counter —— 计数与 TopK

```python
c = Counter("mississippi")
c                          #=> Counter({'i': 4, 's': 4, 'p': 2, 'm': 1})
c.most_common(2)           #=> [('i', 4), ('s', 4)]
c["z"]                     #=> 0    缺失键返回 0，不抛 KeyError
c.total()                  #=> 11   （3.10+）
c.update("aaa"); c.subtract("ii")
Counter(a) + Counter(b)    # 加法合并
Counter(a) & Counter(b)    # 取 min（交集）
Counter(a) | Counter(b)    # 取 max（并集）
+c                         # 去掉计数 ≤ 0 的项

# 词频 TopK —— 面试常考的一行解
Counter(text.lower().split()).most_common(10)
```

### deque —— 双端队列 / 滑动窗口

```python
d = deque([1,2,3], maxlen=3)
d.append(4)                #=> deque([2, 3, 4])  超出 maxlen 自动挤掉左端
d.appendleft(0)
d.pop(); d.popleft()       # 两端都是 O(1)
d.rotate(1)                # 循环移位
d.extendleft([1,2])        # 注意：逆序插入

# 场景：BFS 队列、滑动窗口、最近 N 条日志、LRU 手写实现
from collections import deque
recent = deque(maxlen=100)     # 天然的有界缓冲，不会内存泄漏
```

### namedtuple / ChainMap / OrderedDict

```python
Point = namedtuple("Point", "x y")            # 也可传 ["x","y"]
p = Point(1, 2); p.x; x, y = p; p._replace(x=9); p._asdict()
# 现代替代：typing.NamedTuple（带类型注解）或 dataclass

# ChainMap —— 多层配置的"覆盖"链，不复制数据
cfg = ChainMap(cli_args, env_vars, defaults)  # 从左往右找第一个命中
cfg["debug"]

# OrderedDict —— dict 已有序，但它仍有独特能力
od.move_to_end(k, last=False)     # 移到头部（手写 LRU 的关键）
od.popitem(last=False)            # FIFO 弹出
OrderedDict(a=1,b=2) == OrderedDict(b=2,a=1)   #=> False  比较顺序敏感
```

## 2. itertools —— 惰性迭代器代数

```python
import itertools as it
```

### 无限迭代器

```python
it.count(10, 2)            #=> 10, 12, 14, ...
it.cycle("ABC")            #=> A, B, C, A, B, C, ...
it.repeat(x, 3)            #=> x, x, x
# 必须配合 islice / takewhile / zip 使用，否则死循环
list(it.islice(it.count(), 5))    #=> [0,1,2,3,4]
```

### 切割与组合

```python
it.chain(a, b, c)                    # 串联多个可迭代
it.chain.from_iterable(lists)        # 展平一层 ★最常用
it.islice(iterable, start, stop, step)   # 惰性切片（生成器不能用 [::]）
it.tee(iterable, 3)                  # 复制成 3 个独立迭代器（会缓存，注意内存）
it.zip_longest(a, b, fillvalue=None)
it.product(a, b, repeat=2)           # 笛卡尔积（= 嵌套 for）
it.permutations(xs, 2)               # 排列
it.combinations(xs, 2)               # 组合
it.combinations_with_replacement(xs, 2)
```

### 过滤与分组

```python
it.filterfalse(pred, xs)             # filter 的反面
it.takewhile(pred, xs)               # 取到第一个不满足为止
it.dropwhile(pred, xs)               # 丢弃到第一个不满足为止
it.compress(xs, selectors)           # 按布尔掩码选择
it.accumulate([1,2,3,4])             #=> 1, 3, 6, 10   前缀和
it.accumulate(xs, max)               # 前缀最大值
it.pairwise("ABCD")                  #=> AB, BC, CD    3.10+ 滑动窗口
it.batched("ABCDEFG", 3)             #=> ABC, DEF, G   3.12+ 分批 ★

# groupby —— ⚠️ 必须先排序！它只合并【相邻】的相同键
data = sorted(records, key=lambda r: r["dept"])
for dept, group in it.groupby(data, key=lambda r: r["dept"]):
    print(dept, list(group))          # group 是迭代器，且会被下一次迭代失效
```

> **面试落点**：`itertools.groupby` **不排序**，只把相邻的相同键归组——这与 SQL 的
> GROUP BY 和 lodash 的 `groupBy` 都不同。忘记先 sort 是最常见的 bug。
> 不想排序就用 `defaultdict(list)` 手工分组。

### 实用配方

```python
# 展平嵌套
list(it.chain.from_iterable([[1,2],[3],[4,5]]))    #=> [1,2,3,4,5]

# 分批处理（3.12 之前的写法）
def batched(iterable, n):
    itr = iter(iterable)
    while batch := tuple(it.islice(itr, n)):
        yield batch

# 滑动窗口
def window(seq, n):
    itr = iter(seq)
    win = deque(it.islice(itr, n), maxlen=n)
    if len(win) == n: yield tuple(win)
    for x in itr:
        win.append(x); yield tuple(win)

# 去重保序
def unique(seq, key=None):
    seen = set()
    for x in seq:
        k = key(x) if key else x
        if k not in seen:
            seen.add(k); yield x
```

## 3. functools

```python
from functools import (
    lru_cache, cache, cached_property, partial, reduce, wraps,
    singledispatch, singledispatchmethod, total_ordering, cmp_to_key,
)
```

| 工具 | 用途 | 注意 |
|---|---|---|
| `@cache` / `@lru_cache(maxsize=N)` | 记忆化 | 参数须可哈希；**别装饰实例方法**（泄漏） |
| `@cached_property` | 实例级惰性属性 | 与 `__slots__` 不兼容 |
| `partial(f, a, key=v)` | 固定部分参数 | 比 lambda 更快，可 pickle |
| `@wraps(fn)` | 保留元信息 | 写装饰器必加 |
| `@singledispatch` | 按第一个参数类型分派 | 类似"重载" |
| `@total_ordering` | 由 `__eq__`+`__lt__` 补全比较 | |
| `reduce(f, xs, init)` | 折叠 | 能用 `sum`/`math.prod` 就别用 |
| `cmp_to_key(cmp)` | 老式比较函数转 key | 迁移 Python 2 代码时用 |

```python
# partial 的典型用法
int16 = partial(int, base=16)
sorted(rows, key=partial(get_field, name="date"))
callback = partial(handler, ctx=ctx)     # ✅ 可以被 pickle（multiprocessing 友好），lambda 不行

# singledispatchmethod（3.8+，类内版本）
class Formatter:
    @singledispatchmethod
    def fmt(self, x): return str(x)
    @fmt.register
    def _(self, x: list): return ", ".join(map(str, x))
```

## 4. heapq —— 优先队列

Python **没有 PriorityQueue 类型**，用 `heapq` 操作普通 list（最小堆）。

```python
import heapq

h = []
heapq.heappush(h, (priority, item))     # O(log n)
heapq.heappop(h)                         # 弹出最小 O(log n)
h[0]                                     # 查看最小 O(1)
heapq.heapify(lst)                       # 原地建堆 O(n)
heapq.heappushpop(h, x)                  # 先 push 再 pop，比分开快
heapq.heapreplace(h, x)                  # 先 pop 再 push

heapq.nlargest(3, xs, key=len)           # TopK（n 小时比 sorted 快）
heapq.nsmallest(3, xs)
heapq.merge(a, b, c)                     # 归并多个【已排序】迭代器，惰性
```

**最大堆技巧**：取负数。

```python
heapq.heappush(h, -value)
-heapq.heappop(h)
```

**元素不可比较时**加序号打破平局：

```python
counter = itertools.count()
heapq.heappush(h, (priority, next(counter), task))   # task 可能不支持 <
```

> **面试落点**：TopK 问题的标准答法——
> **k 远小于 n 时用 `heapq.nlargest(k, xs)`（O(n log k)），k 接近 n 时直接 `sorted`（O(n log n)）**。

## 5. bisect —— 有序序列二分

```python
import bisect

xs = [1, 3, 5, 7]
bisect.bisect_left(xs, 5)      #=> 2   插入点（相等元素的左边）
bisect.bisect_right(xs, 5)     #=> 3   （= bisect）
bisect.insort(xs, 4)           #=> [1,3,4,5,7]   插入并保持有序（插入本身 O(n)）
bisect.bisect(xs, 5, key=...)  # 3.10+ 支持 key

# 分档查找（成绩转等级）—— 经典用法
grades = "FDCBA"
breaks = [60, 70, 80, 90]
grades[bisect.bisect(breaks, score)]
```

`in` 对有序 list 仍是 O(n)，用 bisect 可降到 O(log n)：

```python
def contains(xs, x):
    i = bisect.bisect_left(xs, x)
    return i < len(xs) and xs[i] == x
```

## 6. 一张"该用哪个"速查

| 想做的事 | 用 |
|---|---|
| 统计频次 / TopK 词 | `Counter(...).most_common(k)` |
| 按 key 分组 | `defaultdict(list)`（无需排序）或 `itertools.groupby`（需先排序） |
| 队列 / 栈 / 有界缓冲 | `deque` |
| 优先队列 / TopK 数值 | `heapq` |
| 有序插入 / 分档 | `bisect` |
| 展平嵌套列表 | `itertools.chain.from_iterable` |
| 分批 | `itertools.batched`（3.12+） |
| 滑动窗口 | `itertools.pairwise` / `deque(maxlen=n)` |
| 笛卡尔积 / 排列组合 | `itertools.product/permutations/combinations` |
| 前缀和 | `itertools.accumulate` |
| 记忆化 | `functools.cache` |
| 多层配置覆盖 | `ChainMap` |

## 相关

- [[stdlib/builtin-data-structures]] —— 复杂度与底层实现
- [[stdlib/stdlib-essentials]] —— 其余常用标准库
- [[language/comprehensions-functional]] —— 与推导式的配合
- [[language/decorators]] —— `lru_cache` / `wraps` 的原理
- [[interview/coding-patterns]] —— 算法题里的应用
