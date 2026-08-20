---
title: "复习题组 02 —— 类、MRO 与 ABC 两题（含批改记录）"
date: 2026-08-20
tags: [面试题, 复习, 类, MRO, C3, super, name-mangling, 类变量, ABC, Protocol]
sources: []
---

# 复习题组 02 —— 类、MRO 与 ABC

对应 [[interview/roadmap]] 的 **Day 5**（类、MRO/C3、super、classmethod/staticmethod），
主题页是 [[language/classes-mro]]。两道题各自串起一整片考点，
**每一行输出都在 CPython 3.11.9 上实测**（报错文案随版本微调处已标注）。

本组带**批改记录**：每小问除「参考答案 / 解析」外，还记下了实际作答时错在哪、
错法属于哪一类。用法——先只看「题目」自测，再对答案，最后读「批改记录」看自己是否同款。

| # | 主题 | 涉及页面 | 高频错点 |
|---|---|---|---|
| 1 | 类变量 / 实例变量 / name mangling / 三种方法 | [[language/classes-mro]] [[language/descriptors-properties]] | 可变类变量被子类共写；mangling 的**目的**是防命名冲突 |
| 2 | MRO / C3 / 协作式 `**kwargs` / ABC vs Protocol | [[language/classes-mro]] [[language/typing]] | 协作链末端撞 `object.__init__`；Protocol 是结构化子类型 |

---

## 第 1 题 · 类变量、name mangling 与三种方法

### 题目

```python
class Base:
    registry = []
    count = 0

    def __init__(self, name):
        self.name = name
        self.__tag = name.upper()
        Base.count += 1
        self.registry.append(name)

    @classmethod
    def create_pair(cls, a, b):
        return [cls(a), cls(b)]

    @staticmethod
    def norm(s):
        return s.strip().lower()


class Sub(Base):
    def __init__(self, name):
        super().__init__(name)
        self.__tag = name[::-1]


x, y = Base.create_pair("p", "q")
s = Sub("Rc")
```

**a.** 写出下列表达式的值，并说明属性**查找/写入**发生在哪个 `__dict__`：

```python
Base.registry
Sub.registry is Base.registry
Base.count, s.count
sorted(s.__dict__)
type(Sub.create_pair("m", "n")[0])
```

**b.** `s.__tag` 在类外访问会怎样？`s.__dict__` 里和 `tag` 相关的键有几个、叫什么？
为什么 `Sub.__init__` 那行赋值没有覆盖 `Base.__init__` 那行？这个机制的**设计目的**是什么？

**c.** 判断真假并说明理由：

```python
Sub.norm is Base.norm
s.norm("  X ") == Base.norm("  X ")
s.__init__.__func__ is Sub.__init__
Base.create_pair.__self__ is Base
```

**d.** 把 `Base.count += 1` 改成 `self.count += 1`，三个 count 分别变成什么？`+=` 展开成了什么？

**e.** `registry = []` 为什么是设计缺陷？修正版是什么？什么情况下「类变量存可变对象」是**故意**的？

### 参考答案

```text
a) ['p', 'q', 'Rc']                       ← ★ 最易错
   True
   3, 3
   ['_Base__tag', '_Sub__tag', 'name']    ← ★ 单下划线开头
   <class '__main__.Sub'>

b) AttributeError: 'Sub' object has no attribute '__tag'
   两个键：_Base__tag='RC'、_Sub__tag='cR'
   目的：防止继承体系中的命名冲突，不是访问控制

c) True / True / True / True

d) Base.count=0, x.count=1, s.count=1
   展开为 self.count = self.count + 1

e) 是缺陷：实例间串味 + 子类不隔离 + 永不释放
```

### 解析

#### a `registry` 里为什么有 `'Rc'`

`self.registry.append(...)` 里的 `self.registry` 是**读**操作：实例 `__dict__` 没有
→ 沿 `type(self).__mro__` 找到 `Base.registry`，拿到**同一个 list 对象**，`append` 就地改。
`Sub("Rc")` 走 `super().__init__` 进的是同一段代码，所以 Base 和 Sub 的所有实例共写一张表。

一条规则管住整题：

> **读**：实例 `__dict__` → 沿 `type(x).__mro__` 各类 `__dict__`；**写**：默认落进实例 `__dict__`。

所以 `s.count` 读到的是 `Base.count`（`'count' not in s.__dict__`），
而 `self.name = ...` 写的是 `s.__dict__`。

#### b name mangling 是改名，不是私有

