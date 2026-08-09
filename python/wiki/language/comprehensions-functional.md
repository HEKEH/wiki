---
title: "推导式、函数式与模式匹配"
date: 2026-08-07
tags: [推导式, 生成器表达式, 函数式, walrus, match, 惯用法]
sources: ["cpython-doc/functional-howto.rst", "python-cheatsheet.md"]
---

# 推导式、函数式与模式匹配

这一页是 Python **代码风格（Pythonic）** 的集中体现。面试中的手撕代码环节，
能不能写出地道的推导式直接影响评价。

## 1. 四种推导式

```python
[x*x for x in range(5)]                    #=> [0, 1, 4, 9, 16]        list
{x*x for x in range(5)}                    #=> {0, 1, 4, 9, 16}        set
{x: x*x for x in range(5)}                 #=> {0: 0, 1: 1, ...}       dict
(x*x for x in range(5))                    #=> <generator>             生成器表达式
```

带条件与多层：

```python
[x for x in nums if x > 0]                          # 过滤（if 在后）
[x if x > 0 else 0 for x in nums]                   # 映射（三元在前）
[x for x in nums if x > 0 if x % 2 == 0]            # 多个 if = and
[(i, j) for i in range(3) for j in range(3) if i != j]   # 嵌套循环：顺序同 for 嵌套
[[row[i] for row in m] for i in range(len(m[0]))]   # 矩阵转置（也可 zip(*m)）
```

**顺序记忆法**：`[表达式 for ... for ... if ...]` 中的 for/if 顺序，
和展开成嵌套 for 循环时**完全一致**：

```python
# [(i, j) for i in A for j in B]  等价于：
res = []
for i in A:
    for j in B:
        res.append((i, j))
```

### 什么时候不该用推导式

```python
# ❌ 超过两层或带复杂条件时，可读性崩塌
r = [f(x, y) for x in a if p(x) for y in g(x) if q(x, y) and r(y)]

# ✅ 老老实实写循环，或拆成生成器函数
def gen():
    for x in a:
        if not p(x): continue
        for y in g(x):
            if q(x, y) and r(y):
                yield f(x, y)

# ❌ 只为副作用用推导式（构造了没人要的列表）
[print(x) for x in items]
# ✅
for x in items: print(x)
```

## 2. Walrus 运算符 `:=`（3.8+）

**在表达式中赋值**，专治"算一次要用两次"。

```python
# 避免重复计算
if (n := len(data)) > 10:
    print(f"too long: {n}")

# 推导式里复用中间结果
results = [y for x in data if (y := transform(x)) is not None]

# while 循环读取
while (chunk := f.read(8192)):
    process(chunk)

# 正则匹配
if (m := re.search(r"\d+", s)):
    print(m.group())
```

> 前端类比：≈ JS 里 `if ((n = arr.length) > 10)`，但 Python 用 `:=` 是为了
> **避免 `=` 和 `==` 的笔误**（Python 的普通 `=` 根本不能出现在表达式里）。

## 3. 内置函数工具箱

```python
# 遍历
enumerate(items, start=1)              # (索引, 元素) —— 别再写 range(len(x))
zip(a, b)                              # 并行遍历，最短的结束就停
zip(a, b, strict=True)                 # 3.10+，长度不等直接报错（推荐！）
itertools.zip_longest(a, b, fillvalue=0)
reversed(seq)
sorted(items, key=..., reverse=True)   # 返回新 list；list.sort() 是就地

# 聚合
sum(xs) / min(xs) / max(xs, key=len) / len(xs)
any(p(x) for x in xs) / all(p(x) for x in xs)     # 短路求值
math.prod(xs)                          # 3.8+

# 映射/过滤（更常用推导式，但 map 在纯函数引用时更快更清晰）
list(map(int, strs))
list(map(str.strip, lines))
list(filter(None, xs))                 # 过滤所有假值

# 解包
a, *rest = xs
dict(zip(keys, values))
```

`zip` + 解包的经典技巧：

```python
pairs = [(1, 'a'), (2, 'b')]
nums, chars = zip(*pairs)              #=> (1, 2), ('a', 'b')    "解压"
matrix_t = list(zip(*matrix))          # 转置
```

## 4. `functools` 与 `operator`

```python
from functools import reduce, partial, lru_cache, cache, singledispatch, cmp_to_key
from operator import itemgetter, attrgetter, add, mul

reduce(add, [1,2,3,4])              #=> 10（能用 sum 就别用 reduce）
reduce(mul, range(1,6), 1)          #=> 120
partial(int, base=16)("ff")         #=> 255
sorted(rows, key=itemgetter(2, 0))  # 按第 3 列再第 1 列排
sorted(objs, key=attrgetter("age"))
```

> **风格提示**：Guido 明确表示不喜欢 `reduce`（3.0 把它从内置移进了 functools）。
> 面试里用 `sum`/`math.prod`/显式循环替代 `reduce` 是更 Pythonic 的选择。

## 5. 排序的完整套路（高频考点）

```python
# 多级排序：主升序、次降序
sorted(people, key=lambda p: (p.dept, -p.salary))

# 非数值字段的降序：分两次排（Python 的排序是【稳定的】！）
s = sorted(people, key=attrgetter("name"))              # 次要键先排
s = sorted(s, key=attrgetter("dept"), reverse=True)     # 主要键后排

# 自定义比较函数（老式 cmp）
from functools import cmp_to_key
sorted(xs, key=cmp_to_key(lambda a, b: -1 if a < b else 1))

# 按频次排序
from collections import Counter
Counter(words).most_common(3)
```

