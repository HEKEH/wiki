---
title: "描述符、property 与 __slots__"
date: 2026-08-07
tags: [描述符, descriptor, property, slots, 属性查找, cached_property]
sources: ["cpython-doc/descriptor-howto.rst", "cpython-doc/datamodel.rst"]
---

# 描述符、property 与 `__slots__`

描述符是 Python **最被低估的核心机制**：方法绑定、`@property`、`@classmethod`、`@staticmethod`、
`__slots__`、Django/SQLAlchemy 的 ORM 字段，全部建立在它之上。面试里能讲清描述符 = 直接进阶。

## 1. 描述符协议

一个类只要实现了下列任一方法，它的实例作为**类属性**时就是描述符：

```python
class Descriptor:
    def __set_name__(self, owner, name):   # 3.6+，类创建时自动调用，告诉你被赋给了哪个名字
        self.name = name
    def __get__(self, obj, objtype=None):  # obj 为 None 表示通过类访问
        ...
    def __set__(self, obj, value):
        ...
    def __delete__(self, obj):
        ...
```

**两种描述符（决定优先级，必考）**：

| 类型 | 定义 | 优先级 |
|---|---|---|
| **data descriptor（数据描述符）** | 定义了 `__set__` 或 `__delete__` | **高于**实例 `__dict__` |
| **non-data descriptor（非数据描述符）** | 只定义了 `__get__` | **低于**实例 `__dict__` |

## 2. 属性查找的完整顺序（核心考点）

`obj.x` 的解析（由 `object.__getattribute__` 实现）：

```text
1. 沿 type(obj).__mro__ 查找 'x'
2. 若找到且是 data descriptor        → 返回 descr.__get__(obj, type(obj))   ★最高优先级
3. 若 'x' 在 obj.__dict__ 中          → 返回 obj.__dict__['x']
4. 若类中找到且是 non-data descriptor → 返回 descr.__get__(obj, type(obj))
5. 若类中找到普通类属性               → 直接返回
6. 否则调用 type(obj).__getattr__(obj, 'x')，还没有就抛 AttributeError
```

验证：

```python
class NonData:
    def __get__(self, obj, t=None): return "from descriptor"

class Data(NonData):
    def __set__(self, obj, v): obj.__dict__["d"] = v

class C:
    n = NonData()
    d = Data()

c = C()
c.n                      #=> 'from descriptor'
c.__dict__["n"] = "instance"
c.n                      #=> 'instance'        ← 非数据描述符被实例字典遮蔽

c.d = "instance"         # 走 Data.__set__
c.__dict__["d"]          #=> 'instance'
c.d                      #=> 'from descriptor'  ← 数据描述符永远优先，实例字典遮不住
```

> **面试落点**：「实例属性和类属性谁优先？」——**取决于类属性是不是数据描述符**。
> 数据描述符 > 实例字典 > 非数据描述符 > 普通类属性。能答出这一句就赢了 90% 的人。

## 3. 方法为什么能自动绑定

函数是**非数据描述符**：

```python
def f(self): return self
f.__get__                        #=> <method-wrapper '__get__' of function>

class C: m = f
c = C()
c.m                              #=> <bound method f of <C object>>
C.__dict__["m"].__get__(c, C)    #=> 同上，这就是 c.m 背后发生的事
```

因为函数是**非**数据描述符，所以你可以用实例属性遮蔽一个方法：

```python
c.m = lambda: "shadowed"
c.m()                            #=> 'shadowed'
```

`classmethod` / `staticmethod` 也是描述符：

```python
class MyStaticMethod:
    def __init__(self, f): self.f = f
    def __get__(self, obj, objtype=None): return self.f        # 不绑定，原样返回

class MyClassMethod:
    def __init__(self, f): self.f = f
    def __get__(self, obj, objtype=None):
        return MethodType(self.f, objtype or type(obj))        # 绑定到"类"
```

## 4. `property`：数据描述符的语法糖

```python
class Circle:
    def __init__(self, r): self._r = r

    @property
    def radius(self):                       # getter
        return self._r

    @radius.setter
    def radius(self, v):                    # setter：加校验
        if v < 0:
            raise ValueError("radius must be >= 0")
        self._r = v

    @radius.deleter
    def radius(self):
        del self._r

    @property
    def area(self):                         # 只读计算属性
        return 3.14159 * self._r ** 2

c = Circle(2)
c.radius = 5        # 走 setter
c.area              #=> 78.5   不能赋值：AttributeError
```

等价的手工写法：

```python
class Circle:
    radius = property(fget=get_r, fset=set_r, fdel=del_r, doc="半径")
```

`property` 是**数据描述符**（定义了 `__set__`），所以即使实例字典里有同名键也遮不住它。

> **实战价值**：`property` 让你可以**在不改变调用方代码的前提下**，把一个公开属性升级成带校验/
> 惰性计算/日志的逻辑。这就是 Python 不需要"给所有字段写 getter/setter"的原因——
> **需要时再加，不破坏 API**。（Java/C# 程序员的 getter/setter 习惯在 Python 里是反模式。）
>
> 前端类比：≈ JS 的 `get x() {} / set x(v) {}` 和 `Object.defineProperty`。