`__x` 在类 `Foo` 体内被编译成 `_Foo__x`（**单**下划线 + 类名）。两个类各写各的，互不干扰：

```python
s.__dict__       #=> {'name': 'Rc', '_Base__tag': 'RC', '_Sub__tag': 'cR'}
s._Base__tag     #=> 'RC'    ← 照样读得到
```

> **面试落点**：mangling 的设计目的是**避免继承体系（尤其多重继承里互不知情的 mixin）
> 的命名冲突**，让基类的内部属性不被子类意外覆盖。它是编译期改名，**不是访问控制**；
> Python 没有真正的 private。答成「防止外部访问」会被反问 `s._Base__tag` 为什么能拿到。

这道题的代码本身就是它保护的东西的活证据：`Base` 与 `Sub` 各写各的 `__tag`，互不干扰。

#### c 四个 True 各有各的理由

| 表达式 | 为什么 |
|---|---|
| `Sub.norm is Base.norm` | `staticmethod.__get__` 原样返回底层函数，不造新对象 |
| `s.norm(...) == Base.norm(...)` | 静态方法不接收 `self`/`cls`，实例访问与类访问完全等价 |
| `s.__init__.__func__ is Sub.__init__` | 绑定方法的 `__func__` 就是 MRO 上找到的那个函数 |
| `Base.create_pair.__self__ is Base` | classmethod 绑定的是**类**——这正是 `cls(a)` 能多态构造的原因 |

对比一下就更清楚：`s.__init__ is s.__init__` 是 **False**（绑定方法每次现造），
而 `Sub.norm is Base.norm` 是 True。

#### d `+=` 对不可变对象 = 读 + 算 + 写，而写永远落在实例上

```python
Base.count = 0 | x.count = 1 | s.count = 1
x.__dict__ = {'name': 'p', 'count': 1}
```

展开是 `self.count = self.count + 1`，**不是** `self.count = Base.count + 1`：
右边第一次因实例 `__dict__` 为空而回退到类，第二次就读到自己的实例变量了。
计数器根本没进类里——这是「类级计数器」最经典的写坏方式。

#### e 三层缺陷

```python
Base.registry                   #=> ['p', 'q', 'Rc']
Sub.registry is Base.registry   #=> True
'registry' in Sub.__dict__      #=> False        ← Sub 没有自己的表
```

1. **实例间串味**：任何实例的 `append` 影响所有实例，实例拿不到「自己的」记录。
2. **子类不隔离**：`Sub` 往 `Base` 的表里写。要每个子类独立，得显式 `registry = []`，或：

   ```python
   class Base:
       def __init_subclass__(cls, **kw):
           super().__init_subclass__(**kw)
           cls.registry = []        # ✅ 每个子类一张独立表
   ```

3. **生命周期**：这张 list 永不释放，长期运行的进程里就是内存泄漏（持续持有元素引用）。

```python
# ✅ 每实例独立
class Base:
    def __init__(self, name):
        self.registry = []
```

**什么时候是故意的**：插件/序列化器注册表、全局缓存、类级常量表。判据是
「**语义上属于类而非实例**」。即便如此也要显式写 `Base.registry.append(...)`
而不是 `self.registry.append(...)`，否则读者分不清是有意共享还是踩坑；
只读的表应该用 `tuple` / `frozenset` / `MappingProxyType` 把可变性掐掉。

### 批改记录

| 小问 | 判定 | 错在哪 |
|---|---|---|
| a | ❌ | `registry` 漏了 `'Rc'`（没看出 `self.registry` 是共享读）；mangled 名写成 `__Base__tag`（多一个下划线）；把实例 `s` 与 `x` 搞混，漏了 `_Sub__tag` |
| b | ⚠️ | 机制对，**目的答成「避免外部使用」**——典型错答 |
| c | ✅ | 四个全对 |
| d | ✅ | 值全对；展开式写成 `self.count = Base.count + 1`，本例恰好等价，但调用两次就错 |
| e | ❌ | 答成「有意的设计，看不出缺陷」 |

---

## 第 2 题 · MRO、C3、协作式继承与 ABC

### 题目

#### (A) 菱形继承与 `super()`

```python
class A:
    def __init__(self, **kw):
        print("A.init", kw)
        super().__init__(**kw)
    def who(self): return "A"

class B(A):
    def __init__(self, *, b=1, **kw):
        print("B.init b=", b)
        super().__init__(**kw)
    def who(self): return "B->" + super().who()

class C(A):
    def __init__(self, *, c=2, **kw):
        print("C.init c=", c)
        super().__init__(**kw)
    def who(self): return "C->" + super().who()

class D(B, C):
    def who(self): return "D->" + super().who()

d = D(b=10, c=20)
print(d.who())
```

