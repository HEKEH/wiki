---
title: "作用域、闭包与名字绑定（LEGB）"
date: 2026-08-07
tags: [作用域, LEGB, 闭包, nonlocal, global, 延迟绑定, cell]
sources: ["interview-python-cn.md", "cpython-doc/faq-programming.rst"]
---

# 作用域、闭包与名字绑定

## LEGB 查找规则

名字解析按 **L → E → G → B** 顺序：

| 层 | 含义 | 例子 |
|---|---|---|
| **L**ocal | 当前函数内 | 函数里的局部变量、参数 |
| **E**nclosing | 外层**函数**（不是外层类！） | 闭包捕获的变量 |
| **G**lobal | 模块级 | 模块顶层定义的名字 |
| **B**uiltin | 内置命名空间 | `len` `print` `Exception` |

```python
x = "global"

def outer():
    x = "enclosing"
    def inner():
        print(x)          # 找不到 L，向上找到 E
    inner()

outer()          #=> enclosing
```

## 关键差异：Python 没有块级作用域

**只有函数、类、模块、推导式创建作用域**。`if` / `for` / `while` / `with` / `try` **不创建作用域**。

```python
for i in range(3):
    pass
print(i)         #=> 2    循环变量泄漏到函数作用域

if True:
    y = 1
print(y)         #=> 1

with open("f") as fp:
    pass
# fp 在 with 之后依然存在（只是已关闭）
```

> 前端类比：这相当于 JS 里**只有 `var` 没有 `let`**。前端习惯了 `let` 的块级作用域，
> 在 Python 里要格外注意循环变量、临时变量的泄漏和覆盖。

推导式是个例外——Python 3 里推导式**有自己的作用域**：

```python
i = "safe"
[i for i in range(3)]
i                #=> 'safe'   Python 3 中推导式变量不泄漏（Python 2 会泄漏）
```

## 赋值即声明：局部变量的判定在编译期

**只要函数体内有对某名字的赋值，该名字在整个函数内就是局部的**——即使赋值在使用之后。

```python
x = 10
def f():
    print(x)     # ❌ UnboundLocalError: cannot access local variable 'x'
    x = 20       # ← 这一行让 x 在整个 f 内都是局部变量
f()
```

编译期就确定了：CPython 编译函数时扫描所有赋值目标，写进 `co_varnames`，生成 `LOAD_FAST` 指令。

```python
import dis
dis.dis(f)       # 能看到 LOAD_FAST x（局部）而非 LOAD_GLOBAL x
f.__code__.co_varnames   #=> ('x',)
```

"赋值"包括：`x = ...`、`for x in ...`、`import x`、`def x`、`class x`、`with ... as x`、
`except ... as x`、`global` 之外的一切绑定。

### `global` 与 `nonlocal`

```python
counter = 0

def incr():
    global counter        # 声明：写的是模块级的那个
    counter += 1

def make_counter():
    n = 0
    def tick():
        nonlocal n        # 声明：写的是最近一层 enclosing 函数的那个
        n += 1
        return n
    return tick

c = make_counter()
c(); c()         #=> 2
```

- `global` → 绑定到模块级；`nonlocal` → 绑定到最近的 enclosing 函数作用域（**不能是全局**）。
- 只**读**外层变量不需要声明；只有要**重新绑定**才需要。
- `lst.append(x)` 是变更不是绑定，不需要 `nonlocal`；`lst = [...]` 才需要。

## 闭包的实现：cell 对象

闭包捕获的是**变量本身（cell），不是变量的值**。

```python
def make():
    n = 0
    def get(): return n
    def inc():
        nonlocal n
        n += 1
    return get, inc

get, inc = make()
get()            #=> 0
inc()
get()            #=> 1    两个闭包共享同一个 cell

get.__closure__          #=> (<cell at 0x...: int object at 0x...>,)
get.__closure__[0].cell_contents   #=> 1
get.__code__.co_freevars           #=> ('n',)
```

> **面试落点**：闭包 = 函数 + 它引用的自由变量的 cell。CPython 里自由变量存在
> `func.__closure__` 的 cell 对象中，多个闭包引用同一 cell 时共享状态。

## 延迟绑定陷阱（Late Binding）——高频考题

```python
fs = [lambda: i for i in range(3)]
[f() for f in fs]        #=> [2, 2, 2]   ❌ 不是 [0, 1, 2]
```

原因：lambda 捕获的是**变量 `i`**，不是创建时 `i` 的值。循环结束时 `i == 2`，三个 lambda 都读到 2。

三种修法：

```python
# ✅ 1. 默认参数在定义时求值，把当前值"冻结"进去
fs = [lambda i=i: i for i in range(3)]

# ✅ 2. 用工厂函数额外造一层作用域
def make(i): return lambda: i
fs = [make(i) for i in range(3)]

# ✅ 3. functools.partial
from functools import partial
fs = [partial(lambda i: i, i) for i in range(3)]

[f() for f in fs]        #=> [0, 1, 2]
```

> 前端类比：这就是 ES5 时代 `for (var i = 0; ...) setTimeout(() => log(i))` 全打印 3 的同一个问题。
> JS 用 `let` 的每轮新绑定解决了它，**Python 没有 `let`，所以这个坑至今存在**。
> 这个对照在面试里说出来很加分。

同样的坑在 `for` 循环 + 注册回调、`for` 循环 + 创建 partial 的场景反复出现。

## 类作用域的特殊性（进阶考点）

**类体不是 enclosing 作用域**——方法内部看不到类体里的名字。

```python
class A:
    x = 1
    def get(self):
        return x        # ❌ NameError！类作用域不参与 LEGB 的 E
    def get2(self):
        return self.x   # ✅ 或 A.x
```

更刁钻的：类体内的推导式也看不到类变量（除了最外层的可迭代对象）。

```python
class B:
    vals = [1, 2, 3]
    ok    = [v * 2 for v in vals]                    # ✅ 只有最外层 iterable 在类作用域求值
    bad   = [v * n for v in vals for n in vals]      # ❌ NameError: vals（第二个 for 在推导式内部求值）
    worse = [x for x in range(3) if x < len(vals)]   # ❌ NameError: vals（条件在推导式内部求值）
```

原因：推导式被编译成一个隐式函数，该函数的 enclosing 作用域跳过了类作用域。

> **面试落点**：「类作用域不参与闭包」——因为类体执行完就变成 `type()` 的 namespace 字典，
> 不产生 cell。

## 快速自查表

```python
def f():
    print(locals())     # 当前局部命名空间（字典快照）
print(globals())        # 模块全局命名空间（真字典，改它有效）
import builtins; dir(builtins)

f.__code__.co_varnames  # 局部变量名
f.__code__.co_freevars  # 自由变量名（来自 enclosing）
f.__code__.co_cellvars  # 被内层函数捕获的本地变量名
f.__closure__           # cell 元组
```

## 相关

- [[language/functions-arguments]] —— 默认参数在定义时求值，正是修 late binding 的原理
- [[language/decorators]] —— 装饰器本质就是闭包
- [[language/comprehensions-functional]] —— 推导式的隐式函数作用域
- [[internals/bytecode-execution]] —— LOAD_FAST / LOAD_GLOBAL / LOAD_DEREF 的区别
- [[bridge/js-to-python]] —— var/let vs Python 无块级作用域
- [[interview/question-bank-language]] —— 相关面试题
