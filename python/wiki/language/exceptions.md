---
title: "异常体系与错误处理"
date: 2026-08-07
tags: [异常, EAFP, 异常链, ExceptionGroup, finally, 自定义异常]
sources: ["cpython-doc/datamodel.rst", "python-cheatsheet.md"]
---

# 异常体系与错误处理

## 1. 异常层级（必须记住的部分）

```text
BaseException
├── SystemExit              sys.exit()
├── KeyboardInterrupt       Ctrl+C
├── GeneratorExit           生成器 close()
└── Exception               ← 业务代码只该捕获这一支
    ├── ArithmeticError → ZeroDivisionError, OverflowError
    ├── LookupError     → IndexError, KeyError
    ├── OSError         → FileNotFoundError, PermissionError, ConnectionError,
    │                     TimeoutError, IsADirectoryError...
    ├── ValueError      → UnicodeDecodeError
    ├── TypeError
    ├── AttributeError
    ├── NameError       → UnboundLocalError
    ├── RuntimeError    → RecursionError, NotImplementedError
    ├── StopIteration / StopAsyncIteration
    ├── ImportError     → ModuleNotFoundError
    └── Warning         → DeprecationWarning...
```

> **面试落点**：`except Exception` **不会**捕获 `KeyboardInterrupt` 和 `SystemExit`，
> 因为它们直接继承 `BaseException`。这是设计上的刻意安排——`except Exception:` 的兜底
> 不应该让程序无法被 Ctrl+C 终止。写 `except BaseException:` 或裸 `except:` 是 bug。

## 2. 完整的 try 结构

```python
try:
    r = risky()
except (ValueError, TypeError) as e:      # 多个异常用元组
    log.warning("bad input: %s", e)
    raise                                  # 裸 raise = 原样重抛，保留完整 traceback
except OSError as e:
    handle(e)
else:
    use(r)          # 只有 try 块没抛异常才执行（把"不该被保护的代码"移出 try）
finally:
    cleanup()       # 无论如何都执行，包括 return/break/continue 路径
```

`else` 的价值（很多人不知道）：**缩小 try 的保护范围**，避免误捕获 `use(r)` 里抛的同类异常。

```python
# ❌ use() 里的 ValueError 也会被吞
try:
    r = parse(s)
    use(r)
except ValueError: ...

# ✅
try:
    r = parse(s)
except ValueError: ...
else:
    use(r)
```

### `finally` 里的 return 会吞掉异常（陷阱题）

```python
def f():
    try:
        raise ValueError
    finally:
        return 1        # ❌ 异常被静默丢弃

f()      #=> 1，没有任何异常
```

同理 `finally` 里的 `break`/`continue` 也会吞异常。**永远不要在 `finally` 里 return。**

## 3. EAFP vs LBYL —— Python 的错误处理哲学

```python
# LBYL (Look Before You Leap) —— 前端/C 风格
if key in d:
    v = d[key]

# EAFP (Easier to Ask Forgiveness than Permission) —— Python 风格
try:
    v = d[key]
except KeyError:
    v = default
```

**Python 社区偏好 EAFP**，理由：

1. **消除 TOCTOU 竞态**：`if os.path.exists(p): open(p)` 之间文件可能被删除。
2. **快乐路径更快**：`try` 块在无异常时开销接近零（3.11 起是"零成本异常"，
   try 的进入完全没有运行时开销）；而 `if` 每次都要检查。
3. 代码更线性。

但异常路径**很慢**（比 if 慢一个数量级）。**判据：预期成功率高 → EAFP；预期一半以上会失败 → LBYL。**

```python
# 高频失败场景，LBYL 更快
if key in cache:      # 命中率 10% 时，用 in 比 try/except 快
    ...
# 或者干脆用 dict.get / contextlib.suppress
v = d.get(key, default)
```

> **面试落点**：能说出「EAFP 是 Python 的默认风格，因为它避免竞态且快乐路径零开销；
> 但异常本身很贵，高失败率场景仍应 LBYL」——这是有经验的表现。

## 4. 异常链（`raise from`）

```python
try:
    json.loads(raw)
except json.JSONDecodeError as e:
    raise ConfigError("配置文件损坏") from e      # __cause__ = e，显式因果
```

三种形式：

| 写法 | traceback 显示 | 语义 |
|---|---|---|
| `raise New() from e` | "The above exception was the **direct cause** of..." | 显式转换，`__cause__` |
| `raise New()`（在 except 内） | "During handling..., another exception occurred" | 隐式，`__context__` |
| `raise New() from None` | 只显示新异常 | **抑制**上游细节（对外 API 常用） |

