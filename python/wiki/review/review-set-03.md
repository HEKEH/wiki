---
title: "复习题组 03 —— 数据模型一题（含完整解答）"
date: 2026-08-23
tags: [面试题, 复习, 数据模型, dunder, 魔术方法, 协议, 运算符重载, 哈希]
sources: []
---

# 复习题组 03 —— 数据模型（dunder 全景）

对应 [[interview/roadmap]] 的 **Day 6**（数据模型），主题页是 [[language/data-model]]。
**一道题串起 11 个输出 + 6 个追问**，覆盖 str/repr、eq/hash、容器协议回退、
运算符重载与就地运算、类型上查找 vs 实例上查找、`__getattr__` 兜底六大考点。
**每一行输出都在 CPython 3.11.9 上实测**。

用法：先只看「题目」自测，把 11 个输出全写下来（注意 `__new__` 的打印会插在中间，
顺序也算分），再对「参考答案」，最后读「解析」补机制。
本组带**批改记录**（实际作答 7.5 / 11），记下每问错在哪、属于哪一类错法。

| # | 主题 | 涉及页面 | 高频错点 |
|---|---|---|---|
| 1 | 数据模型 / dunder 全景 | [[language/data-model]] [[internals/cpython-object-model]] | dunder 在**类型**上查找；`__iadd__` 忘 `return self` |

六个考点的分布：

| 考点 | 高频错点 |
|---|---|
| `__str__` / `__repr__` 分工 | `print(a, b)` 走 str，容器里永远是 repr |
| `__eq__` / `__hash__` 成对 | 定义 `__eq__` 后类自动 unhashable |
| `NotImplemented` vs `False` | 返回 `False` 破坏对称性 |
| 容器协议回退边界 | `reversed()` 不在回退范围内，除非 `__len__` + `__getitem__` 都有 |
| `__iadd__` 忘记 `return self` | `x += y` 静默把 `x` 变成 `None` |
| dunder 在类型上查找 | 实例属性挂 `__len__` 对 `len()` 无效 |
| `__getattr__` 兜底 | `hasattr` 恒 True；`deepcopy` 直接炸 |

---

## 第 1 题 · 一个类，十一个输出

### 题目

```python
class Tally:
    def __new__(cls, *counts):
        print("new", counts)
        return super().__new__(cls)

    def __init__(self, *counts):
        self._c = list(counts)

    def __repr__(self):
        return f"Tally{tuple(self._c)}"

    def __str__(self):
        return "+".join(map(str, self._c))

    def __len__(self):
        return len(self._c)

    def __getitem__(self, i):
        return self._c[i]

    def __eq__(self, other):
        if not isinstance(other, Tally):
            return NotImplemented
        return self._c == other._c

    def __add__(self, other):
        return Tally(*(self._c + list(other)))

    def __iadd__(self, other):
        self._c += list(other)

    def __getattr__(self, name):
        return f"?{name}"
```

**a.** 写出下列每一行的**全部**输出（含 `__new__` 打印的行，顺序也要对）：

```python
t = Tally(1, 2)
print(t)                                   # (1)
print([t])                                 # (2)
print(len(t), bool(t), bool(Tally()))      # (3)
```

**b.** 继续：

```python
print(t == Tally(1, 2), t == (1, 2))       # (4)
print({t})                                 # (5)
```

**c.** 继续（`Tally` 没有定义 `__iter__`、`__contains__`、`__reversed__`）：

```python
print(list(t), 2 in t)                     # (6)
print(list(reversed(t)))                   # (7)
```

**d.** 继续：

```python
u = t + [3]
print(u, t)                                # (8)
t += [9]
print(t)                                   # (9)
```

**e.** 继续：

```python
v = Tally(5)
v.__len__ = lambda: 99
print(len(v), v.__len__())                 # (10)
```

**f.** 继续：

```python
print(hasattr(v, "__iter__"), hasattr(v, "nope"))   # (11)
```

