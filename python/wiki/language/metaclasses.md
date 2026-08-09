---
title: "元类与类的创建过程"
date: 2026-08-07
tags: [元类, metaclass, type, __new__, __init_subclass__, 单例]
sources: ["interview-python-cn.md", "cpython-doc/datamodel.rst"]
---

# 元类与类的创建过程

「Python 中的元类」是中文面试题库里的**第 2 题**，几乎必问。但真正的加分点不是背诵定义，
而是能说出**「99% 的场景不该用元类，用 `__init_subclass__` 或类装饰器就够了」**。

## 1. 类也是对象

```python
class Foo: pass

type(Foo)          #=> <class 'type'>       类的类型是 type
type(type)         #=> <class 'type'>       type 是它自己的类型
isinstance(Foo, type)   #=> True
Foo.__class__      #=> type
```

**元类（metaclass）= 类的类**。实例由类创建，类由元类创建。

```text
实例  --instance of-->  类  --instance of-->  元类(type)
 3    ------------->   int  ------------->    type
```

`type` 的三参数形式可以**动态创建类**，与 `class` 语句完全等价：

```python
def greet(self): return f"hi {self.name}"

# 等价于 class Dog(Animal): kind = "dog"; greet = greet
Dog = type("Dog", (Animal,), {"kind": "dog", "greet": greet})

d = Dog(); d.name = "rex"; d.greet()   #=> 'hi rex'
```

三个参数：`type(name, bases, namespace)`。

## 2. `class` 语句执行时到底发生了什么

```python
class Foo(Base, metaclass=Meta, **kwds):
    x = 1
    def m(self): ...
```

解释器实际执行：

1. 确定元类：显式 `metaclass=` > 各基类元类中最派生的那个 > `type`
2. 调用 `Meta.__prepare__(name, bases, **kwds)` 得到一个命名空间映射（默认 `dict`）
3. **在该命名空间中执行类体**（`x = 1`、`def m` 都写进这个 dict）
4. 调用 `Meta(name, bases, namespace, **kwds)` → 走 `Meta.__call__`
   → `Meta.__new__` 创建类对象 → `Meta.__init__` 初始化
5. 对每个属性调用 `__set_name__`（若有）
6. 调用父类的 `__init_subclass__(cls, **kwds)`
7. 把结果绑定到名字 `Foo`

```python
class Meta(type):
    @classmethod
    def __prepare__(mcls, name, bases, **kw):
        print("1. prepare")
        return {}                                   # 返回 OrderedDict 可保留定义顺序
    def __new__(mcls, name, bases, ns, **kw):
        print("2. new  ", name, list(ns))
        return super().__new__(mcls, name, bases, ns)
    def __init__(cls, name, bases, ns, **kw):
        print("3. init ")
        super().__init__(name, bases, ns)
    def __call__(cls, *a, **kw):                    # 控制"实例化类"这个动作
        print("4. call (创建实例时)")
        return super().__call__(*a, **kw)

class Foo(metaclass=Meta):
    x = 1
#=> 1. prepare
#   2. new   Foo ['__module__', '__qualname__', 'x']
#   3. init

Foo()
#=> 4. call (创建实例时)
```

> **面试落点**：`Meta.__call__` 控制**实例创建**，`Meta.__new__` 控制**类创建**。
> 这是元类做单例、做实例缓存的入口。

## 3. `__new__` vs `__init__`（普通类层面）

```python
class C:
    def __new__(cls, *a, **kw):
        print("new")
        obj = super().__new__(cls)     # 必须返回实例，否则 __init__ 不会被调用
        return obj
    def __init__(self, x):
        print("init")
        self.x = x
```

| | `__new__` | `__init__` |
|---|---|---|
| 角色 | **构造器**：分配并返回对象 | **初始化器**：给已有对象填状态 |
| 首参 | `cls`（隐式 staticmethod） | `self` |
| 返回 | 必须返回实例（返回非 cls 实例则不调用 `__init__`） | 必须返回 `None` |

**什么时候必须用 `__new__`**：子类化不可变类型。

```python
class PositiveInt(int):
    def __new__(cls, v):
        if v <= 0:
            raise ValueError("must be positive")
        return super().__new__(cls, v)      # int 的值在 __new__ 就定死了
    # ❌ 在 __init__ 里改是没用的，int 已经创建完了

PositiveInt(5) + 1      #=> 6
```

## 4. 元类的真实用途