## 5. `cached_property`

```python
from functools import cached_property

class Report:
    @cached_property
    def data(self):
        print("computing...")
        return heavy_query()

r = Report()
r.data          # computing... 只打印一次
r.data          # 直接返回缓存
r.__dict__["data"]   # 结果就存在实例字典里
del r.data      # 清除缓存
```

原理：`cached_property` 是**非数据描述符**——第一次访问后把结果写进 `obj.__dict__`，
之后按查找顺序第 3 步就命中实例字典，再也不走描述符。**这正是"非数据描述符 < 实例字典"的经典应用。**

限制：

- 要求实例有 `__dict__`（**和 `__slots__` 不兼容**）。
- 不是线程安全的强保证（3.12 起移除了内部锁，可能重复计算，但不会出错）。

## 6. 自定义描述符：可复用的字段校验

```python
class Typed:
    def __init__(self, type_, default=None):
        self.type_, self.default = type_, default

    def __set_name__(self, owner, name):
        self.private = f"_{name}"          # 3.6+ 自动获知字段名

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self                     # 通过类访问时返回描述符自身（惯例）
        return getattr(obj, self.private, self.default)

    def __set__(self, obj, value):
        if not isinstance(value, self.type_):
            raise TypeError(f"{self.private[1:]} must be {self.type_.__name__}")
        setattr(obj, self.private, value)

class User:
    name = Typed(str)
    age  = Typed(int, 0)

u = User()
u.name = "bob"
u.age = "x"        # ❌ TypeError: age must be int
```

这就是 **Django Model 字段、SQLAlchemy Column、Pydantic v1 字段** 的实现骨架。

## 7. `__slots__`：省内存 + 禁止动态属性

```python
class Point:
    __slots__ = ("x", "y")            # 不再创建 __dict__
    def __init__(self, x, y):
        self.x, self.y = x, y

p = Point(1, 2)
p.z = 3           # ❌ AttributeError: 'Point' object has no attribute 'z'
p.__dict__        # ❌ AttributeError
```

原理：`__slots__` 里的每个名字被编译成一个 **`member_descriptor`（数据描述符）**，
值存在对象的固定偏移槽位上，而不是哈希表里。

```python
Point.x           #=> <member 'x' of 'Point' objects>
type(Point.x)     #=> <class 'member_descriptor'>
```

内存对比（百万实例场景很显著）：

```python
import sys
class A:            pass
class B: __slots__ = ("x", "y")

a = A(); a.x = a.y = 1
b = B(); b.x = b.y = 1
sys.getsizeof(a)                    #=> 48    实例对象本体
sys.getsizeof(a.__dict__)           #=> 296   ← 真正的开销在这里
sys.getsizeof(b)                    #=> 48    没有 __dict__，值直接存在对象的槽位里
# CPython 3.13 / 64 位实测：344 vs 48，约 7 倍差距。
# 百万级实例时这就是 130 MB vs 46 MB（见 internals/memory-model 的完整对比表）
```

注意事项：

- 子类**不写** `__slots__` 会重新获得 `__dict__`，省内存效果消失。
- 与 `cached_property`、`functools.lru_cache` 装饰的实例方法、weakref 有冲突
  （需要 `__weakref__` 就写进 `__slots__`）。
- 多重继承时多个父类都有非空 `__slots__` 会报错。
- `dataclass(slots=True)`（3.10+）可自动生成。

> **面试落点**：`__slots__` 的收益不只是省内存，还有**更快的属性访问**（数组偏移 vs 哈希查找）
> 和**防止拼写错误式的属性污染**。代价是失去动态性。适用于**大量小对象**的场景（如解析出的记录）。

## 8. 一张图串起来

```text
obj.attr
   │
   ├─ type(obj).__mro__ 中找到 attr？
   │     ├─ 是 data descriptor（有 __set__/__delete__）→ descr.__get__()   ← property、slots、Model 字段
   │     └─ 否 ↓
   ├─ obj.__dict__ 里有？ → 直接返回                                        ← cached_property 命中处
   ├─ 类里那个是 non-data descriptor（只有 __get__）→ descr.__get__()      ← 普通方法、classmethod
   ├─ 类里那个是普通值 → 返回                                              ← 类变量
   └─ __getattr__ 兜底 → 否则 AttributeError
```

## 相关

- [[language/classes-mro]] —— 属性沿 MRO 查找
- [[language/data-model]] —— `__getattr__` / `__getattribute__`
- [[language/decorators]] —— property/classmethod 作为装饰器的用法
- [[language/dataclasses-models]] —— `dataclass(slots=True)`
- [[internals/memory-model]] —— `__dict__` vs slots 的内存布局
- [[web/pydantic]] —— Pydantic 字段与描述符思想的关系
- [[sources/cpython-docs]] —— 来源：官方 descriptor HOWTO
