---
title: "复习题组 01 —— 语言核心五题（含完整解答）"
date: 2026-08-17
tags: [面试题, 复习, 对象模型, 作用域, 参数, 装饰器, 生成器, 上下文管理器]
sources: []
---

# 复习题组 01 —— 语言核心五题

对应 [[interview/roadmap]] 的 **Day 1–4**。五道题各自是一个主题的综合体检，
**每道题的每一行输出都在 CPython 3.13.3 上实测过**（版本差异处单独标注）。

用法：先只看「题目」自测，写完再对「参考答案」，最后读「解析」补机制。
与 [[interview/traps]] 的分工——那边是单点陷阱速查，这边是**多考点串联的综合题**，
更接近真实面试里「一段代码问你五个输出」的形态。

| # | 主题 | 涉及页面 | 高频错点 |
|---|---|---|---|
| 1 | 对象模型、可变性、拷贝 | [[language/objects-mutability]] | `copy.copy(tuple)` 是 no-op |
| 2 | 作用域、闭包、延迟绑定 | [[language/scope-closure]] | 类作用域只有最外层 iterable 可用 |
| 3 | 参数全谱与签名内省 | [[language/functions-arguments]] | `__defaults__` 分家 + 右对齐 |
| 4 | 装饰器 | [[language/decorators]] | 同步 wrapper 包协程「静默错」 |
| 5 | 迭代器/生成器/上下文管理器 | [[language/iterators-generators]] [[language/context-managers]] | `@contextmanager` 不看返回值 |

---

## 第 1 题 · 对象模型、可变性、拷贝

### 题目

```python
import copy

def tag(item, bucket=[], seen=({},)):
    bucket.append(item)
    seen[0][item] = len(bucket)
    return bucket, seen

print(tag("a"))                    # (1)
print(tag("b"))                    # (2)

t = ([1, 2], "x")
shallow = copy.copy(t)
deep = copy.deepcopy(t)
t[0].append(3)

print(shallow[0], deep[0])         # (3)
print(shallow is t)                # (4)
print(hash(t))                     # (5)
print(-7 // 2, -7 % 2, int(-7 / 2))  # (6)
```

**b.** `seen` 是 tuple，"不可变"，为什么还是被改了？根因是什么？
**c.** `copy.copy(t)` 到底做了什么？
**d.** 用 `if not x:` 判断"调用方没传参数"会在什么情况下出错？

### 参考答案

```
(1) (['a'], ({'a': 1},))
(2) (['a', 'b'], ({'a': 1, 'b': 2},))
(3) [1, 2, 3] [1, 2]
(4) True                                   ← ★ 最易错
(5) TypeError: unhashable type: 'list'
(6) -4 1 -3
```

### 解析

#### (1)(2) 可变默认值：根因是"定义时求值一次"

默认值在 `def` 执行时求值**一次**，存活在函数对象上，所有调用共享同一批对象：

```python
tag.__defaults__   #=> (['a', 'b'], ({'a': 1, 'b': 2},))   ← 两次调用的痕迹全在这
```

#### (4) `copy.copy` 对不可变类型是 **no-op**

```python
# CPython copy.py
def _copy_immutable(x):
    return x
for t in (types.NoneType, int, float, bool, complex, str, tuple,
          bytes, frozenset, type, range, slice, property, ...):
    d[t] = _copy_immutable        # ← tuple 在这张表里
```

同源的一组事实（全部 `True`）：

```python
copy.copy(t) is t        #=> True
tuple(t) is t            #=> True
t[:] is t                #=> True
copy.deepcopy(t) is t    #=> False   ← 内层 list 被真复制，必须造新 tuple
```

`_deepcopy_tuple` 的规则是：逐元素深拷贝，**只有当每个元素的拷贝都 `is` 原元素时才复用原 tuple**。

⚠️ 常见误解是「浅拷贝造了新容器但共享元素」——那是 **list/dict** 的行为，tuple 连容器都不造：

```python
l = [[1], 2]
copy.copy(l) is l               #=> False   可变容器：真造新容器，共享元素
copy.copy(([1], 2)) is ([1], 2) #=> （同一对象时）True   不可变容器：直接返回原对象
```

所以 (3) 里 `shallow[0]` 跟着变，不是"浅拷贝共享内层"这么温和的原因——**它压根就是同一个 tuple**。

#### (5) 哈希是**递归**的

`tuple.__hash__` 对每个元素调 `hash()`，撞上 `list`（`__hash__ = None`）就抛 `TypeError`。

> **面试落点**：「tuple 能当 dict 的 key 吗？」——**当且仅当它递归地只含可哈希元素**。
> 这是 b 小问的另一面：不可变保证的是「引用槽位不变」，可哈希要求的是「值不变」，两者不是一回事。

#### (6) floor division 与 JS 的分歧

Python 保证 `a == (a // b) * b + (a % b)`，且 **`a % b` 的符号跟随除数 `b`**，
所以 `//` 是**向下取整**（floor）而非截断（trunc）。

```python
-7 // 2    #=> -4     floor
-7 % 2     #=>  1     符号跟随除数
int(-7/2)  #=> -3     先浮点除再截断
```

对照 JS：`-7 % 2 === -1`（符号跟随**被除数**）。移植代码算哈希桶、分页、星期几时的静默 bug 源。

#### b. 用 tuple 包一层是**零防护**

两句话才完整：

1. **默认值只求值一次** → 所有调用共享同一批对象（根因）；
2. **tuple 的不可变性只作用于结构，不递归到元素** → 装在里面的 dict 照样能被就地变更。

修法仍是 `None` 哨兵；当 `None` 本身是合法取值时用 `_MISSING = object()`（见 [[language/functions-arguments]]）。

#### d. Python 的假值集比 JS 大得多