1. 写出 `D.__mro__` 和 `D.__bases__`，说清区别。
2. 写出**完整输出**（含打印顺序）。
3. `B.who` 里的 `super().who()` 调用了谁？为什么不是 `A.who`（关键词 `type(self)`）？
4. 把 `B.who` 改成 `return "B->" + A.who(self)`，结果变成什么？丢了什么？
5. `D(b=1, c=2, extra=3)` 会怎样？在哪一层、为什么？删掉 `A.__init__` 里的
   `super().__init__(**kw)` 能「修好」吗？
6. `B().who()` 与 `D().who()` 对比，说明「同一行 `super()` 在不同实例上指向不同类」。

#### (B) C3 失败

```python
class X: pass
class Y: pass
class P(X, Y): pass
class Q(Y, X): pass
class R(P, Q): pass          # ← ?
```

写出 merge 的逐步推演，指出在哪一步卡死、判定条件是什么、抛什么异常、**在什么时刻**抛。

#### (C) ABC

```python
from abc import ABC, abstractmethod

class Storage(ABC):
    @abstractmethod
    def get(self, key): ...
    def get_or_default(self, key, d=b""):
        try:
            return self.get(key)
        except KeyError:
            return d

class Mem(Storage):
    pass
```

1. `Storage()` / `Mem()` 分别报什么错？检查发生在**什么时刻**？
2. 写一个既是 `abstractmethod` 又是 `property` 的成员，说明顺序，以及写反会怎样。
3. 改用 `typing.Protocol` 表达同样约束，在「子类型判定」和「报错时机」上有哪两个本质差异？
   什么时候必须加 `@runtime_checkable`？

### 参考答案

```text
A1) __mro__ = (D, B, C, A, object)   __bases__ = (B, C)

A2) B.init b= 10
    C.init c= 20
    A.init {}
    D->B->C->A

A3) C.who —— super() 查 type(self).__mro__ 里 B 之后的类
A4) "D->B->A"，C.who 被整条跳过
A5) TypeError: object.__init__() takes exactly one argument
    （在 MRO 终点 object 处抛；删 super() 是错的修法）
A6) B().who() = "B->A"   vs   D().who() = "D->B->C->A"

B)  merge 在第三步双头被拒 → 死锁
    TypeError: Cannot create a consistent method resolution order (MRO) for bases X, Y
    抛在 class R 的**定义时刻**

C1) 都是实例化时 TypeError；Mem 的类定义完全成功
C2) @property 在上、@abstractmethod 紧贴函数；写反在 3.11 直接 AttributeError
C3) 名义 vs 结构化子类型；运行时 vs 静态；isinstance/issubclass 时必须 runtime_checkable
```

### 解析

#### A1 `__bases__` 是 C3 的输入，`__mro__` 是输出

`__bases__` 只是声明顺序的直接父类；`__mro__` 是 C3 线性化的结果，
属性查找和 `super()` 分派都按它走。

#### A2 三个易错点

```console
B.init b= 10
C.init c= 20
A.init {}
D->B->C->A
```

- **没有 `D.init`**——`D` 根本没定义 `__init__`。`D(...)` 沿 MRO 找到的第一个是 `B.__init__`。
- **`b=10, c=20` 不是默认值 1、2**：默认值只在没传时生效。
- **`A.init {}` 是空字典**：`b` 被 `B` 的 keyword-only 形参吃掉、`c` 被 `C` 吃掉，
  `**kw` 下传时已空。这正是协作式 `**kwargs` 的设计意图——
  **每层只摘走自己的参数，剩下的原样下传**。

#### A3 零参 `super()` 的真实语义

`super()` 等价于 `super(B, self)`：`B` 是**词法上所在的类**（编译器塞进 `__class__` cell），
`self` 决定用哪条 MRO。所以查的是 `type(self).__mro__` 里 `B` **之后**的类 = `C`。

#### A4 硬编码父类的真实代价

`D().who()` 变成 `"D->B->A"`，`C.who` 被整条跳过。危害不在这个玩具例子，而在：
以后有人写 `class E(B, AuditMixin, C)`，`AuditMixin` 的行为会因为 `B` 里那句硬编码
**静默失效**，没有任何报错。这是「永远用 `super()`、不写死父类名」的唯一理由。