### 追问

1. (4) 里 `t == (1, 2)` 为什么是 `False` 而不是 `TypeError`？如果把 `__eq__` 里的
   `NotImplemented` 改成 `False`，会坏掉什么？
2. (5) 为什么会失败？怎么修？修完之后 `Tally` 又多了什么新风险？
3. (7) 成功了。可是 [[language/data-model]] 里说 `reversed()` 对只实现 `__getitem__` 的类
   会报错——差别在哪？
4. (9) 的 bug 怎么修？为什么 `tuple` 里放 `list` 时 `t[0] += [x]` 会「既改成功又报错」？
5. (10) 为什么 `len(v)` 和 `v.__len__()` 结果不一样？说到 CPython 的哪一层？
6. (11) 两个 `True` 里哪个是「对的」？`__getattr__` 这么写在工程上有什么真实事故？

---

## 参考答案

### a

```text
new (1, 2)          ← t = Tally(1, 2)
1+2                 (1)
[Tally(1, 2)]       (2)
new ()              ← bool(Tally()) 里的构造，先于 (3) 打印
2 True False        (3)
```

- (1) `print` 走 `str()` → `__str__` → `"1+2"`。
- (2) 列表的显示走**元素的 `repr`** → `Tally(1, 2)`。同一个对象两种面孔。
- (3) `len(t)` = 2；`bool(t)` 无 `__bool__` → 退回 `__len__` = 2 → `True`；
  `Tally()` 的 `__len__` = 0 → `False`。
- `__new__` 拿到的是**和 `__init__` 完全相同的实参**（`(1, 2)` / `()`），
  且先于 `__init__` 执行。

### b

```text
new (1, 2)                          ← 表达式里的 Tally(1, 2)
True False                          (4)
TypeError: unhashable type: 'Tally' (5)
```

- (4) 左边：两个 `Tally`，比 `_c` 列表 → `True`。
  右边：`Tally.__eq__` 返回 `NotImplemented` → 解释器去试反射方法 `tuple.__eq__((1,2), t)`
  → 也是 `NotImplemented` → 最终退回**身份比较** `is` → `False`。
- (5) 定义了 `__eq__` 而没定义 `__hash__`，Python 自动把 `Tally.__hash__` 设成 `None`，
  类变成 unhashable，进不了 `set`、当不了 `dict` 键。

```python
print(Tally.__hash__)   #=> None    ← 不是「继承了 object.__hash__」，是被显式清掉
```

### c

```text
[1, 2] True    (6)
[2, 1]         (7)
```

- (6) 没有 `__iter__`，但有「接受 0,1,2,… 并在越界抛 `IndexError`」的 `__getitem__`
  → 走**老式迭代协议**回退，`list()` 可迭代。`in` 没有 `__contains__` 时也退化成线性扫描迭代。
- (7) `reversed()` **不在**迭代回退范围内，它要么要 `__reversed__`，
  要么**同时**要 `__len__` 和 `__getitem__`。`Tally` 两个都有，所以成功。

### d

```text
new (1, 2, 3)   ← __add__ 内部构造新对象
1+2+3 1+2       (8)
None            (9)
```

- (8) `t + [3]` 走 `__add__`，`list(other)` 把 `[3]` 摊平后拼接，返回**新对象**，
  原 `t` 不变。注意 `print(u, t)` 是 `str` → `1+2+3 1+2`，**不是** `Tally(1, 2, 3) Tally(1, 2)`。
- (9) `__iadd__` 改了 `self._c` 却**没有 `return self`** → 隐式返回 `None`
  → `t += [9]` 等价于 `t = t.__iadd__([9])` = `t = None`。
  对象本身其实改成功了（`_c == [1, 2, 9]`），但名字 `t` 已经丢了。**静默错，最难查的一类。**

### e

```text
new (5,)
1 99      (10)
```

