---
title: "数据模型与协议（Data Model / Dunder Methods）"
date: 2026-08-07
tags: [数据模型, dunder, 魔术方法, 协议, 鸭子类型, 运算符重载]
sources: ["cpython-doc/datamodel.rst", "python-cheatsheet.md"]
---

# 数据模型与协议（Data Model）

Python 的数据模型是**整门语言的骨架**。所有语法糖——`len(x)`、`x[k]`、`a + b`、`with`、`for`、
`f(x)`——都被解释器翻译成对**双下划线方法（dunder method）** 的调用。理解这一层，Python 从
"一堆语法规则"变成"一套可插拔的协议"。

> 前端类比：JS 也有类似机制（`Symbol.iterator`、`toString`、`valueOf`、Proxy trap），
> 但 JS 只暴露了零星几个；Python 把**几乎每个语法结构**都开放成了协议。

## 核心心智：语法 → dunder 调用

| 你写的 | 解释器实际做的 |
|---|---|
| `len(x)` | `type(x).__len__(x)` |
| `x[k]` | `type(x).__getitem__(x, k)` |
| `x + y` | `type(x).__add__(x, y)`，失败再试 `type(y).__radd__(y, x)` |
| `-x` | `type(x).__neg__(x)` |
| `x()` | `type(x).__call__(x)` |
| `for i in x` | `iter(x)` → `type(x).__iter__(x)`，再反复 `__next__` |
| `with x:` | `type(x).__enter__(x)` / `__exit__` |
| `x.attr` | `type(x).__getattribute__(x, 'attr')`，找不到再 `__getattr__` |
| `print(x)` | `str(x)` → `type(x).__str__(x)`，无则退回 `__repr__` |
| `k in d` | `type(d).__contains__(d, k)` |
| `await x` | `type(x).__await__(x)` |

**关键细节（高频考点）**：隐式调用的 dunder 方法在**类型上查找，不在实例上查找**。
实例属性里挂一个 `__len__` 是无效的：

```python
class Foo:
    pass

f = Foo()
f.__len__ = lambda: 42
len(f)          # ❌ TypeError: object of type 'Foo' has no len()
f.__len__()     #=> 42   显式调用才走实例属性
```

原因：CPython 走的是 `type(f)->tp_as_sequence->sq_length` 这类 C 层槽位（slot），
不走通用的属性查找。这也是为什么 dunder 调用**比普通方法调用快**。

## 分组速查

### 1. 对象生命周期

```python
class Resource:
    def __new__(cls, *a, **kw):        # 分配（返回实例），比 __init__ 早
        print("new")
        return super().__new__(cls)
    def __init__(self, name):          # 初始化（返回值必须是 None）
        print("init")
        self.name = name
    def __del__(self):                 # 引用计数归零时（不保证何时/是否调用，别当 finalizer 用）
        print("del")
```

`__new__` 是 `staticmethod`（隐式），常用于不可变类型子类化和单例。
详见 [[language/metaclasses]]。

### 2. 字符串化

```python
class Point:
    def __init__(self, x, y):
        self.x, self.y = x, y
    def __repr__(self):                # 给开发者看，目标是能 eval 回来
        return f"Point({self.x!r}, {self.y!r})"
    def __str__(self):                 # 给用户看；缺省时退回 __repr__
        return f"({self.x}, {self.y})"
    def __format__(self, spec):        # f"{p:>10}" / format(p, 'spec')
        return format(str(self), spec)

p = Point(1, 2)
repr(p)         #=> 'Point(1, 2)'
str(p)          #=> '(1, 2)'
f"{p}"          #=> '(1, 2)'
[p]             #=> [Point(1, 2)]   容器里显示的永远是 repr！
```

> **面试落点**：只实现一个的话实现 `__repr__`——因为 `str()` 会退回它，而 `repr()` 不会退回 `__str__`；
> 且容器的显示、调试器、日志都用 `repr`。

