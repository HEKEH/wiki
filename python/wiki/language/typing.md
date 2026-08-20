---
title: "类型注解与静态类型检查"
date: 2026-08-07
tags: [类型注解, typing, mypy, Protocol, 泛型, TypedDict]
sources: ["fastapi-doc/python-types.md", "python-cheatsheet.md"]
---

# 类型注解与静态类型检查

对前端工程师来说这是**最容易上手的一章**——Python 的类型系统和 TypeScript 高度同构，
而且 FastAPI/Pydantic 把注解从"文档"变成了**运行时行为的驱动**。这是转 Python 的最大优势项。

## 1. 基础语法与 TS 对照

```python
name: str = "bob"
age: int = 30
ratio: float = 1.5
ok: bool = True
raw: bytes = b"x"

def greet(name: str, times: int = 1) -> str:
    return f"hi {name}" * times

def log(msg: str) -> None: ...          # 无返回值
def loop() -> NoReturn: ...             # 永不返回（抛异常/死循环）
```

| TypeScript | Python 3.12+ |
|---|---|
| `string[]` / `Array<string>` | `list[str]` |
| `[string, number]` | `tuple[str, int]`（定长）/ `tuple[str, ...]`（变长） |
| `Record<string, number>` | `dict[str, int]` |
| `Set<string>` | `set[str]` |
| `string \| number` | `str \| int` |
| `string \| null` | `str \| None`（旧写法 `Optional[str]`） |
| `unknown` | `object` |
| `any` | `Any` |
| `never` | `Never` / `NoReturn` |
| `readonly T[]` | `Sequence[T]`（协变只读） |
| `interface` | `Protocol` 或 `TypedDict` |
| `type X = ...` | `type X = ...`（3.12+）/ `X: TypeAlias = ...` |
| `<T>` 泛型 | `def f[T](x: T) -> T`（3.12+）/ `TypeVar` |
| `as const` | `Literal[...]` / `Final` |
| `keyof T` | 无直接对应 |

> **版本提示**：Python 3.9 起可以直接用内置 `list[int]`/`dict[str,int]`，
> 不必再 `from typing import List, Dict`。3.10 起用 `X | Y` 代替 `Union[X, Y]`、`Optional[X]`。
> 面试时写老写法会显得知识陈旧。

## 2. 常用构造

```python
from typing import Any, Literal, Final, ClassVar, TypedDict, NamedTuple, Callable, Iterable, Self

Mode = Literal["r", "w", "a"]                # 字面量联合，等价 TS 的 'r' | 'w' | 'a'
MAX: Final = 100                             # 不可重新赋值（静态检查层面）

Handler = Callable[[int, str], bool]         # (n: number, s: string) => boolean
AnyHandler = Callable[..., None]

class Cfg(TypedDict):                        # 形状化的 dict，≈ TS interface
    host: str
    port: int
class CfgOpt(TypedDict, total=False):        # 全部可选，≈ Partial<T>
    debug: bool

class Point(NamedTuple):                     # 不可变、可解包的轻量记录
    x: int
    y: int = 0

class C:
    registry: ClassVar[dict] = {}            # 类变量，不是实例字段
    def clone(self) -> Self: ...             # 3.11+，返回"当前子类"类型
```

## 3. 泛型（3.12 新语法）

```python
# Python 3.12+ —— 和 TS 几乎一样
def first[T](items: list[T]) -> T | None:
    return items[0] if items else None

class Stack[T]:
    def __init__(self) -> None:
        self._items: list[T] = []
    def push(self, x: T) -> None: self._items.append(x)
    def pop(self) -> T: return self._items.pop()

type Pair[T] = tuple[T, T]                   # 泛型类型别名

# 3.11 及以前的等价写法（面试/老项目里仍常见）
from typing import TypeVar, Generic
T = TypeVar("T")
def first(items: list[T]) -> T | None: ...
class Stack(Generic[T]): ...
```

约束与边界：

```python
def maxi[T: (int, float)](a: T, b: T) -> T: ...      # 约束：只能是 int 或 float
def sortk[T: Comparable](xs: list[T]) -> list[T]: ...# 边界：T 必须是 Comparable 的子类型
```

## 4. Protocol：结构化子类型（= TS 的 interface）

```python
from typing import Protocol, runtime_checkable

class Closable(Protocol):
    def close(self) -> None: ...

def shutdown(x: Closable) -> None:      # 任何有 close() 的对象都能传，无需继承
    x.close()

@runtime_checkable                       # 加上才能 isinstance（只检查方法名，不检查签名）
class Sized(Protocol):
    def __len__(self) -> int: ...

isinstance([1, 2], Sized)               #=> True
```

两个运行时坑（都是「以为有保护，其实没有」，3.11.9 实测）：

```python
class Weird: __len__ = 123               # 只是个同名属性，不是方法
isinstance(Weird(), Sized)               #=> True    ← 只查名字，不查签名/是不是 callable

class Empty(Closable): pass              # 显式继承 Protocol
Empty()                                  #=> 成功！ ← 没实现 close 也放过
```

不加 `@runtime_checkable` 就用 `isinstance` 是**硬错误**（不是静默返回 False）：

```python
isinstance(x, Closable)
# TypeError: Instance and class checks can only be used with @runtime_checkable protocols
```

