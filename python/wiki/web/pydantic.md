---
title: "Pydantic v2：校验、序列化与配置"
date: 2026-08-07
tags: [pydantic, 校验, 序列化, BaseModel, validator, settings]
sources: ["pydantic-doc/models.md", "pydantic-doc/validators.md", "pydantic-doc/fields.md", "pydantic-doc/performance.md"]
---

# Pydantic v2：校验、序列化与配置

Pydantic 是 FastAPI 的数据层核心。**v2（2023 年发布）把核心用 Rust 重写（pydantic-core），
比 v1 快 5~50 倍，但 API 有大量破坏性变更**——面试时用错 API 会暴露知识陈旧。

> 前端类比：**Pydantic ≈ zod**。都是"schema 即类型"、都做运行时校验 + 类型推导。
> 差别：zod 的 schema 是值，Pydantic 的 schema 是**类**，且能直接生成 JSON Schema。

## 1. 基本模型

```python
from pydantic import BaseModel, Field, ConfigDict, EmailStr
from datetime import datetime

class User(BaseModel):
    model_config = ConfigDict(
        extra="forbid",              # 多余字段直接报错（默认 'ignore'）
        str_strip_whitespace=True,
        frozen=False,                # True 则不可变且可哈希
        validate_assignment=True,    # 赋值时也校验（默认只在创建时校验）
        populate_by_name=True,       # 允许用字段名而非 alias 传入
        from_attributes=True,        # 允许从 ORM 对象读取（原 orm_mode）
    )

    id: int
    name: str = Field(min_length=1, max_length=50, description="用户名")
    email: EmailStr
    age: int = Field(default=0, ge=0, le=150)
    tags: list[str] = []                              # ✅ 每个实例独立，不共享
    created_at: datetime = Field(default_factory=datetime.now)
    internal: str = Field(default="", exclude=True)   # 序列化时排除
    user_id: int = Field(alias="userId")              # JSON 用 camelCase，Python 用 snake_case ★

u = User(id=1, name="bob", email="b@x.com", userId=9)
```

**alias 是前后端协作的关键**——前端要 camelCase，Python 要 snake_case：

```python
class Base(BaseModel):
    model_config = ConfigDict(
        alias_generator=lambda s: "".join(
            w if i == 0 else w.capitalize() for i, w in enumerate(s.split("_"))
        ),
        populate_by_name=True,
    )
# 之后所有模型继承 Base，自动 snake_case ↔ camelCase
u.model_dump(by_alias=True)      #=> {'userId': 9, ...}
```

## 2. v1 → v2 API 对照（必须记住）

| v1（已废弃） | v2 |
|---|---|
| `class Config:` | `model_config = ConfigDict(...)` |
| `.dict()` | **`.model_dump()`** |
| `.json()` | **`.model_dump_json()`** |
| `parse_obj()` | **`.model_validate()`** |
| `parse_raw()` | **`.model_validate_json()`** |
| `.schema()` | **`.model_json_schema()`** |
| `@validator` | **`@field_validator`** |
| `@root_validator` | **`@model_validator`** |
| `.copy()` | `.model_copy()` |
| `Config.orm_mode` | `model_config = ConfigDict(from_attributes=True)` |
| `Config.allow_mutation` | `frozen` |
| `constr(min_length=..)` | `Annotated[str, Field(min_length=..)]` |

```python
u.model_dump()                        # → dict
u.model_dump(mode="json")             # → dict，但 datetime/UUID 已转成 JSON 兼容类型
u.model_dump(exclude={"internal"}, exclude_none=True, by_alias=True)
u.model_dump_json(indent=2)
User.model_validate({"id": 1, ...})
User.model_validate_json(raw_bytes)   # 直接从 JSON 字节，最快（跳过中间 dict）
User.model_json_schema()
u.model_copy(update={"name": "x"}, deep=True)
```

## 3. 校验器

