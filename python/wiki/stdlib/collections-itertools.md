---
title: "collections / itertools / functools 工具箱"
date: 2026-08-07
tags: [collections, itertools, functools, heapq, bisect, 标准库]
sources: ["python-cheatsheet.md", "cpython-doc/functional-howto.rst"]
---

# collections / itertools / functools 工具箱

这三个模块 + `heapq` / `bisect` 是**「写出地道 Python」的核心弹药库**。
面试时能直接甩出 `Counter`、`defaultdict`、`itertools.groupby`，比手写循环加分得多。

> 本页所有 `#=>` 输出均在 CPython 3.11 实测；标注 3.12+ 的条目按官方文档语义给出。

## 1. collections

```python
from collections import (
    defaultdict, Counter, deque, namedtuple, OrderedDict, ChainMap, UserDict,
)
```

一句话选型：

| 类型 | 它解决的问题 | JS 里的近亲 |
|---|---|---|
| `defaultdict` | 键不存在时自动建默认值 | 手写 `m[k] ??= []` |
| `Counter` | 计数、词频、TopK | `lodash.countBy` |
| `deque` | 两端 O(1) 增删、有界缓冲 | 无（JS 的 `shift()` 是 O(n)） |
| `namedtuple` | 轻量不可变记录 | 冻结的 plain object |
| `ChainMap` | 多层配置覆盖，不复制 | `{...defaults, ...env, ...cli}`（但 ChainMap 不拷贝） |
| `OrderedDict` | 需要「移动到端点」的有序字典 | `Map` + 手工重排 |

### defaultdict —— 消灭 "key 不存在" 判断

`defaultdict(factory)` 在**读到不存在的键**时，调用 `factory()` 生成一个值、插进字典、再返回它。
factory 是个无参可调用对象（`list` / `int` / `set` / 自定义 lambda 都行）。

```python
from collections import defaultdict

words = ["apple", "avocado", "banana", "blueberry", "cherry"]

# ❌ 啰嗦写法
d = {}
for w in words:
    if w[0] not in d:
        d[w[0]] = []
    d[w[0]].append(w)

# ✅ defaultdict
d = defaultdict(list)
for w in words:
    d[w[0]].append(w)        # 首次访问 d['a'] 时自动变成 []

dict(d)
#=> {'a': ['apple', 'avocado'], 'b': ['banana', 'blueberry'], 'c': ['cherry']}
```

三种最常用的 factory：

```python
cnt = defaultdict(int)               # 计数器：int() == 0
for ch in "hello":
    cnt[ch] += 1
dict(cnt)                            #=> {'h': 1, 'e': 1, 'l': 2, 'o': 1}

groups = defaultdict(set)            # 去重分组：set() == set()
groups["even"].add(2); groups["even"].add(2)
dict(groups)                         #=> {'even': {2}}

tree = defaultdict(lambda: defaultdict(list))   # 嵌套两层
tree["2026"]["08"].append("event")
```

`defaultdict` 的机制是 `__missing__`——只有 `d[k]` 这种**下标读取**会触发，
`d.get(k)`、`k in d` 都不会：

```python
# ⚠️ 陷阱：读取不存在的键会【创建】它
d = defaultdict(list)
if d["missing"]:                 # 看起来只是"判断一下"
    ...
bool(d["missing"])               #=> False    条件确实为假
"missing" in d                   #=> True     但键已经被插进去了！
dict(d)                          #=> {'missing': []}

# ✅ 只读判断
d.get("missing")                 #=> None     不会插入
"missing" in d                   #=> False
```

这个副作用在「遍历字典的同时判断键」时会直接抛
`RuntimeError: dictionary changed size during iteration`。

> **面试落点**：`defaultdict` 的默认值不是"取值时返回默认"，而是 **`__missing__` 钩子在下标访问时
> 真的写入一个新条目**。所以纯查询场景要用 `.get()`，别用 `d[k]`。

### Counter —— 计数与 TopK

`Counter` 是 `dict` 的子类，值为计数。可以从任何可迭代对象构造（统计元素出现次数），
也可以像 dict 一样直接传键值。

```python
from collections import Counter

c = Counter("mississippi")
c                          #=> Counter({'i': 4, 's': 4, 'p': 2, 'm': 1})
c.most_common(2)           #=> [('i', 4), ('s', 4)]     按计数降序
c.most_common()[-2:]       #=> [('p', 2), ('m', 1)]     最少的两个
c["z"]                     #=> 0    缺失键返回 0，不抛 KeyError（也不会插入！）
c.total()                  #=> 11   所有计数之和（3.10+；旧版用 sum(c.values())）
list(c.elements())         #=> ['m','i','i','i','i','s','s','s','s','p','p']  展开回元素
```