- `len(v)` = 1：`len()` 走 `type(v)->tp_as_sequence->sq_length` 的 C 层槽位，
  **只在类型上查找**，实例 `__dict__` 里那个 lambda 被完全忽略。
- `v.__len__()` = 99：这是**普通属性查找**。类里的函数是非数据描述符，
  实例 `__dict__` 优先级更高 → 拿到 lambda → 99。

### f

```text
True True     (11)
```

两个都是 `__getattr__` 编出来的字符串 `"?__iter__"` / `"?nope"` 撑起来的假象：

- `Tally` **根本没有** `__iter__`（它靠 `__getitem__` 回退才可迭代），
  所以第一个 `True` 也是假的——**用 `hasattr(x, "__iter__")` 判断可迭代性本来就不可靠**，
  该用 `isinstance(x, collections.abc.Iterable)`，或者直接 `try: iter(x)`。
- 有了无条件兜底的 `__getattr__`，**任何** `hasattr` 都是 `True`，
  一切基于「有没有这个方法」的鸭子类型探测全部失效。

---

## 解析（追问答案）

### 1. `NotImplemented` vs `False`

`NotImplemented` 是**「我不懂这个类型，你去问对方」**；`False` 是**「我断言不相等」**。
返回 `False` 会掐断反射链，破坏对称性：

```python
class BadEq:
    def __eq__(self, o): return False     # ❌ 错误写法
class Odd:
    def __eq__(self, o): return True

BadEq() == Odd()    #=> False   ← BadEq 抢答了
Odd() == BadEq()    #=> True    ← 换个顺序结论就变
```

`a == b` 与 `b == a` 结果不同，等价关系被破坏，`in`、`dict` 查找、`unittest` 断言
的行为都会依赖操作数顺序。

### 2. 修 unhashable

```python
def __hash__(self):
    return hash(tuple(self._c))     # ✅ 与 __eq__ 用同一组字段
```

新风险：`Tally` 是**可变**的（`__iadd__`、`__getitem__` 背后的 list 都能改），
哈希值会随 `_c` 变化。放进 `set` 之后再修改，对象就永远找不回来了：

```python
x = Tally(1)
s = {x}
x._c.append(2)      # 改了参与哈希的字段
x in s              #=> False，桶位算错了；set 里从此躺着一个「幽灵」元素
```

正解是二选一：**要么让它不可变**（tuple 存储 + 去掉 `__iadd__`，
或直接上 `@dataclass(frozen=True)`——它会替你合成一致的 `__eq__` + `__hash__`），
**要么就别给 `__hash__`**。「可变 + 可哈希」是设计错误，不是实现细节。

### 3. `reversed()` 的边界

回退协议只覆盖**正向迭代**（`iter` / `in`）。`reversed()` 需要知道「从哪个下标开始倒着走」，
所以它的条件是 `__reversed__`，或者 `__len__` + `__getitem__` 齐全：

```python
class OnlyGet:
    def __getitem__(self, i):
        if i > 2: raise IndexError
        return i

list(OnlyGet())            #=> [0, 1, 2]   ✅ 正向可迭代
list(reversed(OnlyGet()))  # ❌ TypeError: object of type 'OnlyGet' has no len()
```

`Tally` 之所以 (7) 能过，就是因为它多了 `__len__`。**一句话记法：正向靠 `__getitem__`
就够，反向必须能报出长度。**

### 4. 修 `__iadd__` + tuple 陷阱

```python
def __iadd__(self, other):
    self._c += list(other)
    return self              # ✅ 就地运算的 dunder 必须返回 self
```

不实现 `__iadd__` 时 `a += b` 会退化成 `a = a + b`（新对象）；实现了才是就地修改。
这也解释了那个著名陷阱——

```python
tup = ([1], "x")
tup[0] += [2]
# ❌ TypeError: 'tuple' object does not support item assignment
tup
#=> ([1, 2], 'x')     ← 但列表已经改了！
```