```python
False 0 0.0 0j '' b'' [] {} () set() frozenset() range(0) None
```

外加任何 `__bool__`/`__len__` 返回假的对象。

**JS 里不存在这个坑**——`[]` 和 `{}` 在 JS 是**真值**，`if (!arr)` 不会误判空数组。
这是前端转 Python 的最高频踩坑点，面试主动说出这个对照是加分项（见 [[bridge/js-to-python]]）。

语义上要立起来的区分是「**没传** vs **传了空值**」：`if not x` 混为一谈，`if x is None` 才分得开。

---

## 第 2 题 · 作用域、闭包、延迟绑定

### 题目

```python
x = "global"
vals = [1, 2, 3]

class A:
    x = "class"
    doubled = [v * 2 for v in vals]                # (1)
    tripled = [v * n for v in vals for n in vals]  # (2)
    def show(self): return x                       # (3)

def outer():
    x = "enclosing"
    def rebind(): x = "inner"
    def read(): return x
    rebind()
    return read()                                  # (4)

def f():
    print(x)
    x = 1
f()                                                # (5)

fs = [lambda: i for i in range(3)]
print([g() for g in fs])                           # (6)

gens = [(i for _ in range(2)) for i in range(3)]
print([list(g) for g in gens])                     # (7)
```

**b.** (5) 报什么错？哪个 dunder 属性能证明原因？
**c.** (6) 的三种修法。
**d.** 生成器表达式/推导式里，哪些部分是**创建时**求值、哪些是**迭代时**求值？
**e.** 为什么类作用域不参与闭包？

### 参考答案

```
(1) [2, 4, 6]
(2) [1, 2, 3, 2, 4, 6, 3, 6, 9]            ← ★ 见下面的分岔
(3) global
(4) enclosing
(5) UnboundLocalError
(6) [2, 2, 2]
(7) [[2, 2], [2, 2], [2, 2]]
```

### 解析

#### (2) 关键分岔：`vals` 是**全局**还是**类变量**

```python
vals = [1, 2, 3]                                    # ← 模块级
class A:
    tripled = [v * n for v in vals for n in vals]   # ✅ [1,2,3,2,4,6,3,6,9]

class B:
    vals = [1, 2, 3]                                # ← 类变量
    ok    = [v * 2 for v in vals]                   # ✅ [2, 4, 6]
    bad   = [v * n for v in vals for n in vals]     # ❌ NameError: name 'vals' is not defined
    worse = [v for v in vals if v < len(vals)]      # ❌ NameError
```

推导式被编译成一个**隐式函数**，其自由变量查找是 `L → E(外层函数) → G → B`，**跳过类作用域**：

- 名字在**模块级** → `LOAD_GLOBAL` 找得到 → 全部成立；
- 名字是**类变量** → 隐式函数里查不到 → **只有最外层 iterable 能用**（它在类作用域求值后作为参数 `.0` 传入），其余全炸。

> **面试落点**：类作用域的坑不是「推导式不能用类变量」，而是「**只有最外层 iterable 能用**」。
> 这条本身是 d 那条更普适规则的推论。

#### (5) 局部性是**编译期**决定的

```python
f.__code__.co_varnames   #=> ('x',)     ← x 被编译成局部变量的铁证
dis.dis(f)                              #=> LOAD_FAST x（不是 LOAD_GLOBAL）
```

配套三兄弟：

| 属性 | 含义 |
|---|---|
| `co_varnames` | 局部变量名（含参数） |
| `co_freevars` | 自由变量名（来自 enclosing，读的是 cell） |
| `co_cellvars` | 本地变量中**被内层函数捕获**的那些 |

报错原文（措辞随版本变化，两种都要认得）：

- 3.11+：`cannot access local variable 'x' where it is not associated with a value`
- 3.10-：`local variable 'x' referenced before assignment`

**编译器扫完整个函数体的所有绑定目标就定案了，赋值写在哪一行无关。**

#### (6) 三种修法

```python
# ✅ 1. 默认参数：注意 i=i 写在【参数列表里】，冒号前面
fs = [lambda i=i: i for i in range(3)]

# ✅ 2. 工厂函数：多造一层真作用域，每次调用产生独立的 cell
def make(i): return lambda: i
fs = [make(i) for i in range(3)]

# ✅ 3. functools.partial：把值绑进 partial 的 args
from functools import partial
fs = [partial(lambda i: i, i) for i in range(3)]
```

⚠️ 常见语法错误：`[lambda: i=i for i in range(3)]` → `SyntaxError`。默认值属于**参数**，位置在冒号前。

三者的共同点：**把"迭代时再去外层取变量"换成"创建时就把值搬到自己身上"**。

> 对照 JS：这就是 ES5 `for (var i...) setTimeout(() => log(i))` 全打印 3 的同一个 bug。
> JS 用 `let` 的**每轮新绑定**从语言层解决了；**Python 没有 `let`，所以这坑至今存在**，
> 只能靠上面三招手动开作用域。

#### d. 求值时机：只有最外层 iterable 是急切的

```python
gen = (i * f(y) for y in EXPR if cond(y))
#                          ↑ EXPR：创建时立即求值（当参数塞进隐式函数）
#      ↑ i ↑ f ↑ cond      其余全部：迭代时才求值
```

演示：

```python
g = (x for x in undefined_name)     # ❌ 立刻 NameError
g = (undefined_name for x in [1])   # ✅ 创建成功，next(g) 时才 NameError
```

拿 (7) 验证：`range(2)` 创建时求值，输出表达式里的 `i` 迭代时才取。
而 `[list(g) for g in gens]` 在外层推导式**跑完之后**才迭代，那时 `i == 2`。

(7) 比 (6) 更隐蔽——`lambda` 至少你知道它延迟执行，生成器表达式看起来像在原地循环。

