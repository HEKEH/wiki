---
title: "数据类与数据建模选型"
date: 2026-08-07
tags: [dataclass, NamedTuple, TypedDict, Enum, pydantic, attrs, 数据建模]
sources: ["pydantic-doc/models.md", "python-cheatsheet.md"]
---

# 数据类与数据建模选型

「一堆字段的数据对象」在 Python 里有 6 种写法，选错会被面试官追问。这一页给出决策树。

## 1. 六种方案速览

```python
# ① 裸 dict —— 无结构、无校验、拼错 key 不报错
u = {"name": "bob", "age": 30}

# ② NamedTuple —— 不可变、可解包、轻量、有序
from typing import NamedTuple
class Point(NamedTuple):
    x: int
    y: int = 0
p = Point(1, 2); p.x; x, y = p; p._replace(x=9)

# ③ TypedDict —— 就是 dict，只给静态检查器看形状（零运行时开销）
from typing import TypedDict
class UserDict(TypedDict):
    name: str
    age: int

# ④ dataclass —— 标准库，自动生成 __init__/__repr__/__eq__
from dataclasses import dataclass, field
@dataclass(slots=True, frozen=True, kw_only=True)
class User:
    name: str
    age: int = 0
    tags: list[str] = field(default_factory=list)

# ⑤ attrs —— dataclass 的前身与超集，功能更多（converter、更强 validator）
import attrs
@attrs.define
class U:
    name: str = attrs.field(validator=attrs.validators.min_len(1))

# ⑥ Pydantic BaseModel —— 运行时校验 + 类型强制转换 + JSON 序列化
from pydantic import BaseModel, Field
class UserIn(BaseModel):
    name: str = Field(min_length=1)
    age: int = 0
```

## 2. 决策树

```text
数据要从外部（HTTP/JSON/YAML/DB）进来，需要校验和类型转换？
├── 是 → Pydantic BaseModel                     ← API 边界、配置、消息队列
└── 否（纯内部数据）
    ├── 需要不可变 + 元组语义（可解包、可当 dict key）？→ NamedTuple / frozen dataclass
    ├── 只是给 dict 加个类型形状（不想改运行时结构）？→ TypedDict
    ├── 需要海量小对象、极致内存？→ dataclass(slots=True) / __slots__ 类
    └── 一般内部实体 → dataclass                 ← 默认选它
```

> **面试落点**：「dataclass 和 pydantic 有什么区别？」
> 核心一句：**dataclass 只生成样板代码，不做任何运行时校验；Pydantic 会校验并强制转换类型**。
> `@dataclass class U: age: int` 传 `U(age="abc")` 完全合法；Pydantic 会抛 ValidationError。
> 因此**边界用 Pydantic，内部用 dataclass** 是通行架构。

## 3. dataclass 深入

```python
from dataclasses import dataclass, field, asdict, astuple, replace, fields

@dataclass(
    frozen=True,        # 不可变 + 自动生成 __hash__
    slots=True,         # 3.10+，生成 __slots__，省内存
    kw_only=True,       # 3.10+，全部字段变关键字-only（解决"有默认值字段必须在后面"的限制）
    order=True,         # 生成 __lt__/__le__/__gt__/__ge__（按字段顺序比较）
    eq=True,            # 默认 True
    repr=True,          # 默认 True
)
class Item:
    name: str
    price: float
    tags: list[str] = field(default_factory=list)     # ✅ 可变默认值必须用 factory
    _cache: dict = field(default_factory=dict, repr=False, compare=False)
    computed: str = field(init=False, default="")     # 不进 __init__

    def __post_init__(self):                          # __init__ 之后的钩子
        self.computed = self.name.upper()             # frozen 时要用 object.__setattr__

i = Item(name="a", price=1.0)
asdict(i)          #=> {'name': 'a', ...}   递归转 dict
astuple(i)
replace(i, price=2.0)         # 生成修改了某字段的新实例（frozen 场景必备）
[f.name for f in fields(i)]
```

**踩坑清单**：

```python
@dataclass
class Bad:
    tags: list[str] = []          # ❌ ValueError: mutable default ... use default_factory
                                  #    （dataclass 主动帮你拦住了这个经典陷阱）

@dataclass
class Bad2:
    a: int = 0
    b: int                        # ❌ TypeError: non-default argument follows default
                                  #    解法：kw_only=True 或调整顺序

@dataclass(frozen=True)
class F:
    x: int
    def __post_init__(self):
        self.x = 1                # ❌ FrozenInstanceError
        object.__setattr__(self, "x", 1)   # ✅ 绕过

@dataclass
class NoAnn:
    x = 1                         # ⚠️ 没有类型注解 → 不是字段，只是类变量！
```