增减计数：

```python
c = Counter("ab")
c.update("aaa")            # 累加（不是 dict.update 的覆盖语义！）
c                          #=> Counter({'a': 4, 'b': 1})
c.subtract("bbb")          # 相减，允许出现负数
c                          #=> Counter({'a': 4, 'b': -2})
```

四则 / 集合运算——**丢弃计数 ≤ 0 的项**，这点很容易考：

```python
a = Counter("aab")         #=> Counter({'a': 2, 'b': 1})
b = Counter("abc")         #=> Counter({'a': 1, 'b': 1, 'c': 1})

a + b                      #=> Counter({'a': 3, 'b': 2, 'c': 1})   计数相加
a - b                      #=> Counter({'a': 1})    b 和 c 的结果为 0，被丢弃
a & b                      #=> Counter({'a': 1, 'b': 1})   取 min（交集）
a | b                      #=> Counter({'a': 2, 'b': 1, 'c': 1})   取 max（并集）

+Counter(a=1, b=-1, c=0)   #=> Counter({'a': 1})    一元 + 用来清洗掉非正计数
```

注意 `a - b` 和 `a.subtract(b)` 语义不同：前者丢负数并返回新对象，后者原地修改且保留负数。

```python
# 词频 TopK —— 面试常考的一行解
text = "the quick brown fox jumps over the lazy dog the end"
Counter(text.lower().split()).most_common(3)
#=> [('the', 3), ('quick', 1), ('brown', 1)]

# 判断两个字符串是否为变位词（anagram）
Counter("listen") == Counter("silent")      #=> True
```

> **面试落点**：`most_common(n)` 内部用 `heapq.nlargest`（O(n log k)），不传 n 时退化为 `sorted`
> （O(n log n)）。`Counter` 的减法**默认丢弃非正计数**，要保留负数得用 `.subtract()`。

### deque —— 双端队列 / 滑动窗口

`deque` 是块状双向链表：两端 `append/pop` 都是 **O(1)**，
但**随机索引 `d[i]` 是 O(n)**——这正好和 `list` 相反（list 索引 O(1)、头部插删 O(n)）。

```python
from collections import deque

d = deque([1, 2, 3], maxlen=3)   # maxlen：有界队列
d.append(4)                #=> deque([2, 3, 4], maxlen=3)   满了则挤掉【左】端
d.appendleft(0)            #=> deque([0, 2, 3], maxlen=3)   从左推入则挤掉【右】端
d.pop()                    #=> 3     右端弹出，O(1)
d.popleft()                #=> 0     左端弹出，O(1)
```

`rotate` 和 `extendleft` 各有一个反直觉点：

```python
d = deque([1, 2, 3])
d.rotate(1)                #=> deque([3, 1, 2])    正数向【右】转，末尾绕到开头
d.rotate(-1)               #=> deque([1, 2, 3])    负数向左，转回来了

d = deque([1, 2, 3])
d.extendleft([4, 5])       #=> deque([5, 4, 1, 2, 3])
# ⚠️ 逐个 appendleft，所以插入的部分是【逆序】的
```

典型场景：

```python
# 1) BFS 队列——list.pop(0) 是 O(n)，deque.popleft() 是 O(1)
queue = deque([start])
while queue:
    node = queue.popleft()
    queue.extend(graph[node])

# 2) 有界缓冲：最近 N 条日志，天然不会内存泄漏
recent = deque(maxlen=100)
recent.append(log_line)    # 永远只留最后 100 条

# 3) 固定长度滑动窗口
win = deque(maxlen=3)
out = []
for x in "ABCDE":
    win.append(x)
    if len(win) == 3:
        out.append("".join(win))
out                        #=> ['ABC', 'BCD', 'CDE']
```

> **面试落点**：被问「Python 怎么实现队列」时，答 **`collections.deque`（append/pop 是原子的，
> 单纯做队列无需加锁）**；需要阻塞语义或跨进程才上 `queue.Queue` / `asyncio.Queue`。
> 千万别答 `list.pop(0)`——那是 O(n)。

### namedtuple —— 轻量不可变记录

```python
from collections import namedtuple

Point = namedtuple("Point", "x y")        # 字段名也可传 ["x", "y"]
p = Point(1, 2)

p                      #=> Point(x=1, y=2)     repr 自带字段名，比裸 tuple 好读
p.x                    #=> 1                   按名访问
p[0]                   #=> 1                   它仍然是 tuple，支持索引
x, y = p               # 解包照常
p._replace(x=9)        #=> Point(x=9, y=2)     返回新对象（不可变，原 p 不变）
p._asdict()            #=> {'x': 1, 'y': 2}
Point._fields          #=> ('x', 'y')
p == (1, 2)            #=> True                和普通 tuple 相等！
```

