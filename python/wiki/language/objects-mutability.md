---
title: "对象、引用、可变性与拷贝"
date: 2026-08-07
tags: [可变性, 引用, 浅拷贝, 深拷贝, 参数传递, is, 驻留]
sources: ["interview-python-cn.md", "cpython-doc/faq-programming.rst", "wtfpython.md"]
---

# 对象、引用、可变性与拷贝

这是 Python 面试的**第一道分水岭**。前端工程师对 JS 的「原始类型按值、对象按引用」很熟，
但 Python 的模型不一样——**Python 里没有"值类型"，一切皆对象，变量永远是名字绑定（name binding）**。

## 1. 变量是标签，不是盒子

```python
a = [1, 2, 3]
b = a            # 不是拷贝！b 和 a 是贴在同一个列表上的两张标签
b.append(4)
a                #=> [1, 2, 3, 4]
a is b           #=> True
```

每个对象有三样东西：**identity（`id()`，即内存地址）、type（`type()`，不可变）、value**。
变量只持有 identity 的引用。

```python
x = 256
y = 256
x is y           #=> True   小整数缓存
x = 257
y = 257
x is y           #=> False（在 REPL 中）/ True（同一编译单元内常量折叠）
```

> ⚠️ **永远不要用 `is` 比较值**。`is` 比较 identity，`==` 比较值（走 `__eq__`）。
> `is` 的正当用途只有三处：`x is None`、`x is True/False`、哨兵对象 `x is _SENTINEL`。

## 2. 可变 vs 不可变

| 不可变（immutable） | 可变（mutable） |
|---|---|
| `int` `float` `bool` `complex` | `list` `dict` `set` `bytearray` |
| `str` `bytes` `tuple` `frozenset` | 绝大多数自定义类实例 |
| `range` `NoneType` | `collections` 里的容器 |

**不可变 ≠ 内容不可变**：`tuple` 本身不可变（不能改变引用了哪些对象），但它引用的对象可以是可变的。

```python
t = ([1, 2], "x")
t[0].append(3)
t                #=> ([1, 2, 3], 'x')   合法！tuple 的"元素引用"没变
t[0] = []        # ❌ TypeError: 'tuple' object does not support item assignment
hash(t)          # ❌ TypeError: unhashable type: 'list'  ← tuple 含可变元素就不可哈希
```

> **面试落点**：「tuple 是不可变的吗？」正确答案是「tuple 的**结构**不可变，但如果元素本身可变，
> 元素的内容可以改；因此含可变元素的 tuple 不可哈希」。

## 3. 参数传递：既不是传值也不是传引用

Python 的传参机制叫 **call by object reference / call by sharing（传对象引用）**：
函数接收的是实参对象的引用副本。

```python
def f(lst, num):
    lst.append(1)     # 就地修改 → 调用方可见
    lst = [99]        # 重新绑定局部名字 → 调用方不可见
    num += 1          # int 不可变，等价于重新绑定 → 调用方不可见

data, n = [], 0
f(data, n)
data, n          #=> ([1], 0)
```

心智模型：**"能不能改到外面"取决于你做的是「就地变更（mutation）」还是「重新绑定（rebinding）」**，
而不是取决于类型。这跟 JS 完全一致（JS 也是 call by sharing），区别只在 Python 的
`int`/`str`/`tuple` 是对象但不可变。

### 默认参数陷阱（几乎必考）

```python
def add(item, target=[]):      # ❌ 默认值在【函数定义时】求值一次，此后共享
    target.append(item)
    return target

add(1)           #=> [1]
add(2)           #=> [1, 2]   ← 惊喜
```

正确写法：

```python
def add(item, target=None):    # ✅
    if target is None:
        target = []
    target.append(item)
    return target
```

原理：默认值存在 `func.__defaults__` 元组里，是函数对象的属性，定义时求值一次。

```python
add.__defaults__     #=> (None,)
```

同样的坑存在于 `datetime.now()` 作默认值（会被冻结在导入时刻）。
dataclass 用 `field(default_factory=list)` 解决这个问题，见 [[language/dataclasses-models]]。

## 4. 拷贝：赋值 / 浅拷贝 / 深拷贝

```python
import copy

orig = [[1, 2], [3, 4]]

alias   = orig                 # 别名：完全同一个对象
shallow = orig[:]              # 浅拷贝，等价于 list(orig) / copy.copy(orig)
deep    = copy.deepcopy(orig)  # 深拷贝：递归复制

orig[0].append(99)
alias            #=> [[1, 2, 99], [3, 4]]   跟着变
shallow          #=> [[1, 2, 99], [3, 4]]   跟着变！内层还是同一个对象
deep             #=> [[1, 2], [3, 4]]       独立
```

各类型的浅拷贝写法：

```python
list(x)      x[:]      x.copy()      # list
dict(x)      x.copy()  {**x}         # dict
set(x)       x.copy()                # set
copy.copy(x)                         # 通用，走 __copy__
```

`deepcopy` 的两个要点：

- **能正确处理循环引用**（内部用 memo 字典记录已复制对象的 id）。
- **慢**，且会连数据库连接、socket 一起复制（用 `__deepcopy__` 或 `copy.deepcopy(x, memo)` 定制）。