第二个坑是与 ABC 的关键区别：**Protocol 不提供 ABC 那种运行时抽象性保护**，
约束力全在静态检查器。对比详见 [[language/classes-mro]] §6。

> **面试落点**：「Python 怎么表达接口？」——三条路：
> ① `abc.ABC` 名义子类型（必须显式继承）；② `typing.Protocol` 结构化子类型（鸭子类型的静态化）；
> ③ 纯鸭子类型（不声明）。库的公开 API 优先 Protocol，需要提供默认实现或强制约束时用 ABC。

## 5. 注解在运行时的真相

**注解默认不做任何运行时检查**：

```python
def f(x: int) -> str:
    return x
f("not an int")     #=> 'not an int'，运行时毫无怨言
```

注解只是存进 `__annotations__`：

```python
f.__annotations__   #=> {'x': <class 'int'>, 'return': <class 'str'>}
```

需要运行时校验就用：

- **Pydantic**（FastAPI 的基石）——把注解变成校验 + 解析 + 序列化，见 [[web/pydantic]]
- `typeguard` / `beartype` —— 装饰器式运行时检查
- `pydantic.validate_call` —— 给普通函数加参数校验

```python
from pydantic import validate_call

@validate_call
def f(x: int) -> str:
    return str(x)

f("3")      #=> '3'   自动转换
f("abc")    # ❌ ValidationError
```

### 前向引用与 `from __future__ import annotations`

```python
class Node:
    def add(self, child: "Node") -> "Node":  # 类还没定义完，用字符串
        ...

# 或者在文件顶部（推荐）：所有注解变成惰性字符串，不再在定义时求值
from __future__ import annotations

class Node:
    def add(self, child: Node) -> Node: ...   # ✅ 不用引号
```

⚠️ **但 FastAPI/Pydantic 需要在运行时解析注解**，用 `from __future__ import annotations` 时，
必须保证被引用的名字在模块作用域可见（否则 `get_type_hints()` 会失败）。这是真实踩坑点。

```python
from typing import get_type_hints
get_type_hints(f)        # 解析字符串注解为真实类型对象
```

> Python 3.14 引入了 PEP 649 的**延迟注解求值**，让这个矛盾从根本上缓解。

## 6. mypy 实战

```bash
pip install mypy
mypy app/                      # 检查
mypy --strict app/             # 严格模式（推荐新项目直接开）
```

`pyproject.toml` 配置（≈ tsconfig.json）：

```toml
[tool.mypy]
python_version = "3.12"
strict = true
warn_unreachable = true
plugins = ["pydantic.mypy"]

[[tool.mypy.overrides]]
module = ["untyped_lib.*"]
ignore_missing_imports = true      # ≈ TS 的 skipLibCheck / 缺 @types
```

常用逃生舱：

```python
x = cast(int, get_value())          # ≈ TS 的 as int
assert isinstance(x, int)           # 运行时收窄 + 静态收窄
y: Any = something                  # 关闭检查
z = value  # type: ignore[arg-type] # 精确忽略某条规则
if TYPE_CHECKING:                   # 只在检查时导入，避免循环导入/运行时开销
    from heavy import Thing
```

### 类型收窄（narrowing）

和 TS 的 type guard 完全同构：

```python
def handle(x: int | str | None) -> str:
    if x is None:
        return "none"           # 这里 x: None
    if isinstance(x, int):
        return str(x + 1)       # 这里 x: int
    return x.upper()            # 这里 x: str

from typing import TypeGuard
def is_str_list(v: list[object]) -> TypeGuard[list[str]]:   # ≈ TS 的 v is string[]
    return all(isinstance(x, str) for x in v)
```

## 7. 协变/逆变与容器选择（进阶）

```python
from collections.abc import Sequence, Iterable, Mapping

def f(items: list[str]): ...        # 不变（invariant）：只能传 list[str]
def g(items: Sequence[str]): ...    # 协变（covariant）：list/tuple 都行，只读
def h(items: Iterable[str]): ...    # 最宽松：生成器也行
```

**经验法则**：**参数用最抽象的类型（`Iterable`/`Sequence`/`Mapping`），返回值用最具体的类型
（`list`/`dict`）**。这与 TS 里"参数逆变、返回协变"的直觉一致。

`list[int]` 不是 `list[float]` 的子类型（因为可变容器不变），这是新手常见的 mypy 报错源。

## 8. 现代 Python 类型工具生态对照

| 前端 | Python |
|---|---|
| TypeScript 编译器 `tsc` | `mypy` / `pyright`（微软，VS Code Pylance 内核）/ `ty`（Astral，新） |
| ESLint | `ruff`（同时做 lint + format，Rust 写的，极快） |
| Prettier | `ruff format`（原 `black`） |
| `zod` 运行时校验 | `pydantic` |
| `@types/*` | `types-*` stub 包 + `py.typed` 标记 |
| `tsconfig.json` | `pyproject.toml` 的 `[tool.mypy]` |

## 相关

- [[language/dataclasses-models]] —— dataclass / TypedDict / NamedTuple / Pydantic 的选型
- [[language/data-model]] —— Protocol 与鸭子类型
- [[web/pydantic]] —— 注解驱动的运行时校验
- [[web/fastapi-core]] —— FastAPI 完全靠注解生成 API 与文档
- [[engineering/tooling-quality]] —— mypy/ruff 工具链配置
- [[bridge/js-to-python]] —— TS↔Python 类型对照速查