```python
e.__cause__       # from 指定的
e.__context__     # 自动记录的"处理时正在处理的异常"
e.__traceback__   # traceback 对象
e.__suppress_context__   # from None 时为 True
```

## 5. 自定义异常

```python
class AppError(Exception):
    """所有业务异常的基类——调用方可以只 catch 这一个"""

class NotFound(AppError):
    def __init__(self, kind: str, id_: str):
        self.kind, self.id = kind, id_
        super().__init__(f"{kind} {id_} not found")   # ← 一定要调 super().__init__

class ValidationError(AppError):
    def __init__(self, field: str, msg: str):
        self.field, self.msg = field, msg
        super().__init__(f"{field}: {msg}")

try:
    raise NotFound("user", "42")
except AppError as e:
    e.args           #=> ('user 42 not found',)
    str(e)           #=> 'user 42 not found'
```

设计原则：

- **每个库/应用有一个异常基类**，让调用方能一网打尽。
- 异常里带**结构化字段**（不只是字符串），方便上层做 HTTP 状态码映射。
- 继承最贴切的内建异常（找不到 → `LookupError` 系；参数不对 → `ValueError`）。

在 FastAPI 里的落地：

```python
@app.exception_handler(NotFound)
async def not_found_handler(request, exc: NotFound):
    return JSONResponse(status_code=404, content={"kind": exc.kind, "id": exc.id})
```

见 [[web/fastapi-architecture]]。

## 6. ExceptionGroup 与 `except*`（3.11+）

并发场景下会同时产生多个异常，`ExceptionGroup` 用来打包它们。

```python
async def main():
    async with asyncio.TaskGroup() as tg:      # 3.11+
        tg.create_task(fail1())
        tg.create_task(fail2())
# 两个任务都失败 → 抛出 ExceptionGroup 包含两个异常

try:
    await main()
except* ValueError as eg:        # except* 按类型"筛选"出子组
    print("值错误:", eg.exceptions)
except* ConnectionError as eg:
    print("连接错误:", eg.exceptions)
```

`except*` 与普通 `except` 的区别：**可能匹配多个分支**（一个组里既有 ValueError 又有
ConnectionError 时，两个分支都会执行）。这是 asyncio 结构化并发的配套设施，
见 [[concurrency/asyncio-patterns]]。

## 7. 实战要点

```python
# ① 记日志要用 exception/exc_info，自动带 traceback
try: ...
except Exception:
    log.exception("处理失败")            # ✅ 等价 log.error(..., exc_info=True)
    log.error("处理失败: %s", e)         # ❌ 丢失 traceback

# ② 不要用异常做控制流的滥用，但 StopIteration/KeyError 是语言内建的例外

# ③ 重试要区分"可重试"和"不可重试"
RETRYABLE = (ConnectionError, TimeoutError)

# ④ 断言不是错误处理：python -O 会移除所有 assert
assert user.is_admin      # ❌ 安全检查绝不能用 assert
if not user.is_admin:     # ✅
    raise PermissionError

# ⑤ 用 contextlib.suppress 表达"故意忽略"
from contextlib import suppress
with suppress(FileNotFoundError):
    os.remove(tmp)

# ⑥ 自定义异常的 pickle 兼容（multiprocessing 场景）
class E(Exception):
    def __init__(self, a, b):
        self.a, self.b = a, b
        super().__init__(a, b)     # ← args 完整才能被正确 pickle/unpickle

# ⑦ 3.11+ 给异常追加上下文笔记
try: ...
except Exception as e:
    e.add_note(f"request_id={rid}")
    raise
```

## 8. 与 JS 的差异

| | JavaScript | Python |
|---|---|---|
| 捕获所有 | `catch (e)` | `except Exception`（不是裸 `except`） |
| 类型区分 | 手动 `instanceof` 判断 | `except SpecificError` 原生多分支 |
| finally 吞异常 | 同样会（return 在 finally） | 同样会 |
| 异常链 | `new Error(m, {cause: e})`（ES2022） | `raise X from e`（很早就有） |
| 多异常聚合 | `AggregateError`（Promise.any） | `ExceptionGroup`（3.11+） |
| 异步错误 | unhandled rejection 静默 | Task 异常未取回会在 GC 时警告 |
| 自定义 | `class E extends Error` | `class E(Exception)` |

## 相关

- [[language/context-managers]] —— `__exit__` 与异常传播
- [[language/iterators-generators]] —— StopIteration 与 PEP 479
- [[concurrency/asyncio-patterns]] —— TaskGroup / CancelledError / ExceptionGroup
- [[web/fastapi-architecture]] —— 异常到 HTTP 响应的映射
- [[interview/question-bank-language]] —— finally/return 等陷阱题