```python
from pydantic import field_validator, model_validator, ValidationInfo

class Order(BaseModel):
    price: float
    quantity: int
    total: float | None = None
    password: str
    password2: str

    @field_validator("price")
    @classmethod                                   # ← v2 必须是 classmethod
    def price_positive(cls, v: float) -> float:
        if v <= 0:
            raise ValueError("price must be positive")
        return v

    @field_validator("quantity", mode="before")    # before: 在类型转换【之前】
    @classmethod
    def parse_qty(cls, v):
        return int(v) if isinstance(v, str) else v

    @field_validator("*")                          # 所有字段
    @classmethod
    def no_empty(cls, v, info: ValidationInfo):
        info.field_name, info.data                 # 已校验完的其它字段
        return v

    @model_validator(mode="after")                 # 全部字段校验完后，做跨字段校验 ★
    def check_passwords(self):
        if self.password != self.password2:
            raise ValueError("passwords do not match")
        self.total = self.price * self.quantity
        return self

    @model_validator(mode="before")                # 拿到原始输入（dict），可整体改写
    @classmethod
    def preprocess(cls, data):
        if isinstance(data, dict) and "amount" in data:
            data["price"] = data.pop("amount")
        return data
```

**`mode` 的含义**：`before` = 类型强制转换之前（拿到原始值）；
`after` = 转换与基础校验之后（拿到目标类型的值）。跨字段校验必须用 `model_validator(mode="after")`。

序列化器：

```python
from pydantic import field_serializer, model_serializer

class M(BaseModel):
    created: datetime
    @field_serializer("created")
    def ser_dt(self, v: datetime, _info) -> str:
        return v.strftime("%Y-%m-%d")
```

## 4. 类型强制转换（coercion）——常见困惑源

Pydantic 默认是 **"lax"（宽松）模式**，会做合理的类型转换：

```python
class M(BaseModel):
    n: int
    f: float
    b: bool

M(n="42", f="1.5", b="yes")     #=> n=42, f=1.5, b=True   ✅ 字符串被转换
M(n="abc")                       # ❌ ValidationError
M(n=1.9)                         # ❌ v2 中 float→int 有损转换会报错（v1 会截断成 1）

# 严格模式：完全不转换
class Strict(BaseModel):
    model_config = ConfigDict(strict=True)
    n: int
Strict(n="42")                   # ❌ ValidationError: Input should be a valid integer

# 单字段严格
from pydantic import StrictInt
class M2(BaseModel):
    n: StrictInt
```

> **面试落点**：「Pydantic 会自动转换类型吗？」——**默认会（lax 模式）**，
> 比如 `"42"` → `42`；但 v2 收紧了有损转换（`1.9` → `int` 会报错，v1 会静默截断）。
> 需要严格时用 `strict=True` 或 `StrictInt`/`StrictStr`。
> **API 入口层建议保持 lax**（HTTP 传来的都是字符串），**内部关键逻辑用 strict**。

## 5. 错误处理

```python
from pydantic import ValidationError

try:
    User(id="x", name="")
except ValidationError as e:
    e.errors()
    #=> [{'type': 'int_parsing', 'loc': ('id',), 'msg': 'Input should be a valid integer',
    #     'input': 'x', 'url': '...'},
    #    {'type': 'string_too_short', 'loc': ('name',), ...}]
    e.json()
    e.error_count()
```

FastAPI 会自动把 `ValidationError` 转成 **422 Unprocessable Entity** 响应。
定制格式见 [[web/fastapi-core]] 的异常处理。

## 6. 常用类型

```python
from pydantic import (
    EmailStr, HttpUrl, AnyUrl, IPvAnyAddress, SecretStr, Json,
    PositiveInt, NonNegativeFloat, conint, constr, AwareDatetime,
)
from pydantic.types import UUID4
from typing import Annotated, Literal
from decimal import Decimal

class Cfg(BaseModel):
    url: HttpUrl
    email: EmailStr                              # 需要 pip install "pydantic[email]"
    secret: SecretStr                            # repr 时显示 **********，防日志泄漏 ★
    port: Annotated[int, Field(ge=1, le=65535)]
    mode: Literal["dev", "prod"]
    amount: Decimal = Field(max_digits=10, decimal_places=2)   # 金额用 Decimal
    when: AwareDatetime                          # 强制带时区
    payload: Json[dict]                          # 自动解析 JSON 字符串
```

