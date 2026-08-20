---
title: "类、继承与 MRO"
date: 2026-08-07
tags: [类, 继承, MRO, C3, super, 多重继承, mixin, 抽象基类]
sources: ["interview-python-cn.md", "cpython-doc/datamodel.rst"]
---

# 类、继承与 MRO

## 1. 类的基本构成

```python
class Account:
    bank = "ABC"                       # 类变量：所有实例共享，存在 Account.__dict__

    def __init__(self, owner, balance=0):
        self.owner = owner             # 实例变量：存在 self.__dict__
        self._balance = balance        # 单下划线：约定的"内部"，无强制
        self.__secret = "x"            # 双下划线：名字改写（name mangling）

    def deposit(self, amt):            # 实例方法：第一个参数是实例
        self._balance += amt

    @classmethod
    def from_dict(cls, d):             # 类方法：第一个参数是类，支持子类多态构造
        return cls(**d)

    @staticmethod
    def validate(amt):                 # 静态方法：不接收 self/cls
        return amt > 0
```

### 类变量 vs 实例变量（经典陷阱）

```python
class C:
    items = []                 # ❌ 所有实例共享同一个 list！
    count = 0

a, b = C(), C()
a.items.append(1)
b.items                        #=> [1]   共享

a.count += 1                   # 这是 a.count = a.count + 1 → 创建了实例变量
a.count, b.count, C.count      #=> (1, 0, 0)
a.__dict__                     #=> {'count': 1}
```

规则：**读属性时先查实例 `__dict__`，再查类（沿 MRO）；写属性默认写进实例 `__dict__`。**
可变类变量的就地修改会影响所有实例，不可变类变量的赋值只会遮蔽（shadow）。

```python
# ✅ 每实例独立
class C:
    def __init__(self):
        self.items = []
```

更隐蔽的一层：**子类也不隔离**。子类没有自己的表，它往父类的表里写：

```python
class Base:
    registry = []
    def __init__(self, name):
        self.registry.append(name)      # ❌ 读到 Base.registry，就地改

class Sub(Base): pass

Base("p"); Sub("q")
Base.registry                    #=> ['p', 'q']   ← Sub 的记录混进来了
Sub.registry is Base.registry    #=> True
'registry' in Sub.__dict__       #=> False        ← Sub 压根没有自己的表
```

真想要「每个子类一张独立注册表」，用 `__init_subclass__`：

```python
class Base:
    registry = []
    def __init_subclass__(cls, **kw):
        super().__init_subclass__(**kw)
        cls.registry = []               # ✅ 建子类时各发一张
```

> **面试落点**：可变类变量的三层代价——① 实例间串味；② 子类共写同一张表；
> ③ 这张表永不释放，长期运行的进程里就是内存泄漏。类变量存可变对象**只有**
> 在「语义上属于类而非实例」时才正确（插件注册表、全局缓存），而且要显式写
> `Base.registry.append(...)` 而不是 `self.registry.append(...)`，否则读者分不清你是有意共享还是踩坑。
> 只读的表应该用 `tuple` / `frozenset` / `MappingProxyType` 把可变性掐掉。

### 下划线约定

| 写法 | 含义 | 强制性 |
|---|---|---|
| `name` | 公开 | — |
| `_name` | 内部实现，"别碰" | 纯约定；`from m import *` 不导出 |
| `__name` | **名字改写**为 `_ClassName__name` | 编译器强制，用于避免子类意外覆盖 |
| `__name__` | dunder，语言保留 | 别自己发明 |

```python
class A:
    def __init__(self): self.__x = 1
a = A()
a.__x                #=> ❌ AttributeError
a._A__x              #=> 1     name mangling 只是改名，不是私有
a.__dict__           #=> {'_A__x': 1}
```

> **面试落点**：「Python 有私有变量吗？」——**没有真正的私有**。单下划线是约定，
> 双下划线是**名字改写**，设计目的是**防止多重继承时的命名冲突**，不是访问控制。

## 2. `self` 与绑定方法

```python
class C:
    def m(self): return self

c = C()
C.m                  #=> <function C.m>           普通函数
c.m                  #=> <bound method C.m of ...> 绑定方法
c.m()  is  C.m(c)    #=> True                     完全等价
c.m.__self__ is c    #=> True
c.m.__func__ is C.m  #=> True
c.m is c.m           #=> False    ← 绑定方法每次访问现造，不是缓存的
```

