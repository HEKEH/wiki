---
title: "函数与参数机制"
date: 2026-08-07
tags: [函数, 参数, args, kwargs, 一等函数, 默认参数, 位置参数]
sources: ["python-cheatsheet.md", "interview-python-cn.md", "cpython-doc/faq-programming.rst"]
---

# 函数与参数机制

Python 的参数系统比 JS 丰富得多——JS 只有位置参数 + 默认值 + rest，Python 有
**位置-only / 位置或关键字 / 可变位置 / 关键字-only / 可变关键字** 五类。面试常考完整签名的解析顺序。

## 完整参数签名

```python
def f(pos_only, /, pos_or_kw, *args, kw_only, kw_with_default=1, **kwargs):
    ...
```

| 位置 | 类型 | 调用方能怎么传 |
|---|---|---|
| `/` 之前 | positional-only（3.8+） | 只能按位置 |
| `/` 与 `*` 之间 | positional-or-keyword | 位置或关键字都行 |
| `*args` | var-positional | 收集多余的位置参数成 tuple |
| `*` 或 `*args` 之后 | keyword-only | **只能**按关键字 |
| `**kwargs` | var-keyword | 收集多余的关键字参数成 dict |

```python
def api(url, /, method="GET", *headers, timeout, **opts):
    print(url, method, headers, timeout, opts)

api("/a", "POST", "H1", "H2", timeout=3, retry=2)
#=> /a POST ('H1', 'H2') 3 {'retry': 2}

api(url="/a", timeout=1)      # ❌ TypeError: api() got some positional-only arguments
                              #    passed as keyword arguments: 'url'
api("/a")                     # ❌ TypeError: missing 1 required keyword-only argument: 'timeout'
```

**不带名字的 `*`** 是"从这里开始都是关键字-only"的分隔符，非常常用：

```python
def connect(host, port, *, timeout=10, retries=3):   # timeout/retries 必须写名字
    ...
connect("db", 5432, timeout=5)     # ✅
connect("db", 5432, 5)             # ❌ takes 2 positional arguments but 3 were given
```

> **工程价值**：给布尔开关和配置项加 `*`，可以杜绝 `create(user, True, False, True)` 这种
> 读不懂的调用——这是 Python 代码评审里的常见要求。

## 解包（unpacking）

```python
args = (1, 2)
kw = {"c": 3}
f(*args, **kw)                    # 调用时展开

# 收集
a, *rest = [1, 2, 3]              #=> a=1, rest=[2, 3]
first, *mid, last = range(5)      #=> 0, [1,2,3], 4

# 合并（3.5+）
merged = {**base, **override}     # 类似 JS 的 {...a, ...b}
merged = base | override          # 3.9+，dict 的 | 运算符
combined = [*a, *b]
```

> 前端类比：`*` ≈ JS 数组展开 `...`，`**` ≈ 对象展开 `...`。
> 差别：Python 的 `*rest` 可以在**中间**（`first, *mid, last`），JS 的 rest 只能在末尾。

## 默认参数：定义时求值一次

```python
def f(x=[]):        # ❌ 可变默认值
    x.append(1); return x
f(); f()            #=> [1, 1]

def g(t=time.time()):   # ❌ 时间被冻结在导入时刻
    ...
```

正确写法用 `None` 哨兵。若 `None` 本身是合法取值，用自定义哨兵：

```python
_MISSING = object()

def get(key, default=_MISSING):
    if default is _MISSING:
        raise KeyError(key)      # 区分"没传 default"和"default=None"
    return default
```

默认值存储在函数对象上，可以被检视甚至修改：

```python
f.__defaults__           #=> ([1],)         位置参数默认值
f.__kwdefaults__         #=> {'timeout': 10} 关键字-only 默认值
```

## 函数是一等对象