#### A5 协作链的末端撞上 `object.__init__`

```console
B.init b= 1
C.init c= 2
A.init {'extra': 3}
TypeError: object.__init__() takes exactly one argument (the instance to initialize)
```

`extra` 没人认领，一路漂到 `A.__init__`；`A` 在 D 的 MRO 里的下一个是 **`object`**，
而 `object.__init__` 不接受多余参数，于是在**终点处**抛错。

**这是特性，不是 bug**：它让打错的关键字参数在启动时炸掉，而不是被无声吞掉。

删掉 `A.__init__` 里的 `super().__init__(**kw)` 能「不报错」，但错得彻底：
① `extra=3` 被静默丢弃，参数名写错永远发现不了；② 链条断在 `A`——
今天 `A` 之后是 `object` 看不出问题，换成 `class F(A, SomeOtherBase)`，
`SomeOtherBase.__init__` 就永远不会被调用了（和 A4 是同一个病）。
真要容忍额外参数，在**入口层**显式校验/剥离，不要在链条中间断链。

#### A6 同一行代码，不同目标

```python
B().who()   #=> 'B->A'            ← B.who 里的 super() → A
D().who()   #=> 'D->B->C->A'      ← 同一行 super() → C
```

`B.who` 一个字没改，目标从 `A` 变成 `C`，只因为 `self` 换了类型。

> **面试落点**：这就是「`super()` 不是父类，是 MRO 中的下一个」的铁证。
> 注意 `C.who` 演示不了这件事——`C` 的 MRO 是 `(C, A, object)`，
> C 实例和 D 实例上 `C.who` 的 `super()` 恰好都指向 `A`。

#### B 逐步推演

```text
L[X] = [X, object]                L[Y] = [Y, object]
L[P] = [P, X, Y, object]          L[Q] = [Q, Y, X, object]

L[R] = R + merge([P,X,Y,object], [Q,Y,X,object], [P,Q])
  ① 头 P：不在任何列表尾部              → 取 P
     merge([X,Y,object], [Q,Y,X,object], [Q])
  ② 头 X：出现在 [Q,Y,X,object] 的尾部   → 拒绝，试下一个列表的表头
     头 Q：不在任何尾部                  → 取 Q
     merge([X,Y,object], [Y,X,object], [])
  ③ 头 X：出现在 [Y,X,object] 的尾部     → 拒绝
     头 Y：出现在 [X,Y,object] 的尾部     → 拒绝
     所有表头都被拒 → 死锁 ✗
```

判定条件是「某个表头是否出现在**其它列表的尾部**（非首位）」。
根因一句话：**`P` 要求 X 在 Y 之前，`Q` 要求 Y 在 X 之前，两个约束不可能同时满足**。

报错发生在 **`class R` 的定义时刻**（`type` 建类时），不是实例化时——
和 (C) 的抽象方法检查时机正好相反，这组对比是常考点。

#### C1 ABC 的检查时机：定义期只收集，实例化才拦

```python
class Mem(Storage): pass         # ← 定义完全成功
Mem.__abstractmethods__          #=> frozenset({'get'})
Storage()   # TypeError: Can't instantiate abstract class Storage with abstract method get
Mem()       # TypeError: Can't instantiate abstract class Mem with abstract method get
```

`ABCMeta` 在建类时只**收集**抽象方法名到 `__abstractmethods__`，不报错；
拦截点在 `object.__new__`。（报错文案随 CPython 版本微调，上面是 3.11.9 实测原文。）

这是 ABC 的本质弱点：**一个从没被实例化的坏子类可以一路合并进主干**。

#### C2 `abstractmethod` 必须在最里层

```python
class C(ABC):
    @property
    @abstractmethod          # ✅ 紧贴函数
    def name(self) -> str: ...
```

原理：`abstractmethod` 干的事就是给它包住的对象打 `__isabstractmethod__ = True`；
`property` 会检查 `fget/fset/fdel` 有没有这个标记并**向上传播**。
所以必须让 `abstractmethod` 标记原始函数，再由 `property` 把标记冒泡出来。

写反在 3.11.9 上**类定义那一刻就炸**：

```python
class Bad(ABC):
    @abstractmethod
    @property                # ❌
    def name(self): ...
# AttributeError: attribute '__isabstractmethod__' of 'property' objects is not writable
```

因为 `property.__isabstractmethod__` 是只读的计算属性，写不进去。