```python
# 用途 1：注册表（插件系统 / 序列化器 / ORM 表映射）
class RegistryMeta(type):
    registry = {}
    def __new__(mcls, name, bases, ns):
        cls = super().__new__(mcls, name, bases, ns)
        if bases:                                # 跳过基类自身
            RegistryMeta.registry[ns.get("key", name.lower())] = cls
        return cls

class Plugin(metaclass=RegistryMeta): pass
class JsonPlugin(Plugin): key = "json"
class YamlPlugin(Plugin): key = "yaml"

RegistryMeta.registry    #=> {'json': JsonPlugin, 'yaml': YamlPlugin}

# 用途 2：单例
class SingletonMeta(type):
    _inst = {}
    def __call__(cls, *a, **kw):
        if cls not in cls._inst:
            cls._inst[cls] = super().__call__(*a, **kw)
        return cls._inst[cls]

class Config(metaclass=SingletonMeta): pass
Config() is Config()     #=> True

# 用途 3：类定义时校验（ABC 就是这么实现的）
class InterfaceMeta(type):
    def __new__(mcls, name, bases, ns):
        if bases and "handle" not in ns:
            raise TypeError(f"{name} must implement handle()")
        return super().__new__(mcls, name, bases, ns)
```

**真实世界的元类**：`abc.ABCMeta`、Django `ModelBase`、SQLAlchemy `DeclarativeMeta`、
Pydantic v1 `ModelMetaclass`、`enum.EnumMeta`、`typing` 的部分实现。

## 5. 更该用的三个替代方案

### `__init_subclass__`（3.6+）—— 90% 的元类需求都能替代

```python
class Plugin:
    registry = {}
    def __init_subclass__(cls, /, key=None, **kw):     # 定义在父类，子类创建时自动调用
        super().__init_subclass__(**kw)
        Plugin.registry[key or cls.__name__.lower()] = cls

class Json(Plugin, key="json"): pass       # 类关键字参数直接传进来
Plugin.registry                            #=> {'json': <class 'Json'>}
```

比元类简单太多，且**不会污染继承体系的元类**（多重继承时元类冲突是真实痛点）。

### `__set_name__`（3.6+）—— 让描述符知道自己叫什么

以前需要元类扫描类体给字段命名，现在描述符自己就能拿到名字。见 [[language/descriptors-properties]]。

### 类装饰器 —— 后处理类对象

```python
def register(cls):
    REGISTRY[cls.__name__] = cls
    return cls

@register
class Handler: ...
```

`dataclass`、`total_ordering`、`attrs` 都走这条路。

> **面试落点**（这段话背下来）：
> 「元类控制类的创建。但从 Python 3.6 起，`__init_subclass__` + `__set_name__` + 类装饰器
> 覆盖了绝大多数原本需要元类的场景，且不会引入元类冲突。我在业务代码里不会用元类，
> 只有写框架、需要在类体执行前介入（`__prepare__`）或需要控制实例化行为（`__call__`）时才考虑。」
>
> Tim Peters 的名言值得引用：*"Metaclasses are deeper magic than 99% of users should ever worry
> about. If you wonder whether you need them, you don't."*

## 6. 元类冲突

```python
class M1(type): pass
class M2(type): pass
class A(metaclass=M1): pass
class B(metaclass=M2): pass
class C(A, B): pass
# ❌ TypeError: metaclass conflict: the metaclass of a derived class must be a
#    (non-strict) subclass of the metaclasses of all its bases
```

解法：造一个同时继承 M1、M2 的元类。这类问题在混用 ABC + Django Model + Pydantic 时真实发生。

```python
class M12(M1, M2): pass
class C(A, B, metaclass=M12): pass    # ✅
```

## 7. 相关内省

```python
Foo.__name__ / __qualname__ / __module__
Foo.__bases__ / __mro__
Foo.__dict__            # mappingproxy，类命名空间（只读视图）
Foo.__class__           # 元类
type(Foo).__mro__       #=> (type, object)
```

## 相关

- [[language/classes-mro]] —— 类的基础与 MRO
- [[language/descriptors-properties]] —— `__set_name__` 与描述符
- [[language/data-model]] —— `__new__` 在对象生命周期中的位置
- [[language/dataclasses-models]] —— dataclass 用类装饰器而非元类
- [[web/pydantic]] —— Pydantic v2 的模型构建
- [[interview/question-bank-language]] —— 元类面试题与标准答法
- [[sources/interview-python-cn]] —— 来源：中文面试题库第 2 题