```python
a = [1]
a.append(a)             # 自引用
b = copy.deepcopy(a)    # 不会栈溢出
b[1] is b               #=> True   结构被正确保留
```

> 前端类比：`{...obj}` / `Object.assign` ≈ 浅拷贝；`structuredClone(obj)` ≈ `copy.deepcopy`
> ——**共同点是都能正确处理循环引用**。
>
> ⚠️ 但对函数的处理**完全不同**：`structuredClone` 遇到函数会**抛 `DataCloneError`**，
> 而 `deepcopy` 把函数当作原子对象**原样返回同一个引用**（`copy.deepcopy(fn) is fn` → `True`）。
> 模块、类、socket、数据库连接同理——deepcopy 不会真的复制它们，但也不会报错。

## 5. 驻留（interning）与 `is` 的迷惑行为

CPython 为了性能会缓存一些不可变对象：

```python
# 小整数缓存：-5 ~ 256 在解释器启动时预先创建
a, b = 100, 100
a is b           #=> True
a, b = 1000, 1000
a is b           #=> True  ← 在同一行/同一代码对象里，编译期常量折叠使其共享

# 但跨语句就不一定：
a = 1000
b = 1000
a is b           #=> False（CPython 交互式）

# 字符串驻留：编译期可确定的"标识符样"字符串被驻留
s1 = "hello"
s2 = "hello"
s1 is s2         #=> True
s3 = "hello world"
s4 = "hello world"
s3 is s4         #=> True（同一编译单元内被折叠）
s5 = "".join(["hel", "lo"])
s5 is s1         #=> False ← 运行期构造的不驻留
import sys
sys.intern(s5) is s1   #=> True  手动驻留
```

> **面试落点**：这些行为都是 **CPython 实现细节，不是语言规范**，PyPy/其它实现可能不同。
> 能主动说出"这是实现细节，业务代码绝不能依赖"是加分项。

## 6. 等价性判断的完整决策表

| 想表达 | 写法 |
|---|---|
| 是不是同一个对象 | `a is b` |
| 值是否相等 | `a == b` |
| 是不是 None | `a is None`（不要写 `a == None`） |
| 类型是否精确匹配 | `type(a) is B` |
| 是否是某类（含子类） | `isinstance(a, B)` |
| 是否符合某个协议 | `isinstance(a, MyProtocol)`（需 `@runtime_checkable`） |
| 真值判断 | `if a:`（走 `__bool__` → `__len__`） |

真值陷阱（前端工程师尤其容易踩）：

```python
# JS: if (arr.length) / if (x !== undefined)
# Python 的假值有：False 0 0.0 0j '' [] {} () set() None range(0)
def f(x=None):
    if not x:        # ❌ x=0 或 x=[] 会被当成"没传"
        x = default
    if x is None:    # ✅ 明确区分"没传"和"传了空值"
        x = default
```

## 7. 与 JS 的关键差异清单

| | JavaScript | Python |
|---|---|---|
| 相等 | `==` 有类型转换，`===` 严格 | 只有 `==`（走 `__eq__`）和 `is`（身份） |
| 数字 | 全是 double（+ BigInt） | `int` **任意精度**，`float` 是 double |
| 字符串 | 不可变，UTF-16 码元 | 不可变，Unicode 码点 |
| 拷贝 | 展开运算符 / structuredClone | 切片 / `copy` / `deepcopy` |
| 空值 | `null` 和 `undefined` 两个 | 只有 `None` |
| 真值 | `""` `0` `null` `undefined` `NaN` 为假；`[]` `{}` 为**真** | `[]` `{}` `""` `0` `None` 均为**假** |
| 整数除法 | `/` 得浮点，`Math.floor` | `/` 得 float，`//` 是**向下取整**除法 |

`[]` 和 `{}` 的真值差异是前端转 Python 最容易踩的坑之一（JS 里空数组是真值！）。

```python
if []:           # 不进入
    ...
# JS: if ([]) { ... }   // 会进入
```

`//` 的向下取整（floor）而非截断（truncate），负数上与 C/JS 不同：

```python
-7 // 2          #=> -4    （向下取整）
-7 % 2           #=> 1     （符号跟随除数）
int(-7 / 2)      #=> -3    （截断）
import math; math.trunc(-7 / 2)   #=> -3
```

> **面试落点**：`-7 // 2 == -4` 且 `-7 % 2 == 1`，因为 Python 保证
> `a == (a // b) * b + (a % b)` 且 `a % b` 的符号与 `b` 一致。

## 相关

- [[language/data-model]] —— `__eq__` / `__hash__` / `__bool__` / `__copy__` 协议
- [[language/functions-arguments]] —— 默认参数求值时机的完整机制
- [[internals/cpython-object-model]] —— PyObject、引用计数、驻留的实现层
- [[internals/garbage-collection]] —— 循环引用与 deepcopy 的 memo
- [[interview/traps]] —— 可变默认值、tuple += list 等经典陷阱
- [[bridge/js-to-python]] —— 完整的 JS↔Python 对照表
- [[sources/interview-python-cn]] —— 来源：中文经典面试题库