#### e. 类作用域不参与闭包

**类体执行完，它的命名空间就变成一个普通 dict 交给 `type()` 建类——编译器不为类体里的名字创建 cell，没有 cell 就没有闭包可捕获。**

- 函数的局部变量若被内层函数引用，编译器把它升级成 **cell**（进 `co_cellvars`），内层用 `LOAD_DEREF` 读。这是闭包的物理载体。
- 类体的产物是 `type(name, bases, ns)` 的第三个参数，生命周期止于建类那一刻。没有 cell，内层名字解析直接 `LOAD_GLOBAL`。

所以 (3) 拿到 `"global"` 不是"被 global 覆盖了"，而是**类作用域压根不在查找链上**。
要拿类变量只能走属性访问：`self.x` / `A.x` / `type(self).x`。

**唯一的例外**——方法里用 `super()` 或 `__class__` 时，编译器会偷偷插一个 `__class__` cell：

```python
class Meth:
    def m(self): return __class__

Meth.m.__code__.co_freevars   #=> ('__class__',)   ← 真的有个自由变量
Meth().m()                    #=> <class 'Meth'>
```

零参数 `super()` 就靠这个 cell + 第一个位置参数还原出 `super(__class__, self)`。
这也解释了**在类外定义再赋值进类的函数里，零参 `super()` 会失效**——编译时不在类体内，没有那个 cell。
详见 [[language/classes-mro]]。

---

## 第 3 题 · 参数全谱与签名内省

### 题目

```python
import inspect
from functools import singledispatch

def api(a, /, b=1, *args, c, d=2, **kw):
    return a, b, args, c, d, kw

api(1, 2, 3, 4, c=5, e=6)        # (1)
api(a=1, c=2)                    # (2)
api(1, c=2, b=3)                 # (3)
# (4) 6 个形参的 Parameter.kind 分别是什么？

def h(x, /, **kw):
    return x, kw
h(1, x=2)                        # (5)

def f(a, b=[], *, c={}, d=None): ...
f.__defaults__                   # (6)
f.__kwdefaults__                 # (7)

@singledispatch
def render(x): return "obj"
@render.register
def _(x: bool): return "bool"
@render.register
def _(x: int): return "int"

render(True), render(1), render(1.0)   # (8)
```

**b.** 把 (2) 里的 `**kw` 去掉，报错会变成另一句话。是哪一句？为什么会变？
**c.** (5) 说明 positional-only 有什么特殊用途？标准库靠它解决了什么真实问题？
**d.** `b=[]` 和 `c={}` 哪个更危险？
**e.** 「Python 支持函数重载吗」——完整回答。

### 参考答案

```
(1) (1, 2, (3, 4), 5, 2, {'e': 6})      ← *args 是 tuple
(2) TypeError: api() missing 1 required positional argument: 'a'
(3) (1, 3, (), 2, 2, {})                ← 空 args 是 () 不是 {}
(4) POSITIONAL_ONLY / POSITIONAL_OR_KEYWORD / VAR_POSITIONAL / KEYWORD_ONLY / KEYWORD_ONLY / VAR_KEYWORD
(5) (1, {'x': 2})                       ← 能过！
(6) ([],)                               ← ★ 只有 b
(7) {'c': {}, 'd': None}
(8) ('bool', 'int', 'obj')
```

### 解析

#### 收集类型对应表

| 形参 | 收集成 | 空值 |
|---|---|---|
| `*args` | **tuple** | `()` |
| `**kwargs` | **dict** | `{}` |

（`args` 是 tuple 而非 list：不可变意味着函数体不能意外改动调用方的参数包，且创建更快、可哈希。）

#### (4) `Parameter.kind`：五类参数的可编程形式

```python
def api(a, /, b=1, *args, c, d=2, **kw): ...
```

| 形参 | `.kind` | 含义 |
|---|---|---|
| `a` | `POSITIONAL_ONLY` | `/` 之前，只能按位置传 |
| `b` | `POSITIONAL_OR_KEYWORD` | `/` 与 `*` 之间，两种都行 |
| `args` | `VAR_POSITIONAL` | `*args` |
| `c` | `KEYWORD_ONLY` | `*` 之后，只能按名字传 |
| `d` | `KEYWORD_ONLY` | ⚠️ 有默认值不改变 kind |
| `kw` | `VAR_KEYWORD` | `**kwargs` |

**`kind`（能怎么传）与 `default`（有没有默认值）是两个正交维度**：

```python
p = inspect.signature(api).parameters["d"]
p.kind          #=> <_ParameterKind.KEYWORD_ONLY: 3>
p.default       #=> 2
p.annotation    #=> <class 'inspect._empty'>   ← 没写注解是 empty，不是 None
```

> **面试落点**：这个枚举是 FastAPI / pytest / typer 的地基。FastAPI 判断
> 「参数是路径参数还是查询参数还是请求体」，第一步就是遍历 `signature.parameters`
> 看 `kind` + `annotation` + `default`。见 [[web/fastapi-di]]。

#### (6)(7) `__defaults__` 分家 + 右对齐

```python
def f(a, b=[], *, c={}, d=None): ...

f.__defaults__     #=> ([],)                    ← 只有 b
f.__kwdefaults__   #=> {'c': {}, 'd': None}
```

两条规则：

1. **分家**：positional-or-keyword 的默认值进 `__defaults__`（tuple），keyword-only 的进 `__kwdefaults__`（dict）。
2. **右对齐、无占位**：`__defaults__` 只装「有默认值的那些」，从右往左对齐到形参列表末尾。
   `a` 没有默认值，在里面**没有位置**，不会用 `None` 占位。

反推形参名要这样算（这段算术就是为什么大家宁可用 `inspect.signature`）：