> **面试落点**：Python 的 `sorted`/`list.sort` 用 **Timsort**，最坏 O(n log n)、
> 对部分有序数据接近 O(n)，且是**稳定排序**。稳定性是"分多次排序实现多级排序"的前提。

## 6. `match` 语句（结构化模式匹配，3.10+）

**不是 switch**——它是解构 + 匹配，更接近 Rust/Scala 的 match 或 JS 的解构赋值 + 判断。

```python
def handle(event):
    match event:
        case {"type": "click", "pos": (x, y)}:          # 映射模式 + 序列模式
            return f"click at {x},{y}"
        case {"type": "key", "code": str(code)}:        # 类型模式（同时捕获）
            return f"key {code}"
        case [first, *rest]:                            # 序列解构
            return f"list of {len(rest)+1}"
        case Point(x=0, y=0):                           # 类模式
            return "origin"
        case Point(x=x, y=y) if x == y:                 # 守卫（guard）
            return "diagonal"
        case str() | bytes():                           # or 模式
            return "text"
        case _:                                         # 通配（default）
            return "unknown"
```

关键规则（易错）：

```python
CONST = 1
match v:
    case CONST:        # ❌ 这是【捕获模式】！会把 v 绑定给 CONST，永远匹配成功
        ...
    case Color.RED:    # ✅ 带点号的才是【值模式】
        ...
    case 1:            # ✅ 字面量也是值模式
        ...
```

**裸名字永远是捕获，带点的才是比较**——这是 match 最反直觉的地方，面试爱考。

映射模式默认是**部分匹配**（额外的键不影响）：

```python
match {"a": 1, "b": 2}:
    case {"a": 1}:   # ✅ 匹配成功，忽略 b
        ...
    case {"a": 1, **rest}:   # rest = {'b': 2}
        ...
```

类模式需要 `__match_args__`（dataclass 自动生成）支持位置参数：

```python
@dataclass
class Point: x: int; y: int
Point.__match_args__       #=> ('x', 'y')
match p:
    case Point(0, 0): ...  # 位置形式
```

## 7. Pythonic 惯用法速查

```python
# 交换
a, b = b, a

# 链式比较
if 0 <= x < 100: ...                       # 而非 x >= 0 and x < 100

# 三元
v = a if cond else b

# 默认值
v = d.get(k, default)
v = d.setdefault(k, [])
from collections import defaultdict
d = defaultdict(list); d[k].append(x)

# 计数
from collections import Counter
Counter(items).most_common()

# 合并 dict
{**a, **b}      /     a | b        # 3.9+

# 拼接字符串（不要用 += 在循环里！O(n²)）
"".join(parts)

# 判空
if not items: ...                          # 而非 len(items) == 0

# 多返回值
def f(): return a, b                       # 其实是返回 tuple
x, y = f()

# 就地反转/切片
xs[::-1]        xs[::2]        xs[a:b:c]
s[::-1]                                    # 字符串反转

# 展平
list(itertools.chain.from_iterable(lists))

# 去重保序
list(dict.fromkeys(items))                 # ✅ dict 3.7+ 保序
list(set(items))                           # ❌ 顺序不确定

# 条件表达式里的 or 默认值（注意假值陷阱）
name = user.name or "anonymous"            # name="" 也会变 anonymous
```

## 8. 与 JS 数组方法对照

| JS | Python |
|---|---|
| `arr.map(f)` | `[f(x) for x in arr]` / `map(f, arr)` |
| `arr.filter(p)` | `[x for x in arr if p(x)]` / `filter(p, arr)` |
| `arr.reduce(f, init)` | `functools.reduce(f, arr, init)`（多数场景用 `sum`） |
| `arr.find(p)` | `next((x for x in arr if p(x)), None)` |
| `arr.some(p)` / `every(p)` | `any(...)` / `all(...)` |
| `arr.includes(v)` | `v in arr` |
| `arr.flat()` | `itertools.chain.from_iterable` |
| `arr.slice(a, b)` | `arr[a:b]` |
| `arr.sort((a,b)=>...)` | `sorted(arr, key=...)` |
| `arr.forEach(f)` | `for x in arr: f(x)` |
| `[...new Set(arr)]` | `list(dict.fromkeys(arr))` |
| `Object.entries(o)` | `o.items()` |
| `Object.keys/values` | `o.keys()` / `o.values()` |
| `arr.join(",")` | `",".join(arr)`（注意主谓颠倒！） |

**最容易搞混的**：`",".join(arr)` 而不是 `arr.join(",")`——因为 `join` 是**字符串**的方法，
且要求元素全是 str（`",".join([1,2])` 会 TypeError，要 `",".join(map(str, xs))`）。

## 相关

- [[language/iterators-generators]] —— 生成器表达式的惰性语义
- [[language/scope-closure]] —— 推导式的独立作用域
- [[stdlib/collections-itertools]] —— itertools 完整工具箱
- [[stdlib/builtin-data-structures]] —— 各操作的复杂度
- [[interview/coding-patterns]] —— 手撕代码里的 Python 惯用法
- [[bridge/js-to-python]] —— 完整语法对照