### 3. 比较与哈希（一致性陷阱）

```python
class Money:
    def __init__(self, cents): self.cents = cents
    def __eq__(self, other):
        if not isinstance(other, Money):
            return NotImplemented        # ⚠️ 返回 NotImplemented，不是 False
        return self.cents == other.cents
    def __hash__(self):
        return hash(self.cents)          # 必须与 __eq__ 一致：a == b ⟹ hash(a) == hash(b)
    def __lt__(self, other):
        return self.cents < other.cents
```

规则（必考）：

- **定义了 `__eq__` 而不定义 `__hash__`，类会变成 unhashable**（`__hash__` 被自动设为 `None`），
  不能进 `set`/当 `dict` 键。这是 Python 保护你不写出「相等但哈希不同」的 bug。
- 返回 `NotImplemented` 会让解释器去试对方的反射方法，最终退回身份比较；返回 `False` 则直接断言不等，
  破坏对称性。
- 只需实现 `__eq__` + `__lt__`，加 `@functools.total_ordering` 即可补全其余四个比较运算。

```python
from functools import total_ordering

@total_ordering
class Version:
    def __init__(self, t): self.t = t
    def __eq__(self, o): return self.t == o.t
    def __lt__(self, o): return self.t < o.t

Version((1,2)) <= Version((1,3))   #=> True，由 total_ordering 合成
```

### 4. 容器协议

```python
class Deck:
    def __init__(self, cards): self._cards = list(cards)
    def __len__(self):            return len(self._cards)
    def __getitem__(self, i):     return self._cards[i]      # 支持切片就手动处理 slice 对象
    def __setitem__(self, i, v):  self._cards[i] = v
    def __delitem__(self, i):     del self._cards[i]
    def __contains__(self, v):    return v in self._cards
    def __iter__(self):           return iter(self._cards)
    def __reversed__(self):       return reversed(self._cards)
```

**只实现 `__getitem__`（接受 0,1,2,… 并在越界时抛 `IndexError`）就已经可迭代、可 `in`**
——这是 Python 的"老式迭代协议"回退。这是经典面试题。

```python
class OnlyGetItem:
    def __getitem__(self, i):
        if i > 2: raise IndexError
        return i

list(OnlyGetItem())        #=> [0, 1, 2]   ✅ 可迭代
1 in OnlyGetItem()         #=> True        ✅ 可 in
reversed(OnlyGetItem())    # ❌ TypeError: object of type 'OnlyGetItem' has no len()
```

⚠️ **但 `reversed()` 不在回退范围内**——它要求 `__reversed__`，
或者**同时**有 `__len__` 和 `__getitem__`（因为它得知道从哪个下标开始倒着走）。
这个边界常被记混。

### 5. 运算符重载与反射

```python
class Vec:
    def __init__(self, *c): self.c = c
    def __add__(self, o):  return Vec(*(a + b for a, b in zip(self.c, o.c)))
    def __radd__(self, o): return self.__add__(o)     # 处理 other + self
    def __iadd__(self, o):                            # 处理 self += other（就地修改）
        self.c = tuple(a + b for a, b in zip(self.c, o.c))
        return self                                   # ⚠️ 必须 return
    def __mul__(self, k): return Vec(*(a * k for a in self.c))
```

- 缺省时 `a += b` 退化成 `a = a + b`；实现了 `__iadd__` 才是真正的就地修改。
  这解释了 `tuple` 里放 `list` 时 `t[0] += [1]` 会**既修改成功又抛 TypeError** 的著名陷阱
  （见 [[interview/traps]]）。
- 反射方法 `__radd__` / `__rmul__` 在左操作数不支持该运算时被调用。

### 6. 属性访问

```python
class Lazy:
    def __getattr__(self, name):            # 仅在常规查找失败后调用
        return f"computed_{name}"
    def __getattribute__(self, name):       # 拦截所有属性访问（危险，易无限递归）
        return object.__getattribute__(self, name)
    def __setattr__(self, name, v): ...
    def __delattr__(self, name): ...
    def __dir__(self): return ["a", "b"]
```