```python
code = f.__code__
n = len(f.__defaults__)
dict(zip(code.co_varnames[code.co_argcount - n : code.co_argcount], f.__defaults__))
#=> {'b': []}
```

#### (8) `singledispatch` 走 MRO，注册顺序无关

按第一个参数类型的 **MRO 从最具体往上找**第一个注册过的。
`bool.__mro__ == (bool, int, object)`，所以 `True` 命中 `bool` 而非 `int`——即使 `int` 后注册。

#### b. `**kwargs` 会**牺牲报错质量**

| 签名 | `api(a=1, c=2)` 的报错 |
|---|---|
| `def api(a, /, ..., **kw)` | `missing 1 required positional argument: 'a'` |
| `def api(a, /, ...)` 无 `**kw` | `got some positional-only arguments passed as keyword arguments: 'a'` |

无 `**kw` 时 CPython **专门为这种情况做了诊断**；一旦有 `**kw` 兜底，`a=1` 就是合法的 kw 项，
解释器无从知道你的意图，只能报「必需参数没传」。

> **工程含义**：拼错的关键字参数会被静默吞进 kwargs 而不是当场 `TypeError`。
> 这是「不要为了图省事随手加 `**kwargs`」的硬理由。

#### c. positional-only 最硬的用途：给 `**kwargs` 腾出名字空间

```python
d = {}
d.update(self=1, other=2)      # ✅ {'self': 1, 'other': 2}
dict(self=1, other=2)          # ✅ {'self': 1, 'other': 2}
```

`dict.update` 的真实签名是 `update(self, other=(), /, **kwds)`。若 `self`/`other` 不是 positional-only，
**任何人想拿 `"self"` 当字典 key 都会撞车**（`TypeError: got multiple values for argument 'self'`）。
同类：`functools.partial(func, /, *args, **kwargs)`、`namedtuple._replace`、`str.format`。

> **面试落点**：**当函数要收 `**kwargs` 且 key 由用户任意决定时，前面的形参必须是 positional-only。**
> PEP 570 引入 `/` 的头号动机就是这个——在此之前 CPython 只能靠 C 层的 Argument Clinic 表达这种签名。

次要用途：保留改名自由（形参名不进公共契约）、对齐内置行为（`len(obj=[1,2])` → `TypeError: len() takes no keyword arguments`）、微弱的性能收益。

#### d. 机制相同，但「危险的形状」不同

```python
def f(a, b=[], *, c={}, d=None):
    b.append(1); c['k'] = 1
    return b, c

f(0)   #=> ([1],    {'k': 1})
f(0)   #=> ([1, 1], {'k': 1})     ← ❗ dict 看起来【毫无变化】
```

- **`b=[]` 更容易暴露**：list 是累积语义，越跑越长，症状明显。
- **`c={}` 更隐蔽**：同 key 反复写入互相覆盖，内容看起来永远正常。它的翻车方式是
  「跨请求脏读」——A 用户写进去的没删，B 用户读到了。**这会变成安全漏洞而不只是逻辑错误。**

判断法则不是看类型，而是**看函数体里对这个参数是读还是写**：

```python
def run(cfg={}):
    return cfg.get("mode")        # 只读 → 侥幸没事，但仍是 ruff B006 告警
def run(cfg={}):
    cfg.setdefault("mode", "x")   # ❗ 写了 → 一定出事
```

#### e. 语言层**不支持**重载

```python
def g(x): return "one"
def g(x, y): return "two"
g(1)     # ❌ TypeError: g() missing 1 required positional argument: 'y'
```

`def` 本质是「创建函数对象 + 绑定到名字」，第二个 `def` 只是重新赋值。

`singledispatch` 是用**分派表在运行时模拟**：

```python
render.registry          # {类型: 实现} 映射表
render.dispatch(bool)    # 手动查表，返回会被调用的实现
```

三条硬限制：只按**第一个参数**类型分派；依赖运行时类型 + MRO；装实例方法要用 `singledispatchmethod`。

`typing.overload` **只对静态检查器说话**：

```python
@overload
def s(x: int) -> int: ...
@overload
def s(x: str) -> str: ...
def s(x): return x            # ← 唯一真实实现，必须写在最后
```

- 存根在运行时被最后的实现覆盖，一个都不执行。
- 单独调用一个 `@overload` 存根 → `NotImplementedError: You should not call an overloaded function...`
- 3.11+ 可用 `typing.get_overloads(s)` 取回存根（返回 2 个）。

| 工具 | 生效时机 | 解决什么 |
|---|---|---|
| `functools.singledispatch` | **运行时** | 真的要按类型走不同代码路径 |
| `typing.overload` | **静态检查时** | 实现只有一个，但入参→返回类型的对应关系要精确表达 |
| 默认参数 / `*args` | 运行时 | 只是参数个数可变，逻辑同一套 |
| `match` 语句 | 运行时 | 按结构而非类型分派 |

---

## 第 4 题 · 装饰器

### 题目

```python
def a(fn):
    print("deco a")
    def w(*ar, **kw):
        print("in a"); return fn(*ar, **kw)
    return w

def b(fn):
    print("deco b")
    def w(*ar, **kw):
        print("in b"); return fn(*ar, **kw)
    return w

@a
@b
def f():
    print("f"); return 42

print(f())                        # (1) 完整输出顺序
```

```python
def deco(fn):
    def w(*ar, **kw): return fn(*ar, **kw)
    return w

@deco
def handler(user_id: int, q: str = "x"): ...

handler.__name__                  # (2)
inspect.signature(handler)        # (3)
```