**判别联合（discriminated union）**——多态请求体的正确写法：

```python
from typing import Literal, Union
from pydantic import Field

class Cat(BaseModel):
    kind: Literal["cat"]
    meow: str

class Dog(BaseModel):
    kind: Literal["dog"]
    bark: str

class Owner(BaseModel):
    pet: Union[Cat, Dog] = Field(discriminator="kind")   # ★ 按 kind 直接分派，快且报错清晰
```

> 前端类比：≈ TS 的可辨识联合（discriminated union）+ zod 的 `z.discriminatedUnion`。

## 7. `pydantic-settings`：类型安全的配置

```python
# pip install pydantic-settings
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        env_prefix="APP_",
        env_nested_delimiter="__",
        extra="ignore",
        case_sensitive=False,
    )

    database_url: str                       # 必填，缺失时启动就报错 ★
    redis_url: str = "redis://localhost"
    secret_key: SecretStr
    debug: bool = False
    allowed_hosts: list[str] = ["*"]        # 环境变量里写 JSON 数组或逗号分隔

settings = Settings()                        # 读取顺序：初始化参数 > 环境变量 > .env > 默认值
```

**价值**：配置错误在**进程启动时**就暴露，而不是半夜第一次读到那个键时才崩。
这是"12-factor app"配置管理的 Python 最佳实践。

## 8. 性能要点

```python
# ① 优先用 model_validate_json 而不是 json.loads + model_validate
User.model_validate_json(raw)          # 快（Rust 层直接解析）
User.model_validate(json.loads(raw))   # 慢（多一次 Python dict 构造）

# ② TypeAdapter：给非模型类型做校验，可复用（避免重复构建 schema）
from pydantic import TypeAdapter
ta = TypeAdapter(list[User])
users = ta.validate_python(raw_list)
ta.dump_json(users)

# ③ 模型定义放在模块级，不要在函数里反复定义（每次都要重新编译 schema）

# ④ 热路径上的内部数据结构用 dataclass 而非 BaseModel
#    Pydantic 的校验开销在高 QPS 下是真实成本

# ⑤ 用 model_construct 跳过校验（确信数据可信时）
User.model_construct(id=1, name="x")   # ⚠️ 完全不校验，只用于内部可信数据
```

> **面试落点**：能说出「Pydantic v2 的核心是 Rust 写的 pydantic-core，
> 校验逻辑在模型定义时被编译成一个 schema 树，运行时直接在 Rust 里跑，
> 所以比 v1 的纯 Python 递归校验快一个数量级」——很加分。

## 9. 与 dataclass / ORM 的关系

```python
# ① Pydantic 版 dataclass（有校验的 dataclass）
from pydantic.dataclasses import dataclass

@dataclass
class Point:
    x: int
    y: int

# ② 从 ORM 对象构造（原 orm_mode）
class UserOut(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    name: str

UserOut.model_validate(orm_user)        # 从对象属性读取，而不是 dict 键

# ③ SQLModel = SQLAlchemy + Pydantic 合体（FastAPI 作者出品）
from sqlmodel import SQLModel, Field
class Hero(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    name: str
```

选型建议：**API 层用 Pydantic，DB 层用 SQLAlchemy，两者之间显式转换**——
虽然 SQLModel 让二者合一很诱人，但在复杂项目里分离更清晰（见 [[web/fastapi-async-db]]）。

## 相关

- [[web/fastapi-core]] —— Pydantic 模型在路由中的使用
- [[web/fastapi-async-db]] —— ORM 模型与 Pydantic 模型的转换
- [[language/dataclasses-models]] —— 数据建模选型
- [[language/typing]] —— 注解基础
- [[interview/question-bank-web]] —— Pydantic 面试题
- [[sources/fastapi-docs]] —— 来源：Pydantic 官方文档