`__getattr__` vs `__getattribute__` 的区别是必考点：前者是**兜底**（找不到才调），后者是**拦截**（每次都调）。
ORM、配置对象、Mock 库大量使用 `__getattr__`。

### 7. 可调用与描述符

```python
class Multiplier:
    def __init__(self, k): self.k = k
    def __call__(self, x): return x * self.k

double = Multiplier(2)
double(21)      #=> 42
callable(double) #=> True
```

描述符协议 `__get__` / `__set__` / `__delete__` / `__set_name__` 见 [[language/descriptors-properties]]。

### 8. 上下文与异步

| 同步 | 异步 |
|---|---|
| `__iter__` / `__next__` | `__aiter__` / `__anext__` |
| `__enter__` / `__exit__` | `__aenter__` / `__aexit__` |
| — | `__await__` |

见 [[language/context-managers]]、[[concurrency/asyncio-fundamentals]]。

### 9. 其它常用

```python
__slots__      # 取消 __dict__，省内存、禁止动态属性（见 descriptors 页）
__bool__       # if x: —— 无 __bool__ 时退回 __len__，都无则恒 True
__index__      # 用作序列下标 / bin()、hex() 的整数转换
__copy__ / __deepcopy__      # copy 模块钩子
__reduce__ / __getstate__ / __setstate__   # pickle 钩子（multiprocessing 依赖它）
__class_getitem__            # 支持 MyClass[int] 这样的泛型下标
```

`__bool__` 的回退链是常考点：

```python
class Empty:
    def __len__(self): return 0

bool(Empty())   #=> False   ← 没有 __bool__，用 __len__ == 0
bool(object())  #=> True    ← 两个都没有，默认真
```

## 鸭子类型 vs 协议类型

Python 传统上是**鸭子类型**：不看你是什么类，只看你有什么方法。

```python
def total(items):
    return sum(x.price for x in items)   # 任何有 .price 的对象都行
```

Python 3.8+ 用 `typing.Protocol` 给鸭子类型加上了**静态可检查**的形状（等价于 TypeScript 的
结构化类型 / interface）：

```python
from typing import Protocol

class HasPrice(Protocol):
    price: float

def total(items: list[HasPrice]) -> float:   # mypy 能静态校验，运行时无需继承
    return sum(x.price for x in items)
```

> 前端类比：`Protocol` ≈ TypeScript 的 `interface`（结构化子类型），
> 而 `abc.ABC` ≈ 需要显式 `implements` 的名义子类型。见 [[language/typing]]。

## 面试落点汇总

> - **dunder 在类型上查找，不在实例上查找**——能答出这点说明你懂 CPython 的槽位机制。
> - **`__eq__` 与 `__hash__` 必须成对维护**，只定义前者会让对象 unhashable。
> - **`__repr__` 优先于 `__str__`**，容器显示用 repr。
> - **只实现 `__getitem__` 就能迭代**（老式协议回退）。
> - **`__getattr__` 是兜底，`__getattribute__` 是拦截。**
> - `__new__` 负责创建、`__init__` 负责初始化，不可变类型的定制必须在 `__new__`。

## 相关

- [[language/classes-mro]] —— dunder 的查找沿 MRO 进行
- [[language/descriptors-properties]] —— 属性访问背后的描述符层
- [[language/metaclasses]] —— `__new__` / `type` / 类的创建过程
- [[language/iterators-generators]] —— 迭代协议
- [[language/context-managers]] —— with 协议
- [[internals/cpython-object-model]] —— dunder 如何映射到 C 层 slot
- [[bridge/js-to-python]] —— 与 JS Symbol/Proxy 的对照
- [[sources/cpython-docs]] —— 来源：官方 datamodel.rst