#### C3 ABC vs Protocol

| | `abc.ABC` | `typing.Protocol` |
|---|---|---|
| 子类型判定 | **名义**（nominal）：必须显式继承或 `register` | **结构化**：有对应方法就算，双方零耦合 |
| 报错时机 | **运行时**，实例化那一刻 | **静态**（mypy/pyright），运行时默认不管 |
| 依赖方向 | 实现方必须 import 抽象方 | 抽象方可在使用侧定义，实现方毫不知情 |

```python
class Storage(Protocol):
    def get(self, key) -> bytes: ...

class Mem:                       # 完全不继承 Storage
    def get(self, key): return b"x"

def use(s: Storage): return s.get("k")
use(Mem())                       #=> b'x'   ✅ 结构化子类型
```

`@runtime_checkable` 的必要条件：**只要想对 Protocol 用 `isinstance`/`issubclass`**。
不加是硬错误：

```python
isinstance(Mem(), Storage)
# TypeError: Instance and class checks can only be used with @runtime_checkable protocols
```

两个必须知道的坑：

```python
@runtime_checkable
class RStorage(Protocol):
    def get(self, key) -> bytes: ...

class Weird: get = 123
isinstance(Weird(), RStorage)     #=> True    ← 只查方法名，不查签名

class Empty(Storage): pass        # Storage 是 Protocol
Empty()                           #=> 成功！  ← 没实现 get 也放过
```

> **面试落点**：跨库/跨团队的接口用 Protocol（免去 import 耦合）；
> 自己继承体系内要强制子类实现、想在运行时兜底就用 ABC。两者可叠加。

### 批改记录

| 小问 | 判定 | 错在哪 |
|---|---|---|
| A1 | ✅ | |
| A2 | ❌ | 凭空补了不存在的 `D.init`；把实参 10/20 当成默认值 1/2；`A.init` 的 kw 没算成空 |
| A3 | ✅ | |
| A4 | ✅ | |
| A5 | ❌ | 答「看不出问题」——没意识到 `**kw` 会漂到 `object.__init__` |
| A6 | ✅ | |
| B | ⚠️ | 异常猜对，但 merge 推演不成立（说成「表头 X、尾部 Y」，实际是 X/Y 互相堵死） |
| C1 | ✅ | |
| C2 | ✅ | 顺序对；未答写反的后果 |
| C3 | ❌ | 空白 |

> 出题自省：A6 原本让对比 `C().who()`，但 `C.who` 的 `super()` 在 C 实例和 D 实例上
> 都指向 `A`，演示不了「同一行指向不同类」。能演示的是 `B`。上面已改为 `B` 版本。

---

## 贯穿两题的两条主线

**主线一：读和写不对称。** 本组四处考的是同一件事——
Python 读属性沿 MRO 往上找，写属性默认落在实例上：

| 题 | 现象 |
|---|---|
| 1-a | `self.registry.append` 是**读** + 就地改 → 全体共享 |
| 1-d | `self.count += 1` 是读类变量 + 写实例变量 → 类级计数器失效 |
| 1-a | `s.count` 读到 `Base.count`，因为 `'count' not in s.__dict__` |
| 1-e | `'registry' in Sub.__dict__` 是 False —— 子类没有自己的表 |

**主线二：链条断了不报错。** `super()` 与 ABC 的三处考点都是「静默失效」：

| 题 | 你以为 | 实际 |
|---|---|---|
| 2-A4 | 硬编码 `A.who(self)` 只是「跳过一层」 | 后来插进来的任何 mixin 全部静默失效 |
| 2-A5 | 删掉末端 `super().__init__` 是修 bug | 是把参数拼写错误变成静默丢弃 |
| 2-C1 | 抽象方法没实现会在定义时拦住 | 定义期只收集，坏子类能一路合并进主干 |

答题时不要只说结果，要说清「它为什么不报错」——那才是考官想听的。
这与 [[review/review-set-01]] 的「不报错的错」是同一条主线。

## 相关

- [[interview/roadmap]] —— 本组题对应 Day 5
- [[review/review-set-01]] —— 语言核心五题（Day 1–4）
- [[language/classes-mro]] —— 主题页
- [[language/descriptors-properties]] —— 方法绑定、`staticmethod`/`classmethod` 的描述符本质
- [[language/typing]] —— Protocol 与结构化子类型
- [[interview/question-bank-language]] —— MRO / super 分主题题库
- [[interview/traps]] —— 单点陷阱速查