对比三种方法的访问结果（都是描述符在干活，但 `__get__` 返回的东西不同）：

```python
class P:
    def im(self): ...
    @classmethod
    def cm(cls): ...
    @staticmethod
    def sm(): ...

class Sub(P): pass
p = P()

Sub.sm is P.sm                #=> True    staticmethod：原样返回底层函数，不造新对象
Sub.cm.__self__ is Sub        #=> True    classmethod：绑定的是**类**，而且是实际访问的那个类
p.im.__func__ is P.im         #=> True    实例方法：绑定实例，__func__ 才是原函数
```

`Sub.cm.__self__ is Sub`（而不是 `P`）就是 `classmethod` 能多态构造的根源：
`cls` 拿到的是**实际调用的类**。

它们共享一条规则：`Sub.x` 沿 MRO 找到属性后，若它是描述符就调 `__get__`。

`c.m` 之所以能自动传入 `self`，是因为**函数是描述符**（`function.__get__` 返回绑定方法）。
见 [[language/descriptors-properties]]。

> **面试落点**：「为什么方法要显式写 self？」——因为方法就是类命名空间里的普通函数，
> 通过描述符协议在访问时绑定实例。显式 `self` 让"函数"和"方法"是同一种东西（Explicit is better
> than implicit）。这也是为什么可以给类动态挂函数当方法。

## 3. `@classmethod` vs `@staticmethod`

```python
class Date:
    def __init__(self, y, m, d): self.y, self.m, self.d = y, m, d

    @classmethod
    def today(cls):                    # cls 是实际调用的类 → 子类调用返回子类实例
        t = time.localtime()
        return cls(t.tm_year, t.tm_mon, t.tm_mday)

    @staticmethod
    def is_leap(y):                    # 与类相关但不需要类/实例状态
        return y % 4 == 0 and (y % 100 != 0 or y % 400 == 0)

class USDate(Date): pass
type(USDate.today())     #=> USDate    ← classmethod 的多态构造能力
```

| | 接收 | 典型用途 |
|---|---|---|
| 实例方法 | `self` | 操作实例状态 |
| `classmethod` | `cls` | **替代构造器**（`from_json`/`from_orm`）、操作类状态、工厂 |
| `staticmethod` | 无 | 逻辑归属该类但不用状态的工具函数（组织代码用） |

## 4. 继承与 `super()`

```python
class Base:
    def __init__(self, a):
        self.a = a

class Child(Base):
    def __init__(self, a, b):
        super().__init__(a)        # ✅ Python 3 零参形式
        # Base.__init__(self, a)   # ❌ 硬编码父类，多重继承会出问题
        self.b = b
```

`super()` **不是"父类"，而是"MRO 中的下一个类"**——这是最重要的认知修正。

```python
class A:
    def go(self): print("A")
class B(A):
    def go(self): print("B"); super().go()
class C(A):
    def go(self): print("C"); super().go()
class D(B, C):
    def go(self): print("D"); super().go()

D().go()
#=> D
#   B
#   C     ← B 的 super() 指向了 C，而不是 B 的父类 A！
#   A
```

`super()` 在 `B.go` 中解析的是 **`type(self).__mro__` 中 B 之后的那个类**，即 C。
这就是**协作式多重继承（cooperative multiple inheritance）**。

零参 `super()` 等价于 `super(B, self)`：`B` 是**词法上所在的类**（编译器塞进 `__class__` cell），
`self` 决定用哪条 MRO。所以**同一行代码在不同实例上指向不同类**：

```python
B().go()             #=> B A         ← B.go 里的 super() → A
D().go()             #=> D B C A     ← 同一行 super() → C
```

`B.go` 一个字没改，目标从 `A` 变成 `C`，只因为 `self` 换了类型——
这是「`super()` 不是父类」的铁证。（用 `C` 演示不了：`C` 的 MRO 是 `(C, A, object)`，
C 实例和 D 实例上 `C.go` 的 `super()` 恰好都指向 `A`。）

反面对照：把 `B.go` 写成硬编码 `A.go(self)`，`D().go()` 变成 `D B A`——`C.go` 被整条跳过。
危害不在这个玩具例子，而在：以后有人写 `class E(B, AuditMixin, C)`，
`AuditMixin` 会因为 `B` 里那句硬编码**静默失效**，没有任何报错。
这是「永远用 `super()`、不写死父类名」的唯一理由。