```python
class Count:
    def __init__(self, fn): self.fn, self.n = fn, 0
    def __call__(self, *ar, **kw):
        self.n += 1; return self.fn(*ar, **kw)
    def __get__(self, obj, t=None):
        return self if obj is None else partial(self.__call__, obj)

class S:
    @Count
    def m(self): pass

s1, s2 = S(), S()
s1.m(); s2.m(); s1.m()
S.m.n                             # (4)
```

```python
def timed(fn):
    @functools.wraps(fn)
    def w(*ar, **kw): return fn(*ar, **kw)
    return w

@timed
async def job(): return 1

asyncio.run(job())                     # (5)
inspect.iscoroutinefunction(job)       # (6)
```

```python
def logger(fn):
    @functools.wraps(fn)
    def w(*ar, **kw):
        fn(*ar, **kw)
    return w

@logger
def add(x, y): return x + y
add(1, 2)                         # (7)
```

**b.** (4) 暴露了什么设计缺陷？怎么做到每实例独立计数？
**c.** 删掉 `__get__` / 只删 `obj is None` 分支，分别会怎样？
**d.** `timed` 装 `async def` 的两个 bug，以及通用版写法。
**e.** 装饰器 / 上下文管理器 / 中间件的作用范围区别。

### 参考答案

```
(1) deco b → deco a → in a → in b → f → 42
(2) 'w'
(3) (*ar, **kw)
(4) 3                                    ← 全实例共享
(5) 能跑通，返回 1                        ← ★ 最易错：不报错
(6) False
(7) None
```

### 解析

#### (1) 装饰自下而上，调用自上而下

`@a @b def f` ≡ `f = a(b(f))`。**定义时**从最靠近 `def` 的开始执行（`deco b` → `deco a`），
**调用时**从最外层开始（`in a` → `in b` → `f`）。

#### (5) 同步 wrapper 包协程——**不报错，这才是灾难**

```python
@timed
async def job():
    await asyncio.sleep(0.3)
    return 1

asyncio.run(job())
#=> [timed] job 0.7us      ← ❗ 一个 300ms 的协程，测出 0.7 微秒
#=> 1                      ← 返回值完全正确
```

`async def` 调用后**不执行任何函数体**，只返回一个 coroutine 对象。同步 `w` 里的
`return fn(*ar, **kw)` 把它**原样透传**，`asyncio.run()` 拿到的还是那个 coroutine，照常 await。
整条链路没有任何环节被破坏。

```
中间件/正确装饰器  |←------ 真正的执行 ------→|
同步 wrapper 计时  |←→|  ← 只覆盖「创建协程对象」这一瞬间
```

> **面试落点**：用同步装饰器装 `async def` **不会报错**，coroutine 会被透传。
> 但所有「函数执行前后」的逻辑（计时、日志、重试、事务、异常捕获）都只作用于
> 「协程创建」这一瞬间。`try/except` 尤其危险——协程内的异常一个都抓不到，
> 因为异常发生在 `await` 时，那时 wrapper 早已退出。

#### (6) `@wraps` 修不了 `iscoroutinefunction`

`inspect.iscoroutinefunction` 检查的是**代码对象的 `CO_COROUTINE` 标志位**，不看 `__wrapped__`。
同步 wrapper 加了 `@wraps` 依然 `False`。

需要「同步 `def` 返回 coroutine 但要被识别为协程函数」时，3.12+ 有官方标记：

```python
inspect.markcoroutinefunction(w)      # 3.12+
inspect.iscoroutinefunction(w)        #=> True
```

> **面试落点**：`asyncio` 判断协程函数靠 code flags 而非 duck typing。这就是为什么很多老库的
> 装饰器一贴到 FastAPI 路由上，框架就误以为是同步函数、把它扔进线程池执行——性能塌方且不报错。

#### b. 装饰器实例是**类变量**

`@Count` 在类体里执行，产物存进 `S.__dict__['m']`，全类共享一份，`self.n` 自然全实例累加。

修法：**把状态写进对象自己的 `__dict__`**：

```python
class PerInstance:
    def __init__(self, fn):
        functools.update_wrapper(self, fn)
        self.fn = fn
        self.attr = f"_{fn.__name__}_calls"        # 每个被装饰方法一个独立 key

    def __call__(self, obj, *ar, **kw):
        obj.__dict__[self.attr] = obj.__dict__.get(self.attr, 0) + 1
        return self.fn(obj, *ar, **kw)

    def __get__(self, obj, t=None):
        return self if obj is None else partial(self.__call__, obj)

    def count_of(self, obj):
        return obj.__dict__.get(self.attr, 0)

x, y = S(), S()
x.m(); x.m(); y.m()
S.m.count_of(x), S.m.count_of(y)   #=> (2, 1)   ✅ 独立
```

| 存法 | 优点 | 缺点 |
|---|---|---|
| `obj.__dict__[key]` | 简单，随对象一起被 GC | 污染实例命名空间；`__slots__` 类不可用 |
| `WeakKeyDictionary` | 不污染实例 | 对象必须可弱引用 |
| 改用**描述符** | 最正统 | 代码量大 |

> `functools.lru_cache` 装实例方法会**泄漏实例**是同一个原因：cache 在类变量的字典里，
> 把 `self` 当 key 强引用住了。详见 [[language/decorators]] §4。

#### c. `__get__` 两个分支各自的作用

**删掉整个 `__get__`**：`Count` 实例不是描述符，`s1.m` 走普通属性查找**直接返回实例本身**，
`s1.m()` 等于 `Count.__call__(count_inst)`，零个业务参数 →
`TypeError: m() missing 1 required positional argument: 'self'`。

**只删 `obj is None` 分支**：

```python
S.m()      # ✅ 静默执行！self=None 被传进去，函数体不碰 self 就完全不报错 ❗
S.m.n      # ❌ AttributeError: 'functools.partial' object has no attribute 'n'
```