因为它就是 tuple，所以能当 dict 键、能进 set、能被 `pickle`，内存也和 tuple 一样省。

```python
# 现代替代：typing.NamedTuple（带类型注解，推荐）
from typing import NamedTuple
class Point(NamedTuple):
    x: int
    y: int = 0          # 支持默认值

# 需要可变、需要挂方法时用 dataclass
from dataclasses import dataclass
@dataclass(frozen=True, slots=True)
class Point:
    x: int
    y: int
```

选择：**要 tuple 的解包 / 相等语义 → NamedTuple；要普通对象语义 → dataclass**。

### ChainMap —— 多层配置的覆盖链

`ChainMap` 把多个 dict 串成一条查找链，**不复制数据**——查找时从左往右找第一个命中的。

```python
from collections import ChainMap

defaults = {"debug": False, "port": 8000}
env_vars = {"port": 9000}
cli_args = {"debug": True}

cfg = ChainMap(cli_args, env_vars, defaults)   # 优先级：左 > 右
cfg["debug"]           #=> True    来自 cli_args
cfg["port"]            #=> 9000    cli_args 没有，落到 env_vars
dict(cfg)              #=> {'debug': True, 'port': 9000}
cfg.maps               #=> [{'debug': True}, {'port': 9000}, {'debug': False, 'port': 8000}]
```

和 `{**defaults, **env_vars, **cli_args}` 的区别：

- **不拷贝**：底层 dict 之后的修改会立刻反映到 ChainMap 上（字典合并是一次性快照）。
- **写入只落在第一层**：

```python
cfg["port"] = 1234
cfg.maps[0]            #=> {'debug': True, 'port': 1234}   只改了 cli_args
env_vars               #=> {'port': 9000}                  原字典没动
```

- `cfg.new_child(d)` 在最前面压一层，适合"临时覆盖后回滚"的场景（模板作用域、嵌套配置）。

### OrderedDict —— dict 已有序，它凭什么还在

Python 3.7+ 普通 dict 已保证插入序，但 `OrderedDict` 还有三处独有能力：

```python
from collections import OrderedDict

od = OrderedDict(a=1, b=2, c=3)

# 1) move_to_end：把已有键挪到某一端（手写 LRU 的关键操作，O(1)）
od.move_to_end("a")               # 默认挪到尾部
list(od)                          #=> ['b', 'c', 'a']
od.move_to_end("c", last=False)   # 挪到头部
list(od)                          #=> ['c', 'b', 'a']

# 2) popitem(last=False)：FIFO 弹出（普通 dict 的 popitem 只能弹尾部）
od.popitem(last=False)            #=> ('c', 3)

# 3) 相等比较对【顺序敏感】
OrderedDict(a=1, b=2) == OrderedDict(b=2, a=1)   #=> False
dict(a=1, b=2) == dict(b=2, a=1)                 #=> True
```

手写 LRU 缓存的骨架（面试高频）：

```python
class LRU:
    def __init__(self, cap):
        self.cap, self.d = cap, OrderedDict()

    def get(self, k):
        if k not in self.d:
            return -1
        self.d.move_to_end(k)             # 命中即刷新为最近使用
        return self.d[k]

    def put(self, k, v):
        if k in self.d:
            self.d.move_to_end(k)
        self.d[k] = v
        if len(self.d) > self.cap:
            self.d.popitem(last=False)    # 淘汰最久未用
```

## 2. itertools —— 惰性迭代器代数

```python
import itertools as it
```

核心心智：**这些函数全部返回迭代器，不返回 list**。不 `list()` 就不会真正计算，
而且**只能消费一次**。所以下面示例里的 `list(...)` 不是可有可无的装饰。

```python
r = it.count(0)
type(r)                    #=> <class 'itertools.count'>
sum(it.islice(r, 5))       #=> 10    等价 0+1+2+3+4
```

### 无限迭代器

三个永不结束的迭代器，必须靠 `islice` / `takewhile` / `zip` 来"截断"：

```python
list(it.islice(it.count(10, 2), 4))   #=> [10, 12, 14, 16]   从 10 开始，步长 2
list(it.islice(it.cycle("ABC"), 7))   #=> ['A','B','C','A','B','C','A']  循环
list(it.repeat(7, 3))                 #=> [7, 7, 7]          重复 3 次（不传次数则无限）

# ❌ 死循环
list(it.count())

# ✅ 常见搭配
list(it.takewhile(lambda x: x < 5, it.count()))   #=> [0, 1, 2, 3, 4]
list(zip("abc", it.count()))                      #=> [('a',0), ('b',1), ('c',2)]
```