```python
def greet(name): return f"hi {name}"

greet.lang = "en"            # 函数可以挂属性（装饰器常用）
greet.__name__               #=> 'greet'
greet.__doc__                # docstring
greet.__annotations__        #=> {'name': ..., 'return': ...}
greet.__module__
greet.__code__.co_argcount   #=> 1

funcs = {"greet": greet}     # 可存进容器
list(map(greet, ["a", "b"])) # 可传递
```

内省签名（写框架/装饰器必备，FastAPI 的依赖注入就靠它）：

```python
import inspect

def handler(user_id: int, q: str = "x") -> dict: ...

sig = inspect.signature(handler)
sig.parameters
#=> OrderedDict([('user_id', <Parameter "user_id: int">),
#                ('q', <Parameter "q: str = 'x'">)])
sig.parameters["q"].default          #=> 'x'
sig.parameters["user_id"].annotation #=> <class 'int'>
sig.return_annotation                #=> <class 'dict'>

bound = sig.bind(1, q="y")           # 模拟一次调用的参数绑定
bound.arguments                      #=> {'user_id': 1, 'q': 'y'}
```

> **面试落点**：被问「FastAPI 怎么知道路由函数要什么参数」时，答案就是
> `inspect.signature` + 类型注解。见 [[web/fastapi-di]]。

## lambda 与函数式工具

```python
sorted(data, key=lambda x: (x.dept, -x.salary))    # 多级排序，负号实现降序
max(words, key=len)
list(filter(None, [0, 1, "", "a"]))                #=> [1, 'a']  过滤假值
```

`lambda` 只能是**单个表达式**，不能有语句（没有 `return`、`if x: ...`、赋值）。
需要多行就写 `def`——Python 社区明确偏好具名函数。

```python
# ❌ PEP 8 反对：给 lambda 起名字
f = lambda x: x * 2
# ✅
def f(x): return x * 2
```

`operator` 模块比 lambda 更快也更清晰：

```python
from operator import itemgetter, attrgetter, methodcaller
sorted(rows, key=itemgetter(1, 2))
sorted(objs, key=attrgetter("created_at"))
list(map(methodcaller("strip"), lines))
```

## 参数传递语义回顾

见 [[language/objects-mutability]]：Python 是 **call by sharing**。
函数内 `lst.append(x)` 外部可见，`lst = [...]` 外部不可见。

## 常见签名模式

```python
# 1. 转发所有参数（装饰器/包装器）
def wrapper(*args, **kwargs):
    return fn(*args, **kwargs)

# 2. 可选覆盖配置
def run(cfg=None, **overrides):
    cfg = {**DEFAULTS, **(cfg or {}), **overrides}

# 3. 强制关键字 + 类型注解 + 默认（现代 Python 推荐签名）
def fetch(url: str, *, timeout: float = 5.0, retries: int = 3) -> bytes: ...

# 4. 重载（Python 无真正重载，用 singledispatch 或 typing.overload）
from functools import singledispatch

@singledispatch
def render(x): return str(x)

@render.register
def _(x: list): return ", ".join(map(str, x))

@render.register
def _(x: dict): return "; ".join(f"{k}={v}" for k, v in x.items())

render([1, 2])       #=> '1, 2'
render({"a": 1})     #=> 'a=1'
```

> **面试落点**：「Python 支持函数重载吗？」——语言层**不支持**，后定义的同名函数直接覆盖前者。
> 需要重载效果就用默认参数、`*args`、`functools.singledispatch`（按第一个参数类型分派）
> 或 `typing.overload`（只影响静态类型检查，运行时仍需单一实现）。

## 相关

- [[language/scope-closure]] —— 默认参数为什么能修 late binding
- [[language/decorators]] —— `*args, **kwargs` 转发的标准场景
- [[language/typing]] —— 注解与 `typing.overload`
- [[language/comprehensions-functional]] —— functools / operator 工具箱
- [[web/fastapi-di]] —— `inspect.signature` 在依赖注入中的应用
- [[interview/question-bank-language]] —— 相关面试题