**(4) 里 `S.m.n` 能取到计数，全靠 `obj is None` 分支返回 `self`。**
所有内置描述符都写这个分支（`property.__get__(None, cls)` 返回 property 对象本身，
所以 `C.attr` 能拿到 `fget`/`fset`）。语义是「**从类上访问，没有实例可绑定**」，
此时应返回描述符自己，让内省和文档工具能拿到元信息。见 [[language/descriptors-properties]]。

#### d. 同步/异步通用的装饰器

```python
def timed(fn):
    if inspect.iscoroutinefunction(fn):          # ← 分支必须在【装饰时】判断
        @functools.wraps(fn)
        async def w(*ar, **kw):
            t0 = time.perf_counter()
            try:
                return await fn(*ar, **kw)
            finally:
                print(f"[async] {fn.__name__} {time.perf_counter()-t0:.3f}s")
        return w

    @functools.wraps(fn)
    def w(*ar, **kw):
        t0 = time.perf_counter()
        try:
            return fn(*ar, **kw)
        finally:
            print(f"[sync] {fn.__name__} {time.perf_counter()-t0:.3f}s")
    return w

asyncio.run(aj())                    #=> [async] aj 0.201s   ✅ 真实耗时
inspect.iscoroutinefunction(aj)      #=> True                ✅ 身份也保住了
```

三个要点：

1. **分支写在装饰时**（`if` 在 `def w` 外面），否则没法让 wrapper 本身是 `async def`。
2. **`await` 写在 `try` 里，打点放 `finally`**——生产计时希望失败也记录耗时，否则超时请求的数据全丢。
3. (6) 变 `True` 是因为**这次 wrapper 自己就是 `async def`**（真有 `CO_COROUTINE` 标志），不是 `wraps` 的功劳。

#### e. 三者的作用范围

**装饰器 = 单个函数调用 / CM = 任意代码块 / 中间件 = 整个 HTTP 事务。**

「请求前后都要处理」不是区分点（装饰器也能做）。**只有中间件能做**的是四类：

1. **请求根本没进到你的函数**：404、路由不匹配、请求体解析失败、鉴权在路由分发前就拒了——
   这些情况下路由函数**从未被调用**，装饰器一行都不会执行。
2. **装饰器测不到框架自身的开销**：

   ```
   中间件计时  |←--------------- 全部 ---------------→|
                 解析请求  依赖注入  [路由函数]  序列化响应
   装饰器计时                       |←--→|
   ```

   FastAPI 返回 Pydantic 模型后，序列化 + 校验 + JSON 编码都在函数返回**之后**。大对象序列化经常是真正的瓶颈。
3. **横切所有路由**，无需逐个贴，也不会漏掉后来新增的。
4. **需要操作原始 request/response**：统一注入响应头、改写 body、读原始 ASGI `scope`。
   装饰器拿到的是**已解析的参数**和**未序列化的返回值**，碰不到 HTTP 层。

> **面试落点**：装饰器的作用域是「**函数调用**」，中间件的作用域是「**HTTP 事务**」。
> 凡是需要覆盖「请求没能到达路由函数」的情况、或需要包含框架自身的解析/序列化开销、
> 或需要操作原始 request/response 的，只能用中间件。

FastAPI 还有**第三选项** `Depends`：范围介于两者之间，能复用、能访问 `Request`、
能按路由/路由组挂载、在路由函数之前执行（可以拒绝请求）。
生产里「鉴权」通常用 `Depends` 而非装饰器或中间件——既能横切又保留 per-route 粒度。见 [[web/fastapi-di]]。

---

## 第 5 题 · 迭代器 / 生成器 / 上下文管理器

### 题目

```python
def gen():
    try:
        yield 1
        yield 2
    finally:
        print("cleanup")

g = gen()
print(next(g))
del g                      # (1)
```

```python
def inner():
    yield 1
    return "done"

def outer():
    r = yield from inner()
    yield r

list(outer())              # (2)
```

```python
def echo():
    while True:
        x = yield
        print("got", x)

g = echo()
g.send("a")                # (3)
```

```python
nums = map(int, ["1", "2", "3"])
print(sum(nums), sum(nums))   # (4)
```

```python
@contextmanager
def cm():
    print("enter")
    try:
        yield "res"
    finally:
        print("exit")

with cm() as r:
    raise ValueError("boom")   # (5)
```

```python
c = cm()
with c: pass
with c: pass               # (6)
```

```python
class Swallow:
    def __enter__(self): return self
    def __exit__(self, *exc): return True

with Swallow():
    raise ValueError("gone")
print("survived")          # (7)
```

**b.** (1) 的机制是什么？`finally` 里有 `yield` 会怎样？
**c.** (3) 为什么必须先 prime？`next(g)` 和 `g.send(None)` 有区别吗？
**d.** (6) 说明什么限制？类实现的 CM 有同样限制吗？
**e.** 标准库里哪个工具**故意**吞异常？`with A(), B():` 里 B 的 `__enter__` 抛异常时 A 会释放吗？为什么需要 `ExitStack`？

### 参考答案

```
(1) 1 然后 cleanup
(2) [1, 'done']
(3) TypeError: can't send non-None value to a just-started generator   ← ★
(4) 6 0
(5) enter → exit → ValueError 继续传播
(6) 第一次 enter/exit 正常；第二次 AttributeError:
    '_GeneratorContextManager' object has no attribute 'args'          ← ★
(7) survived（异常被吞）
```

### 解析

#### (3) send 之前必须 prime

```python
g = echo()
next(g)         # 或 g.send(None)：跑到第一个 yield 处挂起
g.send("a")     #=> got a
```

刚创建的生成器停在函数体**第一行之前**，没有任何 `yield` 表达式在等待接收值——
你 send 的那个 `"a"` 无处可去。CPython 选择显式报错而非静默丢弃。
`send(None)` 被允许，因为 `None` 就是「不传值，只启动」的语义。