`repeat` 还有个不常见但很实用的用法——给 `map` 补一个固定参数：

```python
list(map(pow, [1, 2, 3], it.repeat(2)))    #=> [1, 4, 9]   各自平方
```

### 串联、切片、复制

```python
# chain：把多个可迭代对象首尾相接（类型可以不一致）
list(it.chain("AB", [1, 2], (True,)))     #=> ['A', 'B', 1, 2, True]

# chain.from_iterable：展平【一层】嵌套 ★最常用
list(it.chain.from_iterable([[1, 2], [3], [4, 5]]))    #=> [1, 2, 3, 4, 5]
```

`chain(*lists)` 与 `chain.from_iterable(lists)` 结果相同，但后者是**惰性**的：
前者要先把 `lists` 解包成参数（生成器会被立刻耗尽，参数个数也有上限），
后者接受无限长的 `lists`。注意它**只展平一层**，且字符串会被拆成单字符：

```python
list(it.chain.from_iterable([[1, [2]], [3]]))   #=> [1, [2], 3]     内层没展开
list(it.chain.from_iterable(["ab", "cd"]))      #=> ['a','b','c','d']

# 等价的纯 Python 实现
def from_iterable(iterables):
    for itr in iterables:
        yield from itr
```

```python
# islice：迭代器的切片（生成器不支持 [::] 语法）
list(it.islice("ABCDEFG", 1, 6, 2))    #=> ['B', 'D', 'F']
list(it.islice("ABCDEFG", 3))          #=> ['A', 'B', 'C']   只给一个参数 = stop
# ⚠️ 不支持负索引；islice 会真的迭代并丢弃前面的元素（不是 O(1) 跳转）
```

```python
# tee：把一个迭代器分叉成 n 个独立迭代器
src = iter([1, 2, 3])
a, b, c = it.tee(src, 3)
list(a)                    #=> [1, 2, 3]
list(b)                    #=> [1, 2, 3]    各自都能完整遍历一遍
```

`tee` 内部共享一个缓冲区：源只被读一次，但**跑得最快和最慢的分支之间的元素必须缓存着**。
所以几个分支齐头并进时内存 O(1)；若一个分支先跑到底，等价于把整个流存成 list
（那还不如直接 `data = list(src)`）。另外**调用 tee 之后不要再动原迭代器**：

```python
src = iter(range(5))
a, b = it.tee(src)
next(src)                  #=> 0    偷走了一个元素
list(a)                    #=> [1, 2, 3, 4]   分支丢数据了
```

### 笛卡尔积与排列组合

这四个是算法题里的暴力枚举利器，产出的元素全是 tuple：

```python
list(it.product("AB", "xy"))
#=> [('A','x'), ('A','y'), ('B','x'), ('B','y')]        等价于嵌套两层 for

list(it.product("AB", repeat=2))
#=> [('A','A'), ('A','B'), ('B','A'), ('B','B')]        自己和自己求积

list(it.permutations("ABC", 2))    # 排列：有序，不重复取
#=> [('A','B'), ('A','C'), ('B','A'), ('B','C'), ('C','A'), ('C','B')]

list(it.combinations("ABC", 2))    # 组合：无序，不重复取
#=> [('A','B'), ('A','C'), ('B','C')]

list(it.combinations_with_replacement("ABC", 2))    # 组合，允许重复取同一个
#=> [('A','A'), ('A','B'), ('A','C'), ('B','B'), ('B','C'), ('C','C')]
```

规模记忆法：`permutations` = n!/(n-r)!，`combinations` = C(n,r)，
`product(xs, repeat=r)` = n^r。它们都是惰性的，但**结果规模爆炸**，别对大集合直接 `list()`。

```python
# zip_longest：zip 会在最短序列处截断，它则补齐
list(zip("ABC", "xy"))                             #=> [('A','x'), ('B','y')]  丢了 C
list(it.zip_longest("ABC", "xy", fillvalue="-"))   #=> [('A','x'), ('B','y'), ('C','-')]
```

### 过滤与累积

```python
# filterfalse：filter 的反面（保留谓词为假的）
list(filter(lambda x: x % 2, [1, 2, 3, 4]))          #=> [1, 3]
list(it.filterfalse(lambda x: x % 2, [1, 2, 3, 4]))  #=> [2, 4]

# takewhile / dropwhile：在【第一个】不满足处切一刀，之后不再判断
list(it.takewhile(lambda x: x < 3, [1, 2, 3, 1, 2]))  #=> [1, 2]
list(it.dropwhile(lambda x: x < 3, [1, 2, 3, 1, 2]))  #=> [3, 1, 2]
# ⚠️ 和 filter 的区别：filter 会逐个筛，保留后面那个 1,2；takewhile 遇到 3 就整体停止

# compress：按布尔掩码挑选，比 zip + 列表推导快
list(it.compress("ABCDEF", [1, 0, 1, 0, 1, 1]))       #=> ['A', 'C', 'E', 'F']
```