`tup[0] += [2]` 被编译成两步：先 `tup[0].__iadd__([2])`（list 就地扩展，**成功**），
再 `tup.__setitem__(0, 结果)`（tuple 不支持，**抛错**）。
错误发生在第二步，第一步的副作用已经落地。见 [[interview/traps]]。

### 5. 类型上查找，不在实例上查找

隐式的 dunder 调用不走通用属性查找，而是直接读 `type(v)` 的 C 层槽位
（`sq_length` / `nb_add` / `tp_iter` …）。这带来两个结论：

- **实例属性挂 dunder 无效**——`v.__len__ = ...`、`v.__add__ = ...` 全都影响不到
  `len(v)`、`v + x`。要动就得改类，或者换类。
- **dunder 调用比普通方法调用快**——省掉了 `__getattribute__` → MRO → 描述符协议这一整条链路。

顺带一个推论：`__enter__` / `__exit__` 也是类上查找，所以「给实例动态塞一对上下文方法」
做不到；同理 mock 一个 dunder 必须 patch 到类上（`unittest.mock` 的 `MagicMock`
之所以能 mock dunder，就是因为它把它们配置在类型上）。

### 6. `__getattr__` 的真实事故

除了 `hasattr` 恒真，最经典的是**标准库对 dunder 的可选钩子探测**。
`copy.deepcopy` 会先 `getattr(x, "__deepcopy__", None)`——在**实例**上取，
于是被 `__getattr__` 拦到，拿到一个字符串，然后当函数调用：

```python
import copy

class G:
    def __getattr__(self, n): return f"?{n}"

copy.copy(G())        #=> 成功（见下：copy() 探测 __copy__ 用的是【类】）
copy.deepcopy(G())    # ❌ TypeError: 'str' object is not callable
```

`copy.copy` 为什么反而没事？`copy.py` 里相邻两个函数的探测位置不一致：

```python
copier = getattr(cls, "__copy__", None)      # copy()：在【类】上取
copier = getattr(x, "__deepcopy__", None)    # deepcopy()：在【实例】上取 ← 被兜底拦截
```

`copy()` 连躲两关：① `__copy__` 在**类对象** `G` 身上找，走 `type(G)` 的查找链，
与实例级的 `G.__getattr__` 无关；② 接着 `getattr(x, "__reduce_ex__")` 虽是在实例上取，
但 `object.__reduce_ex__` **真实存在**，常规查找就成功——`__getattr__` 只在常规查找
**失败后**才调，压根没触发。反证：给 `G` 加一个 metaclass 级 `__getattr__`，
`copy.copy` 立刻同样报 `TypeError`。

凡是「在实例上 `getattr(x, '<某个可选钩子>', None)`，取到就调用」的代码都会中招——
`copy` 的 `__deepcopy__`、Jupyter 的 `_repr_html_`、各种框架的 `__clause_element__` 之类。
（`pickle` 在 3.11+ 反而没事：3.11 给 `object` 加了 `__getstate__`，
在类上就找得到，`__getattr__` 根本不触发；3.10 及更早则会踩同一个坑。）
**正确写法是把 dunder 排除在兜底之外**：

```python
class G2:
    def __getattr__(self, n):
        if n.startswith("__") and n.endswith("__"):
            raise AttributeError(n)      # ✅ 让 dunder 探测正常地「找不到」
        return f"?{n}"

copy.deepcopy(G2())   #=> 成功
```

> **面试落点**：写 `__getattr__` 时必须显式 `raise AttributeError`（而不是返回默认值）
> 来放过 dunder 与私有名，否则会随机破坏 copy/pickle/框架内省。
> 这是 ORM、Mock、配置对象这类「动态属性」代码的标准防御。

---

## 批改记录

实际作答 **7.5 / 11**，另外**五行 `new (...)` 全漏**。