> 这正是「基于生成器的协程」时代 `@types.coroutine` 存在的原因之一，也是很多人给生成器写 `@prime` 装饰器的动机：
> ```python
> def prime(fn):
>     @functools.wraps(fn)
>     def w(*a, **k):
>         g = fn(*a, **k); next(g); return g
>     return w
> ```

#### (5) `@contextmanager` **不看返回值**

这是与类实现 CM 最大的差异：

```python
@contextmanager
def bare_return():
    try:
        yield
    except ValueError:
        return          # 没写 True

with bare_return(): raise ValueError    # ✅ 异常【被吞了】

@contextmanager
def returns_false():
    try:
        yield
    except ValueError:
        return False    # 显式 False

with returns_false(): raise ValueError  # ✅ 照样【被吞了】
```

真实机制是 `_GeneratorContextManager.__exit__` 里的 `gen.throw(exc)`——
**把异常抛进生成器挂起的那个 `yield` 处**，然后看生成器怎么反应：

| 生成器的反应 | `__exit__` 返回 | 结果 |
|---|---|---|
| 不 catch（只有 `finally`）→ 异常穿出 | `False` | ✅ 异常继续传播（本题 (5)） |
| `except` 住了，然后正常结束 | `True` | ❗ 异常被吞 |
| `except` 住了，然后**又 `yield`** | — | `RuntimeError: generator didn't stop after throw()` |
| `raise` 一个**新**异常 | — | 新异常传播（旧的成为 `__context__`） |

```
一句话对照：
  类实现的 CM     → 看 __exit__ 的【返回值】
  @contextmanager → 看生成器【有没有 except 住】，return 值无效
```

> **面试落点**：`@contextmanager` 里想吞异常必须写 `except SomeError: pass`；想放行就只写 `finally`。
> **永远不要在 `@contextmanager` 里写 `return True` 指望它吞异常**——不起作用，还误导读代码的人。

#### (6) 一次性的实现细节

`_GeneratorContextManager.__enter__` 的最后一行是：

```python
del self.args, self.kwds, self.func     # 用完就删掉「重建生成器的原料」
```

CPython **故意**删掉这些属性以断开对参数的引用避免内存泄漏，副作用就是第二次
`__enter__` 找不到 `self.args`。所以这个 `AttributeError` 不是缺陷，
是「一次性」契约的实现细节泄漏。（3.9 和 3.13 上都是 `AttributeError`，实测。）

这种「内部实现的 AttributeError」在线上排查时特别费时间，值得记住它对应的真实原因是「**CM 被复用了**」。

#### b. `finally` 在生成器上**不是「总会」执行**

(1) 的真实机制是三步：

```
del g  →  引用计数归零  →  CPython 回收生成器时自动调用 gen.close()
       →  close() 在挂起的 yield 处抛入 GeneratorExit
       →  GeneratorExit 沿着 try 向外传播，触发 finally
```

反例：

```python
def never():
    try:
        yield 1
    finally:
        print("never's finally")

keep = never()
next(keep)
# ← 只要 keep 这个引用还活着，finally 就【永远不执行】
```

这是生成器版的资源泄漏：挂起中的生成器持有文件句柄/数据库连接，只要有人拿着引用，
`finally` 里的 `close()` 就不跑。**PyPy / Jython 没有引用计数**，回收时机不确定，泄漏更严重。
正确做法是显式 `contextlib.closing(gen)` 或 `gen.close()`，不要依赖 GC。

**`finally` 里有 `yield`**：

```python
def bad():
    try:
        yield 1
    finally:
        yield 99        # ← 收到 GeneratorExit 后还想产出值

g = bad(); next(g)
g.close()   # ❌ RuntimeError: generator ignored GeneratorExit
```

`close()` 抛入 `GeneratorExit` 后生成器**必须结束**——要么让它传播，要么 `return`。
反而 `yield` 等于「拒绝关闭」，CPython 判定为错误。

> **面试落点**：`GeneratorExit` 继承自 `BaseException` 而**不是** `Exception`——
> 正是为了防止 `except Exception:` 意外拦住生成器的关闭流程。同族的还有
> `KeyboardInterrupt` / `SystemExit`。这也是「不要写裸 `except:`」的硬理由。

#### c. `next(g)` 与 `g.send(None)` 等价

生成器的 `__next__` 就实现为 `send(None)`。边角差异：

- `next(x)` 是**通用协议**，对任何 iterator 都能用；`send` 只有生成器（和协程）有。
- `next(x, default)` 支持默认值避免 `StopIteration`；`send` 没有这个重载。

#### d. 可重用 ≠ 可重入

**`@contextmanager` 的产物是一次性的** ✅。但「类实现就没限制」是错的——这里有两个必须分开的概念：

| 概念 | 定义 | 反例 |
|---|---|---|
| **可重用（reusable）** | 同一个对象能**先后**用于多个 `with` | 文件对象 ❌ |
| **可重入（reentrant）** | 同一个对象能**嵌套**用于多个 `with` | `threading.Lock` ❌ |

```python
# ❌ 类实现，但【不可重用】
f = open("x")
with f: pass
with f: pass        # ValueError: I/O operation on closed file.

# ✅ 可重用，但 ❌ 不可重入
lk = threading.Lock()
with lk: pass
with lk: pass                        # ✅ 先后使用没问题
with lk:
    lk.acquire(timeout=0.2)          #=> False ← 嵌套自己 = 死锁

# ✅ 两者都可以
with threading.RLock() as rl:
    with rl: ...
```

**类实现给了你「有能力」做到可重用/可重入，但要不要做到取决于 `__enter__`/`__exit__` 怎么写。**
`@contextmanager` 是**结构上就不可能**——状态全在一个一次性的生成器里。