```python
# accumulate：滚动累积，默认加法 —— 前缀和
list(it.accumulate([1, 2, 3, 4]))            #=> [1, 3, 6, 10]
list(it.accumulate([3, 1, 4, 1, 5], max))    #=> [3, 3, 4, 4, 5]    前缀最大值
list(it.accumulate([1, 2, 3], initial=100))  #=> [100, 101, 103, 106]   3.8+
# 应用：累计收益、running total、区间和预处理
```

```python
# pairwise：相邻两两配对（3.10+），滑动窗口 n=2 的专用版
list(it.pairwise("ABCD"))     #=> [('A','B'), ('B','C'), ('C','D')]

# 判断序列是否严格递增
all(a < b for a, b in it.pairwise([1, 3, 5]))       #=> True
# 求相邻差值
[b - a for a, b in it.pairwise([1, 4, 9])]          #=> [3, 5]

# batched：定长分批（3.12+）★，最后一批可能不足
list(it.batched("ABCDEFG", 3))   #=> [('A','B','C'), ('D','E','F'), ('G',)]
# 应用：批量写库、分页调 API，避免一次性把整个列表读进内存
```

### groupby —— 最容易写错的那个

`groupby` 扫描序列，把**相邻的**、key 相同的元素归为一组，产出 `(key, group_iterator)`。

```python
records = [
    {"dept": "eng", "n": 1},
    {"dept": "hr",  "n": 2},
    {"dept": "eng", "n": 3},
]

# ❌ 不排序：两个 eng 不相邻，被拆成了两组
[(k, [r["n"] for r in g]) for k, g in it.groupby(records, key=lambda r: r["dept"])]
#=> [('eng', [1]), ('hr', [2]), ('eng', [3])]

# ✅ 先按同一个 key 排序
data = sorted(records, key=lambda r: r["dept"])
[(k, [r["n"] for r in g]) for k, g in it.groupby(data, key=lambda r: r["dept"])]
#=> [('eng', [1, 3]), ('hr', [2])]
```

第二个坑：**group 是共享底层迭代器的惰性对象，一旦推进到下一组就失效**：

```python
gs = [(k, g) for k, g in it.groupby("AABB")]     # 先把所有 (k, g) 收集起来
[(k, list(g)) for k, g in gs]
#=> [('A', []), ('B', [])]      ← 全空了！group 已经被后续迭代耗尽

# ✅ 要么在循环体内立刻消费，要么一步到位：
[(k, list(g)) for k, g in it.groupby("AABB")]    #=> [('A', ['A','A']), ('B', ['B','B'])]
```

真正想按 key 分组而不关心顺序时，`defaultdict(list)` 更直接、也更快
（O(n) vs groupby 需要先排序的 O(n log n)）：

```python
groups = defaultdict(list)
for r in records:
    groups[r["dept"]].append(r)
```

那 `groupby` 什么时候才是对的选择？——**数据本身天然有序**（日志按时间、SQL 已 ORDER BY、
文件按行分块），或者你就是想**压缩连续重复段**：

```python
# 游程编码（run-length encoding）
[(k, len(list(g))) for k, g in it.groupby("aaabbc")]   #=> [('a',3), ('b',2), ('c',1)]
```

> **面试落点**：`itertools.groupby` **不排序**，只把相邻的相同键归组——这与 SQL 的
> GROUP BY 和 lodash 的 `groupBy` 都不同。忘记先 sort 是最常见的 bug。
> 而且它返回的 group 是惰性迭代器，跨组保存后会失效。不想排序就用 `defaultdict(list)` 手工分组。

### 实用配方

标准库文档的 "itertools recipes" 一节值得整节读完，这里挑三个最常用的：