| # | 判定 | 错在哪 |
|---|---|---|
| (1) | ✅ | |
| (2) | ✅ | 答对了「容器显示用 repr」 |
| (3) | ✅ | |
| (4) | ✅ | |
| (5) | ❌ | 两个错叠加：把 `{t}` 当成 dict（实为 set 字面量）；不知道 `__eq__` 会把 `__hash__` 清成 `None`。答成了 `{t:Tally(1,2)}` |
| (6) | ⚠️ | `list(t)` 对；`2 in t` 答成 `False`。误以为「没有 `__contains__` 就不支持 `in`」——那也会是 `TypeError` 而非 `False`；实际退化成迭代比对**元素值** |
| (7) | ✅ | 边界题答对了（`__len__` + `__getitem__` 齐全，`reversed` 可用） |
| (8) | ❌ | **值**算对了（新对象 + 原 `t` 不变），但答成 `[1,2,3] [1,2]`——`print` 走 `__str__`，应为 `1+2+3 1+2`。(2) 的知识点没贯彻到多参数 `print` |
| (9) | ❌ | 答 `[1,2,9]`：看懂了就地修改，漏了 `+=` 是「调 `__iadd__` + **重新绑定**」两步，`__iadd__` 无 `return self` → `t` 被绑成 `None` |
| (10) | ✅ | 槽位查找 vs 属性查找答对，本题最能体现理解深度的一问 |
| (11) | ✅ | |
| `new` 行 | ❌ | 五行副作用输出全漏 |

**错法归类**：(8)(9) 与 `new` 行漏答是**同一个答题习惯**——答的是「对象里装着什么」，
而题目问的是「这一行会打印什么」。只算表达式的值、不算求值过程中的副作用与重新绑定，
在「一段代码问你输出」这类题里会成片失分。(9) 尤其致命：只盯 `_c` 的内容，
就完全看不到名字 `t` 已经不指向那个对象了。

**掌握扎实的**：(7) 与 (10)——`reversed` 的回退边界、以及 dunder 只在类型上查找，
这两问答对说明协议层和槽位层的机制是通的。真正的缺口在
「`__eq__`/`__hash__` 成对」(5) 和「`+=` 的两步语义」(9)，都是必考点，需要回炉。

## 一条主线：协议是「查得到」而不是「写得对」

本题 11 个输出里有 6 个的根因都是同一句话——**解释器只按固定名字、在固定位置查一次**：

| 输出 | 现象 | 根因 |
|---|---|---|
| (2) | 容器里显示 repr 而不是 str | 显示协议查的是 `__repr__`，不是「你更想给谁看」 |
| (3) | 空对象为假 | 查 `__bool__` 失败 → 查 `__len__` |
| (5) | 不能进 set | 查到 `__hash__ is None`——Python 主动清掉的 |
| (6) | 没 `__iter__` 也能迭代 | 查 `tp_iter` 失败 → 退到 `sq_item` |
| (7) | reversed 却成功了 | 查的是 `__len__` + `__getitem__` 这一对 |
| (10) | 实例上的 `__len__` 被忽略 | 只查 `type(v)` 的槽位 |

答题时不要停在「我实现了 `__eq__`」，要说清**解释器查什么、查不到时退到哪、退不动时报什么错**。
这与 [[review/review-set-02]] 的「链条断了不报错」是同一条主线的下一层：
那边是 MRO 上的查找断链，这边是槽位层的查找断链。

## 相关

- [[interview/roadmap]] —— 本组题对应 Day 6
- [[review/review-set-01]] —— 语言核心五题（Day 1–4）
- [[review/review-set-02]] —— 类、MRO 与 ABC 两题（Day 5）
- [[language/data-model]] —— 主题页
- [[language/objects-mutability]] —— 可变性与「可变 + 可哈希」为什么是设计错误
- [[language/iterators-generators]] —— 迭代协议与回退
- [[language/descriptors-properties]] —— 为什么实例 `__dict__` 能盖住类里的函数
- [[internals/cpython-object-model]] —— dunder 到 C 层 slot 的映射
- [[interview/traps]] —— tuple 里放 list 的 `+=` 陷阱