写一个可重入的 CM，要点是**用栈保存每层状态**：

```python
class Indent:
    def __init__(self):
        self._depth = 0                  # ← 计数器就是最简单的「栈」
    def __enter__(self):
        self._depth += 1
        return self
    def __exit__(self, *exc):
        self._depth -= 1
        return False
```

标准库里现成的可重入 CM：`threading.RLock`、`contextlib.suppress`、
`contextlib.redirect_stdout`、`decimal.localcontext`、`contextlib.chdir`（3.11+）。

> **面试落点**：判断一个 CM 能不能复用/嵌套，看它的**状态存在哪里**——
> 存在实例上且用栈管理 → 可重入；存在实例上但只有单份 → 可重用不可重入；
> 存在一次性生成器里（`@contextmanager`）→ 两者都不行。想让 `@contextmanager`
> 的产物反复用，只能**每次重新调用工厂函数**：`with cm(): ...` 而非 `c = cm(); with c: ...`。

#### e-1. 故意吞异常的工具：`contextlib.suppress`

```python
from contextlib import suppress

with suppress(FileNotFoundError):
    open("/nope/nope")          # ✅ 静默跳过

# 等价于
try:
    open("/nope/nope")
except FileNotFoundError:
    pass
```

它的 `__exit__` 就是**故意 `return True`**。同族：`pytest.raises` / `unittest.assertRaises`
（吞掉预期的异常，吞不到反而报失败）。

⚠️ 必须知道的坑：**`suppress` 吞掉异常后，`with` 块内剩下的代码不会执行**：

```python
with suppress(KeyError):
    a = d["missing"]
    print("这行永远不执行")     # ❗
```

#### e-2. `with A(), B():` 里 A 会被正确释放

```python
with A(), B(): ...
# 严格等价于
with A():
    with B():
        ...
```

实测：`A enter` → （B 的 `__enter__` 抛异常）→ `A exit` → 异常传播。
因为 `B()` 的构造和 `__enter__` 都发生在 A 的 `with` 块**内部**，A 的 `try/finally` 覆盖着它。
（3.10+ 支持带括号的多行写法 `with (A(), B()):`，语义相同。）

#### e-3. 那为什么还需要 `ExitStack`

因为嵌套 `with` 要求**编译期就知道有几层**。四个它解决不了的场景：

**1. 数量运行时才确定**

```python
with ExitStack() as st:
    files = [st.enter_context(open(n)) for n in filenames]   # n 个，n 未知
    merge(files)
# ← 全部按【逆序】自动关闭，中途某个 open 失败，之前打开的也会关
```

**2. 条件性获取**

```python
with ExitStack() as st:
    conn = st.enter_context(db.connect())
    if need_lock:
        st.enter_context(lock)          # ← 没有「空 with」这种写法
```

**3. `pop_all()` 转移所有权**——解决「构造期获取多个资源，要么全成功要么全回滚」：

```python
class Multi:
    def __enter__(self):
        with ExitStack() as st:
            self.a = st.enter_context(A())
            self.b = st.enter_context(B())      # ← 若 B 失败，A 在这里就被自动释放
            self._st = st.pop_all()             # ← 全部成功：把清理责任【转移】给 self
        return self                             #    离开这个 with 时不清理

    def __exit__(self, *exc):
        return self._st.__exit__(*exc)          # 延迟到外层 with 结束才清理
```

没有 `pop_all()`，就得手写一长串嵌套 `try/except` 保证「部分失败时回滚已获取的部分」——
这是资源管理里最容易写错的代码。

**4. `callback()` 注册任意清理函数**，不必为它专门写 CM：

```python
with ExitStack() as st:
    st.callback(lambda: print("最后执行"))
    st.callback(os.remove, tmpfile)     # 逆序执行，签名是 callback(fn, *args)
```

> **面试落点**：嵌套 `with` 处理「**编译期已知**的固定几个资源」，`ExitStack` 处理
> 「**运行时才确定**的动态资源集合」。判断标准一句话：**层数写不出来的时候用 `ExitStack`**。
> 异步版是 `contextlib.AsyncExitStack`（配 `async with`）。

---

## 贯穿五题的一条主线：**「不报错的错」**

本组题里有三处考的是同一个模式——**Python 选择「静默地做点别的」而不是「拒绝执行」**：

| 题 | 你以为 | 实际 |
|---|---|---|
| 1-(4) | `copy.copy(t) is t` 是 `False` | `True`——不可变类型的拷贝是 no-op |
| 4-(5) | 同步装饰器包协程会报错 | 不报错，coroutine 透传，计时/异常处理全部作用在错误的时刻 |
| 5-(5) | `@contextmanager` 里 `return False` 能放行 | 返回值被忽略，看的是「有没有 except 住」 |

反向的一处（以为不会报，其实会报）：

| 题 | 你以为 | 实际 |
|---|---|---|
| 5-(3) | 没 prime 也能 `send` | `TypeError: can't send non-None value to a just-started generator` |

**面试里高频考的恰恰是这类题**，因为它区分「用过 Python」和「debug 过 Python 线上问题」。
答题时不要只说结果，要说清「它为什么不报错」——那才是考官想听的。

## 相关

- [[interview/roadmap]] —— 本组题对应 Day 1–4
- [[interview/traps]] —— 单点陷阱速查（22 题）
- [[interview/question-bank-language]] —— 语言核心 28 题
- [[language/objects-mutability]] [[language/scope-closure]] [[language/functions-arguments]]
- [[language/decorators]] [[language/iterators-generators]] [[language/context-managers]]
- [[bridge/js-to-python]] —— 假值集、`%` 符号、`let` 与延迟绑定的 JS 对照