```python
# 1) 分批处理（3.12 之前没有 batched 时的写法）
def batched(iterable, n):
    itr = iter(iterable)
    while batch := tuple(it.islice(itr, n)):    # 海象运算符：取到空 tuple 就停
        yield batch

list(batched("ABCDEFG", 3))    #=> [('A','B','C'), ('D','E','F'), ('G',)]

# 2) 任意长度滑动窗口（pairwise 只能 n=2）
from collections import deque
def window(seq, n):
    itr = iter(seq)
    win = deque(it.islice(itr, n), maxlen=n)    # 先填满第一个窗口
    if len(win) == n:
        yield tuple(win)
    for x in itr:
        win.append(x)                            # maxlen 自动挤掉最左端
        yield tuple(win)

list(window([1, 2, 3, 4], 3))    #=> [(1,2,3), (2,3,4)]

# 3) 去重保序（set 会打乱顺序，dict.fromkeys 不支持 key 函数）
def unique(seq, key=None):
    seen = set()
    for x in seq:
        k = key(x) if key else x
        if k not in seen:
            seen.add(k)
            yield x

list(unique([3, 1, 3, 2, 1]))                        #=> [3, 1, 2]
list(unique(["Apple", "APPLE", "b"], key=str.lower)) #=> ['Apple', 'b']

# 元素可哈希且不需要 key 时，最短写法是：
list(dict.fromkeys([3, 1, 3, 2, 1]))                 #=> [3, 1, 2]
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
| `@total_ordering` | 由 `__eq__`+`__lt__` 补全比较 | 推导出的运算符多一层调用 |
| `reduce(f, xs, init)` | 折叠 | 能用 `sum`/`math.prod` 就别用 |
| `cmp_to_key(cmp)` | 老式比较函数转 key | 迁移 Python 2 代码时用 |

### cache / lru_cache —— 记忆化

`@cache`（3.9+）就是 `@lru_cache(maxsize=None)`：无上限、不做淘汰、更快一点。
需要限制内存时用 `@lru_cache(maxsize=N)`，超出后按"最久未使用"淘汰。

```python
from functools import lru_cache, cache

@cache
def fib(n):
    return n if n < 2 else fib(n - 1) + fib(n - 2)

fib(30)              #=> 832040       无缓存时是指数级递归，这里瞬间返回
fib.cache_info()     #=> CacheInfo(hits=28, misses=31, maxsize=None, currsize=31)
fib.cache_clear()    # 清空缓存
fib.__wrapped__      # 拿到未被装饰的原函数
```

三个坑：

```python
# 1) 参数必须可哈希——list / dict / set 直接报错
@cache
def f(xs): ...
f([1, 2])            #=> TypeError: unhashable type: 'list'
# 解法：调用方传 tuple / frozenset，或在包装层做转换

# 2) 关键字与位置参数算不同的 key
f(1)  和  f(n=1)  会各缓存一份

# 3) ❌ 装饰实例方法 → self 被当成 key 存进缓存，实例永远不被回收（内存泄漏）
class Repo:
    @cache                       # ❌
    def load(self, k): ...
# ✅ 改用 cached_property，或把纯计算抽成模块级函数，或每个实例自建缓存
```

> **面试落点**：`lru_cache` 用 **dict + 双向循环链表** 实现（和手写 LRU 同构），命中 O(1)。
> 它是**挂在函数对象上的全局字典**，因此装饰实例方法会让 `self` 常驻，是真实项目里的经典内存泄漏。

### cached_property —— 只算一次的属性

第一次访问时计算，然后把结果**写进实例 `__dict__`**；之后的访问根本不再经过描述符。

```python
from functools import cached_property

class Report:
    def __init__(self, rows):
        self.rows = rows

    @cached_property
    def summary(self):
        print("computing...")        # 只会打印一次
        return sum(self.rows)

r = Report([1, 2, 3])
r.summary        # computing...  → 6
r.summary        #=> 6            没有再计算
r.__dict__       #=> {'rows': [1, 2, 3], 'summary': 6}   结果就存在实例字典里
del r.summary    # 手动"失效"，下次访问重新计算
```

因为要写 `__dict__`，所以**和 `__slots__` 不兼容**；也因为写的是实例字典，缓存随实例一起被回收
（这正是它比用 `@cache` 装饰方法安全的原因）。它**不保证线程安全**：多线程首次并发访问可能算多次。

### partial —— 固定部分参数

`partial(func, *args, **kwargs)` 返回一个新可调用对象，预先绑定了部分参数。

```python
from functools import partial

int16 = partial(int, base=16)
int16("ff")            #=> 255
int16.func             #=> <class 'int'>
int16.keywords         #=> {'base': 16}

# 位置参数从左往右绑定
add10 = partial(lambda a, b: a + b, 10)
add10(5)               #=> 15
```

比 lambda 强在两点：**更快**（少一层 Python 帧）、**可 pickle**（lambda 不行）：

```python
# ✅ multiprocessing / ProcessPoolExecutor 要把可调用对象 pickle 后送进子进程
from concurrent.futures import ProcessPoolExecutor
with ProcessPoolExecutor() as ex:
    ex.map(partial(process, config=cfg), items)      # ✅
    ex.map(lambda x: process(x, config=cfg), items)  # ❌ PicklingError