## 5. MRO 与 C3 线性化

```python
D.__mro__
#=> (D, B, C, A, object)
D.mro()          # 同上，方法形式
```

**C3 线性化算法**保证三条性质：

1. **子类优先于父类**（D 在 B、C 之前）
2. **保持基类的声明顺序**（`class D(B, C)` ⟹ B 在 C 之前）
3. **单调性**：子类的 MRO 中，父类们的相对顺序与它们各自的 MRO 一致

C3 定义（merge 操作）：

```text
L[D] = D + merge(L[B], L[C], [B, C])
```

merge 规则：取第一个列表的表头，若它**不出现在其它任何列表的尾部**，就取出它；否则试下一个列表的表头。

无法线性化时直接报错：

```python
class X: pass
class Y: pass
class A(X, Y): pass
class B(Y, X): pass
class C(A, B): pass
# ❌ TypeError: Cannot create a consistent method resolution order (MRO) for bases X, Y
```

逐步推演（面试要求当场写出来的就是这个）：

```text
L[X] = [X, object]                L[Y] = [Y, object]
L[A] = [A, X, Y, object]          L[B] = [B, Y, X, object]

L[C] = C + merge([A,X,Y,object], [B,Y,X,object], [A,B])
  ① 头 A：不在任何列表尾部              → 取 A
     merge([X,Y,object], [B,Y,X,object], [B])
  ② 头 X：出现在 [B,Y,X,object] 的尾部   → 拒绍，试下一个列表的表头
     头 B：不在任何尾部                  → 取 B
     merge([X,Y,object], [Y,X,object], [])
  ③ 头 X：出现在 [Y,X,object] 的尾部     → 拒绍
     头 Y：出现在 [X,Y,object] 的尾部     → 拒绍
     所有表头都被拒 → 死锁 ✗
```

根因一句话：**`A` 要求 X 在 Y 之前，`B` 要求 Y 在 X 之前，两个约束不可能同时满足**。
注意报错发生在 **`class C` 的定义时刻**（`type` 建类时），不是实例化时。

> **面试落点**：能画出菱形继承的 MRO 并解释 "super 是 MRO 的下一个，不是父类" ——
> 这是区分"背过八股"和"真懂"的分界线。

### 协作式继承的正确姿势

```python
class Base:
    def __init__(self, **kw):
        super().__init__(**kw)          # 即使是"最后一个"也要调，让 MRO 链完整
class LoggingMixin:
    def __init__(self, *, logger=None, **kw):
        self.logger = logger
        super().__init__(**kw)          # ← 把剩余 kwargs 继续往下传
class Service(LoggingMixin, Base):
    def __init__(self, *, name, **kw):
        self.name = name
        super().__init__(**kw)

Service(name="s", logger=log)
```

每层只摘走自己的 keyword-only 参数，剩下的 `**kw` 原样下传。注意**链条末端会撞上 `object.__init__`**：

```python
Service(name="s", logger=log, typo=1)
# TypeError: object.__init__() takes exactly one argument (the instance to initialize)
```

`typo` 无人认领 → 一路漂到 MRO 终点 `object` → 报错。
**这是特性，不是 bug**：它让拼错的关键字参数在启动时就炸掉，而不是被无声吞掉。

```python
# ❌ 错的修法：把末端的 super() 删了或不传 kw
class Base:
    def __init__(self, **kw):
        pass                    # ① typo=1 被静默丢弃，参数名写错永远发现不了
                                # ② 链条断在 Base；今天 Base 之后是 object 看不出问题，
                                #    换成 class F(Base, OtherBase)，OtherBase.__init__ 永远不会被调
```

真要容忍额外参数，在**入口层**显式校验/剥离，不要在链条中间断链。

Mixin 的约定：**只提供方法、不定义 `__init__` 状态（或严格用 `**kwargs` 协作）、
放在继承列表左侧、名字以 `Mixin` 结尾**。

## 6. 抽象基类（ABC）

```python
from abc import ABC, abstractmethod

class Storage(ABC):
    @abstractmethod
    def get(self, key: str) -> bytes: ...

    @abstractmethod
    def put(self, key: str, val: bytes) -> None: ...

    def get_or_default(self, key, d=b""):     # 可以有具体方法
        try:
            return self.get(key)
        except KeyError:
            return d

Storage()            # ❌ TypeError: Can't instantiate abstract class Storage
class S3(Storage): pass
S3()                 # ❌ 同样报错，未实现抽象方法
```