`ClassVar` 也不会成为字段：

```python
from typing import ClassVar
@dataclass
class C:
    registry: ClassVar[dict] = {}   # 类变量，不进 __init__
    x: int
```

## 4. Enum

```python
from enum import Enum, StrEnum, IntEnum, auto, Flag

class Status(StrEnum):          # 3.11+，成员同时是 str，JSON 序列化友好
    ACTIVE = "active"
    BANNED = "banned"

Status.ACTIVE == "active"       #=> True   （StrEnum 特性）
Status("active")                #=> Status.ACTIVE   按值查找
Status["ACTIVE"]                #=> Status.ACTIVE   按名查找
list(Status)                    #=> [Status.ACTIVE, Status.BANNED]

class Color(Enum):
    RED = auto()                # 自动 1, 2, 3
    GREEN = auto()

class Perm(Flag):               # 位标志，可组合
    READ = auto(); WRITE = auto(); EXEC = auto()
p = Perm.READ | Perm.WRITE
Perm.READ in p                  #=> True
```

`StrEnum`/`IntEnum` 在 API 层特别有用——FastAPI 用 Enum 自动生成 OpenAPI 的枚举约束。

> 前端类比：`StrEnum` ≈ TS 的 `enum Status { ACTIVE = "active" }`；
> `Literal["active", "banned"]` ≈ TS 的联合字面量（更轻，无需类）。

## 5. Pydantic v2 简表（详见 [[web/pydantic]]）

```python
from pydantic import BaseModel, Field, field_validator, ConfigDict

class UserIn(BaseModel):
    model_config = ConfigDict(extra="forbid", frozen=False, str_strip_whitespace=True)

    name: str = Field(min_length=1, max_length=50)
    age: int = Field(ge=0, le=150)
    email: str | None = None
    tags: list[str] = []          # ✅ Pydantic 会为每个实例深拷贝默认值，不共享（与 dataclass 不同）

    @field_validator("email")
    @classmethod
    def check_email(cls, v):
        if v and "@" not in v: raise ValueError("invalid")
        return v

u = UserIn(name="bob", age="30")   # 字符串 "30" 被强制转成 int 30
u.model_dump()                     #=> {'name': 'bob', 'age': 30, ...}
u.model_dump_json()
UserIn.model_validate_json(raw)
UserIn.model_json_schema()         # 直接得到 JSON Schema（OpenAPI 的来源）
```

> ⚠️ 注意上面 `tags: list[str] = []` 在 **Pydantic 里是安全的**（每个实例独立），
> 但在 **dataclass / 普通函数默认参数里是 bug**。这个不对称是常见混淆点。

## 6. 完整对比表

| | dict | TypedDict | NamedTuple | dataclass | Pydantic |
|---|---|---|---|---|---|
| 运行时校验 | ❌ | ❌ | ❌ | ❌ | ✅ |
| 类型转换 | ❌ | ❌ | ❌ | ❌ | ✅ |
| 静态类型 | ❌ | ✅ | ✅ | ✅ | ✅ |
| 属性访问 `.x` | ❌ | ❌ | ✅ | ✅ | ✅ |
| 可变 | ✅ | ✅ | ❌ | 可选 | 可选 |
| 可哈希 | ❌ | ❌ | ✅ | frozen 时 | 配置后 |
| 可解包 | ❌ | ❌ | ✅ | ❌ | ❌ |
| JSON 序列化 | 原生 | 原生 | 需转换 | `asdict` | `model_dump_json` |
| JSON Schema | ❌ | ❌ | ❌ | ❌ | ✅ |
| 内存开销 | 中 | 中 | **最低** | 低（slots） | 高 |
| 创建速度 | 最快 | 最快 | 快 | 快 | 慢（有校验） |
| 依赖 | 内置 | 内置 | 内置 | 内置 | 第三方 |

**性能感知**：Pydantic v2 的核心用 Rust（pydantic-core）重写，比 v1 快 5~50 倍，
但仍比 dataclass 慢一个量级。**热路径上的内部对象不要用 Pydantic。**

## 相关

- [[language/typing]] —— TypedDict / NamedTuple 的类型层含义
- [[language/descriptors-properties]] —— `slots=True` 的原理
- [[language/objects-mutability]] —— 可变默认值陷阱
- [[web/pydantic]] —— Pydantic v2 完整用法
- [[web/fastapi-core]] —— 请求/响应模型的分层设计
- [[interview/question-bank-web]] —— dataclass vs pydantic 面试题