# 其他常见用法
sorted(rows, key=partial(get_field, name="date"))
callback = partial(handler, ctx=ctx)
```

`partialmethod` 是它的类内版本，用来定义"预设了某些参数的方法"。

### wraps —— 写装饰器的必备动作

不加 `@wraps`，被装饰函数的 `__name__` / `__doc__` / 注解 / `__module__` 全会变成 wrapper 的，
导致日志、`help()`、pytest 报告、依赖注入框架（FastAPI 靠签名解析参数！）全部错乱。

```python
from functools import wraps

def log(fn):
    @wraps(fn)                        # ✅ 把 fn 的元信息复制到 wrapper 上
    def wrapper(*args, **kwargs):
        print("calling", fn.__name__)
        return fn(*args, **kwargs)
    return wrapper

@log
def add(a, b):
    """Add two numbers."""
    return a + b

add.__name__     #=> 'add'                 不加 wraps 会是 'wrapper'
add.__doc__      #=> 'Add two numbers.'    不加 wraps 会是 None
add.__wrapped__  #=> 原函数（wraps 顺手挂上的，inspect.signature 靠它还原签名）
```

原理详见 [[language/decorators]]。

### singledispatch —— 按第一个参数的类型分派

近似其他语言的「函数重载」，但只看**第一个参数的运行时类型**（沿 MRO 查找最匹配的实现）。

```python
from functools import singledispatch

@singledispatch
def fmt(x):                          # 兜底实现（注册在 object 上）
    return f"obj:{x}"

@fmt.register
def _(x: list):                      # 用类型注解声明分派类型（3.7+）
    return ",".join(map(str, x))

@fmt.register
def _(x: int):
    return f"int:{x}"

fmt("a")        #=> 'obj:a'
fmt([1, 2])     #=> '1,2'
fmt(3)          #=> 'int:3'
fmt.registry.keys()    #=> dict_keys([<class 'object'>, <class 'list'>, <class 'int'>])
```

适合"给一堆不受你控制的类型加统一处理逻辑"（序列化、渲染、格式化），
避免写一长串 `isinstance` 分支。类内版本用 `@singledispatchmethod`（3.8+），
它跳过 `self`、按第二个参数分派：

```python
from functools import singledispatchmethod

class Formatter:
    @singledispatchmethod
    def fmt(self, x):
        return str(x)

    @fmt.register
    def _(self, x: list):
        return ", ".join(map(str, x))
```

### total_ordering / reduce / cmp_to_key

```python
from functools import total_ordering

@total_ordering                  # 只写 __eq__ 和 __lt__，自动补出 <= > >=
class Version:
    def __init__(self, n): self.n = n
    def __eq__(self, o): return self.n == o.n
    def __lt__(self, o): return self.n < o.n

Version(1) < Version(2)          #=> True
Version(3) >= Version(2)         #=> True    这是 total_ordering 推导出来的
# 代价：推导出的运算符要多一次函数调用；
# 性能敏感时手写全部六个，或直接用 dataclass(order=True)
```

```python
from functools import reduce

reduce(lambda a, b: a * b, [1, 2, 3, 4])          #=> 24
reduce(lambda a, b: a * b, [], 1)                 #=> 1    空序列必须给 initial，否则 TypeError

# ✅ 大多数时候有更好的内置写法
sum(xs)                    # 而不是 reduce(add, xs)
math.prod(xs)              # 而不是 reduce(mul, xs)
"".join(parts)             # 而不是 reduce(concat, parts) —— 后者是 O(n²)

# reduce 真正合适的场景：没有对应内置函数的自定义折叠
reduce(lambda a, b: a & b, [{1, 2, 3}, {2, 3}, {3}])   #=> {3}
```

```python
from functools import cmp_to_key

# 老式比较函数：返回负数 / 0 / 正数
def by_len(a, b):
    return len(a) - len(b)

sorted(["bb", "a", "ccc"], key=cmp_to_key(by_len))   #=> ['a', 'bb', 'ccc']
# Python 3 的 sorted 只接受 key（一元）不接受 cmp（二元）。
# 只有当排序规则无法表达成"每个元素映射到一个可比较的值"时才需要它
# （典型：把一组数字拼成最大的数——比较规则依赖两个元素的组合）。
```

## 4. heapq —— 优先队列

Python **没有 PriorityQueue 类型**，`heapq` 直接把**普通 list 当二叉最小堆**操作
（`h[0]` 恒为最小元素，`h` 只满足堆序，**不是排好序的 list**）。

```python
import heapq

h = [5, 1, 3]
heapq.heapify(h)                # 原地建堆 O(n)
h                               #=> [1, 5, 3]    注意：不是 [1,3,5]，只保证 h[0] 最小

heapq.heappush(h, 0)            # O(log n)
h                               #=> [0, 1, 3, 5]
h[0]                            #=> 0     查看最小值 O(1)，不弹出
heapq.heappop(h)                #=> 0     弹出最小 O(log n)