**检查时机：定义期只收集，实例化才拦**。`ABCMeta` 建类时只把抽象方法名收进
`__abstractmethods__`，不报错；拦截点在 `object.__new__`：

```python
class S3(Storage): pass          # ← 类定义完全成功
S3.__abstractmethods__           #=> frozenset({'get', 'put'})
S3()                             # ← 到这一刻才 TypeError
```

> **面试落点**：这是 ABC 的本质弱点——**一个从没被实例化的坏子类可以一路合并进主干**。
> 正好和 C3 失败相反：**MRO 冲突在 `class` 语句执行时（定义期）就报错**，
> 而抽象方法缺失要等到实例化。这组时机对比是常考点。

`abstractmethod` 与 `classmethod`/`staticmethod`/`property` 组合时，`abstractmethod` 放**最里层**：

```python
class C(ABC):
    @property
    @abstractmethod              # ✅ 紧贴函数
    def name(self) -> str: ...
```

原理：`abstractmethod` 干的事就是给它包住的对象打一个 `__isabstractmethod__ = True`；
`property` 会检查自己的 `fget/fset/fdel` 有没有这个标记并**向上传播**。
所以必须让 `abstractmethod` 标记原始函数，再由 `property` 把标记冒泡出来。

写反不是「静默丢失抽象性」，而是**类定义那一刻直接报错**（CPython 3.11.9 实测）：

```python
class Bad(ABC):
    @abstractmethod
    @property                    # ❌
    def name(self): ...
# AttributeError: attribute '__isabstractmethod__' of 'property' objects is not writable
```

因为 `property.__isabstractmethod__` 是只读的计算属性，`abstractmethod` 写不进去。

**ABC vs Protocol**（面试常对比）：

| | `abc.ABC` | `typing.Protocol` |
|---|---|---|
| 子类型 | 名义（必须显式继承或 `register`） | **结构化**（有对应方法即可） |
| 检查时机 | 运行时（实例化时报错） | 静态（mypy），加 `@runtime_checkable` 后可 isinstance |
| 类比 | Java interface / TS `implements` | TypeScript `interface` 的鸭子类型 |
| 依赖方向 | 实现方必须 import 抽象方 | 抽象方可在使用侧定义，实现方毫不知情 |

Protocol 的两个运行时坑（都是「以为有保护，其实没有」）：

```python
@runtime_checkable
class Storage(Protocol):
    def get(self, key) -> bytes: ...

class Weird: get = 123
isinstance(Weird(), Storage)     #=> True    ← 只查方法名，不查签名、不查是不是 callable

class Empty(Storage): pass       # 显式继承 Protocol
Empty()                          #=> 成功！ ← 没实现 get 也放过
```

第二条是关键区别：**Protocol 不提供 ABC 那种运行时抽象性保护**，
它的约束力全在静态检查器。选型：跟库/跟团队的接口用 Protocol（免去 import 耦合）；
自己继承体系内要强制子类实现、想在运行时兜底就用 ABC。两者可叠加。

## 7. 常见问题速答

```python
# 类型检查
isinstance(x, C)      # 含子类，推荐
type(x) is C          # 精确类型
issubclass(D, B)      #=> True

# 内省
D.__bases__           #=> (B, C)  直接父类（声明顺序）
D.__mro__             #=> (D, B, C, A, object)  完整解析顺序
A.__subclasses__()    #=> [B, C]  直接子类
vars(obj)             # == obj.__dict__

# 动态创建类（等价于 class 语句）
D = type("D", (Base,), {"x": 1, "m": lambda self: self.x})

# 运行时给类加方法（monkey patching）
def new_m(self): return 42
C.m = new_m
```

Python **没有真正的方法重载**（同名后者覆盖前者）、**没有 final/private 修饰符**、
**没有接口关键字**——这些都靠约定 + ABC/Protocol + 静态检查工具实现。

## 相关

- [[language/descriptors-properties]] —— 方法绑定、property 的底层
- [[language/metaclasses]] —— 类是怎么被创建的
- [[language/data-model]] —— dunder 沿 MRO 查找
- [[language/typing]] —— Protocol 与结构化子类型
- [[internals/cpython-object-model]] —— `__dict__`、类型对象与属性查找
- [[interview/question-bank-language]] —— MRO / super 面试题
- [[review/review-set-02]] —— 本页考点的自测卷（含批改记录）