heapq.heappushpop(h, x)         # 先 push 再 pop，比分开调快（只做一次下滤）
heapq.heapreplace(h, x)         # 先 pop 再 push；堆为空会报错，而 heappushpop 不会
```

带优先级的任务队列——**元素用 tuple，堆按 tuple 逐项比较**：

```python
h = []
heapq.heappush(h, (2, "write docs"))
heapq.heappush(h, (1, "fix bug"))
heapq.heappop(h)                #=> (1, 'fix bug')    优先级数字小的先出
```

三个常用工具函数：

```python
heapq.nlargest(2, ["aaa", "b", "cc"], key=len)   #=> ['aaa', 'cc']
heapq.nsmallest(2, [5, 1, 3])                    #=> [1, 3]
list(heapq.merge([1, 4], [2, 3], [0]))           #=> [0, 1, 2, 3, 4]
# merge 惰性归并多个【已各自排序】的迭代器，内存只占 O(k)——
# 外部排序、合并多个已按时间排序的日志文件的标准手法
```

**最大堆技巧**：Python 只有最小堆，取负数即可。

```python
h = []
for v in [3, 1, 4]:
    heapq.heappush(h, -v)
-heapq.heappop(h)               #=> 4    最大的先出
# 元素是 tuple 时只取负优先级：(-priority, item)
```

**元素不可比较时**加序号打破平局——否则优先级相同时堆会去比较第二项，可能直接 `TypeError`：

```python
import itertools

counter = itertools.count()
h = []
heapq.heappush(h, (priority, next(counter), task))   # task 可能不支持 <
# 序号单调递增，既保证平局时可比较，又实现了同优先级下的 FIFO
```

> **面试落点**：TopK 问题的标准答法——
> **k 远小于 n 时用 `heapq.nlargest(k, xs)`（O(n log k)，内存 O(k)），
> k 接近 n 时直接 `sorted`（O(n log n)）**。流式数据（无法全部装进内存）只能用堆。

## 5. bisect —— 有序序列二分

前提：序列**必须已排序**。`bisect` 返回的是"插入点下标"，而不是"是否存在"。

```python
import bisect

xs = [1, 3, 5, 5, 7]
bisect.bisect_left(xs, 5)      #=> 2   插到所有相等元素的【左】边
bisect.bisect_right(xs, 5)     #=> 4   插到相等元素的【右】边（bisect 是它的别名）
bisect.bisect(xs, 4)           #=> 2   值不存在时，left 和 right 结果相同

# 由此可得：值 5 出现的次数 = bisect_right - bisect_left = 2
```

```python
ys = [1, 3, 5, 7]
bisect.insort(ys, 4)           # 插入并保持有序
ys                             #=> [1, 3, 4, 5, 7]
# ⚠️ 查找位置是 O(log n)，但 list 插入要搬移后续元素，整体仍是 O(n)。
#    高频插入场景用 heapq 或第三方 sortedcontainers。

bisect.bisect(xs, 5, key=...)  # 3.10+ 支持 key，可以对对象列表按某个字段二分
```

```python
# 分档查找（成绩转等级）—— 经典用法，比一串 if/elif 快也更好维护
grades = "FDCBA"
breaks = [60, 70, 80, 90]
[grades[bisect.bisect(breaks, s)] for s in [33, 65, 75, 85, 95]]
#=> ['F', 'D', 'C', 'B', 'A']
# 原理：bisect 算出 score 落在第几个区间，下标正好对应等级
```

`in` 对有序 list 仍是 O(n)（它并不知道你排过序），用 bisect 可降到 O(log n)：

```python
def contains(xs, x):
    i = bisect.bisect_left(xs, x)
    return i < len(xs) and xs[i] == x
```

> **面试落点**：能用 `set` 判存在就别用 bisect（O(1) vs O(log n)）。
> bisect 的价值在于 **set 做不到的"有序"查询**：前驱/后继、区间计数、分档映射、范围查找。

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
| 去重保序 | `dict.fromkeys(xs)` |
| 记忆化 | `functools.cache` |
| 惰性属性 | `functools.cached_property` |
| 固定参数（且可 pickle） | `functools.partial` |
| 多层配置覆盖 | `ChainMap` |
| 手写 LRU | `OrderedDict.move_to_end` + `popitem(last=False)` |

## 相关

- [[stdlib/builtin-data-structures]] —— 复杂度与底层实现
- [[stdlib/stdlib-essentials]] —— 其余常用标准库
- [[language/comprehensions-functional]] —— 与推导式的配合
- [[language/decorators]] —— `lru_cache` / `wraps` 的原理
- [[interview/coding-patterns]] —— 算法题里的应用
