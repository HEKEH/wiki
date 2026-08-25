---
title: "异常体系与错误处理"
date: 2026-08-07
tags: [异常, EAFP, 异常链, ExceptionGroup, finally, 自定义异常, 日志, 重试, add_note]
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

但异常路径**明显更慢**（实测约 4~5 倍，见 §7.6 的基准数据）。
**判据：预期成功率高 → EAFP；预期一半以上会失败 → LBYL。**

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

### 7.1 日志：永远带上 traceback

```python
try:
    handle(req)
except Exception as e:
    log.exception("处理失败")                     # ✅ 自动附加当前异常的 traceback
    log.error("处理失败", exc_info=True)          # ✅ 完全等价
    log.error("处理失败: %s", e)                  # ❌ 只有一行消息，丢了栈
    log.error(f"处理失败: {e}")                   # ❌❌ 还额外丢了 logging 的懒格式化
```

`log.exception()` 只能在 `except` 块内调用（它读的是 `sys.exc_info()`）；在别处调用会记下
`NoneType: None`。要在 except 块外记录，把异常对象传进去：`log.error("...", exc_info=exc)`。

**`str(e)` 经常是空的**——异常消息不是必填项：

```python
str(ValueError())            #=> ''            ← 只打 str(e) 等于什么都没打
str(KeyError("user_id"))     #=> "'user_id'"   ← KeyError 的 str 是 repr(key)，带引号
repr(ValueError())           #=> 'ValueError()'
f"{type(e).__name__}: {e}"   # ✅ 手工拼消息时的最小安全写法
```

`OSError` 家族带结构化字段，别去解析字符串：

```python
try:
    open("/nope/nope")
except OSError as e:
    e.errno       #=> 2
    e.strerror    #=> 'No such file or directory'
    e.filename    #=> '/nope/nope'
    str(e)        #=> "[Errno 2] No such file or directory: '/nope/nope'"
```

要把栈转成字符串（写进响应体、上报到自建平台）用 `traceback`：

```python
import traceback
traceback.format_exc()                     # 当前异常的完整栈，单个 str
traceback.format_exception(e)              # 3.10+ 可只传异常对象，返回 list[str]
traceback.format_exception_only(e)         # 只要「类型: 消息」+ __notes__
```

### 7.2 兜底屏障：每个「入口」都要有一个

异常只在**它自己的调用栈**里冒泡。进程有几个入口（主线程、子线程、事件循环、信号处理），
就得有几层兜底，否则异常会安静地消失在某个线程/任务里。

```python
# ① 同步主入口
def main() -> int:
    try:
        run()
    except KeyboardInterrupt:
        return 130                      # 用户 Ctrl+C 不是崩溃，别打整页栈
    except Exception:
        log.exception("fatal")
        return 1
    return 0

# ② 进程级最后一道：未捕获异常的钩子（把栈送进日志系统而不是 stderr）
def hook(exc_type, exc, tb):
    log.critical("uncaught", exc_info=(exc_type, exc, tb))
sys.excepthook = hook
threading.excepthook = lambda a: log.critical("thread crash", exc_info=a.exc_value)  # 3.8+

# ③ asyncio：回调与后台任务的异常不走 sys.excepthook
loop.set_exception_handler(lambda loop, ctx: log.error("loop: %s", ctx["message"]))
```

**最常见的线上事故：Task 的异常被吞。** 只要没人 `await` 它、也没人调 `task.result()`，
异常就一直挂在 Task 上，直到对象被 GC 才打印一行 `Task exception was never retrieved`——
时间点完全错位，而且不会让进程失败。

```python
async def boom(): raise ValueError("task died")

t = asyncio.create_task(boom())     # ❌ fire-and-forget，还丢了强引用
# ERROR Task exception was never retrieved   ← 几秒后才出现，甚至可能永远不出现

# ✅ 首选 TaskGroup（3.11+）：异常汇总成 ExceptionGroup 抛给 async with
async with asyncio.TaskGroup() as tg:
    tg.create_task(boom())

# ✅ 真需要 fire-and-forget：保住引用 + 挂 done callback 取回异常
bg: set[asyncio.Task] = set()
def spawn(coro):
    t = asyncio.create_task(coro)
    bg.add(t)
    t.add_done_callback(bg.discard)
    t.add_done_callback(lambda t: None if t.cancelled() or not t.exception()
                        else log.error("bg task failed", exc_info=t.exception()))
    return t
```

同类「异常无处可去」的地方还有 `__del__`：里面抛出的异常无法传播，只会往 stderr 打一段
`Exception ignored in: ...` 然后被丢弃，程序继续跑。**所以 `__del__` 里不要放会失败的清理逻辑**——
用 [[language/context-managers]] 的显式 `close()` / `with`。

### 7.3 重试：先分类，再退避

```python
RETRYABLE = (ConnectionError, TimeoutError)          # 瞬时故障：网络抖动、限流、502
FATAL = (ValueError, PermissionError)                # 重试一万次结果一样：参数错、鉴权失败

def retry(fn, attempts=5, base=0.2, cap=10.0):
    for i in range(attempts):
        try:
            return fn()
        except RETRYABLE as e:
            if i == attempts - 1:
                raise RuntimeError(f"重试 {attempts} 次仍失败") from e   # ← 保留 __cause__
            time.sleep(random.uniform(0, min(cap, base * 2 ** i)))       # 指数退避 + full jitter
```

四个易错点：

1. **别重试非幂等写操作**，除非带幂等键。
2. **退避必须加 jitter**，否则大量客户端同时失败会同步重试，形成惊群（thundering herd）。
3. **`asyncio.CancelledError` 绝不能重试**，也不该被 `except Exception` 吞——3.8 起它改为继承
   `BaseException` 正是为此。见 [[concurrency/asyncio-patterns]]。
4. 生产环境别自己写，用 `tenacity`：
   `@retry(retry=retry_if_exception_type(RETRYABLE), wait=wait_exponential_jitter(), stop=stop_after_attempt(5))`。

### 7.4 assert 不是错误处理

`python -O` 会把 `assert` 整句移除（编译期就不生成字节码）。

```python
assert user.is_admin                  # ❌ 安全检查用 assert = 开 -O 后校验凭空消失
assert isinstance(x, int), "bad x"    # ❌ 校验外部输入
if not user.is_admin:                 # ✅
    raise PermissionError
assert self._lock.locked()            # ✅ 断言只用于「代码有 bug 才会失败」的内部不变量
```

顺带一个坑：`assert (cond, "msg")` 写成元组则**恒为真**（非空元组真值为 True），
3.12 起会给 `SyntaxWarning`。

### 7.5 表达「故意忽略」

```python
from contextlib import suppress

with suppress(FileNotFoundError):     # ✅ 意图明确：这个文件本来就可能不存在
    os.remove(tmp)

try:                                  # ⚠️ 等价但三行噪音，且容易被后人塞逻辑进来
    os.remove(tmp)
except FileNotFoundError:
    pass

except Exception:                     # ❌❌ 裸 pass 吞一切，是最难查的 bug 来源
    pass
```

`suppress` 不是「更快的 try」——它多一层上下文管理器调用开销；它的价值是**语义**。
反过来，清理路径的正确写法是 `try/finally` 或上下文管理器，而不是在每个 return 前复制一遍
cleanup。

### 7.6 性能与内存：两个真实成本

**（一）快乐路径确实零开销，异常路径确实贵。** CPython 3.11 起用零成本异常（异常表取代
`SETUP_FINALLY` 字节码）。实测（3.11.9 / Windows，`timeit` 100 万次，单位 ns/次）：

| 写法 | 命中 | 未命中 |
|---|---|---|
| `try: d[k]` / `except KeyError` | **17** | 76 |
| `if k in d: d[k]` | 24 | **14** |
| `d.get(k)` | 16 | 16 |
| `try: raise ValueError` / `except` | — | 64 |

和 §3 的结论一致：**成功率高 → EAFP 反而更快**（连 `in` 的那次额外哈希都省了）；
失败率高 → LBYL 或 `dict.get`。异常路径约贵 4~5 倍，放进百万次热循环里就足够可观。

**（二）保存异常对象会钉住整个调用栈。** `e.__traceback__` → frame → 该帧全部局部变量，
形成强引用（而且常常是引用循环，只能等分代 GC 回收）：

```python
def f():
    huge = load_10mb()
    raise ValueError

errors = []
try:
    f()
except ValueError as e:
    errors.append(e)          # ❌ 那 10MB 跟着 traceback 活到 errors 被清空为止
    errors.append(str(e))     # ✅ 只留消息；要留栈就存 traceback.format_exc() 字符串
```

正因为这个陷阱，`except X as e:` 的 `e` 在**块结束时会被语言自动 `del`**：

```python
try:
    raise ValueError
except ValueError as e:
    pass
e        #=> NameError: name 'e' is not defined     ← 不是作用域问题，是显式删除
```

所以要在块外用异常对象，必须先拷到别的名字（`err = e`）。这也是 `log.exception()` 比
手工传 `e` 更省心的原因之一。

### 7.7 跨进程：异常必须能 pickle

`multiprocessing` / `ProcessPoolExecutor` 会把子进程的异常 pickle 回主进程。默认的
`BaseException.__reduce__` 用 `type(e)(*e.args)` 重建对象——**`args` 与 `__init__` 签名不匹配就炸**：

```python
class Bad(Exception):
    def __init__(self, a, b):
        self.a, self.b = a, b
        super().__init__(f"{a}/{b}")        # ❌ args == ('1/2',)，只有一个元素
pickle.loads(pickle.dumps(Bad(1, 2)))
#=> TypeError: Bad.__init__() missing 1 required positional argument: 'b'

class Good(Exception):
    def __init__(self, a, b):
        self.a, self.b = a, b
        super().__init__(a, b)              # ✅ args == (1, 2)，可原样重建
```

想同时要漂亮的消息和可 pickle，就自己实现 `__reduce__`：

```python
class NotFound(Exception):
    def __init__(self, kind, id_):
        self.kind, self.id = kind, id_
        super().__init__(f"{kind} {id_} not found")
    def __reduce__(self):
        return (self.__class__, (self.kind, self.id))
```

另外 traceback 对象**不可 pickle**：跨进程只能拿到异常实例，远端栈由
`concurrent.futures` 字符串化后拼进消息里。见 [[concurrency/multiprocessing]]。

### 7.8 `add_note()`：给异常挂运行时上下文（3.11+）

以前想补充上下文只能重新包一层异常（改变了类型、破坏调用方的 `except` 契约）。
现在可以原样重抛并附注：

```python
try:
    process(payload)
except Exception as e:
    e.add_note(f"request_id={rid}")
    e.add_note(f"payload_keys={list(payload)}")
    raise                       # ✅ 异常类型不变，上层的 except 逻辑不受影响

e.__notes__      #=> ['request_id=abc', "payload_keys=['a', 'b']"]
```

traceback 里会紧跟在异常行后逐行打印：

```text
ValueError: boom
request_id=abc
payload_keys=['a', 'b']
```

`__notes__` 只在第一次 `add_note()` 时才创建，所以读它要用
`getattr(e, "__notes__", [])`。中间件、批处理里给异常补上下文，它比
`raise Wrapper(...) from e` 更合适——不改变异常契约。

### 7.9 异常组的程序化拆分

除了 `except*`，`ExceptionGroup` 还提供两个方法用于在普通代码里分流（例如网关层要把
「一半客户端错、一半服务端错」映射成不同的 HTTP 状态码）：

```python
eg = ExceptionGroup("multi", [ValueError("v1"), TypeError("t1"), ValueError("v2")])

match, rest = eg.split(ValueError)
match.exceptions   #=> (ValueError('v1'), ValueError('v2'))
rest.exceptions    #=> (TypeError('t1'),)

eg.subgroup(TypeError).exceptions   #=> (TypeError('t1'),)   ← 只要匹配的那半，丢掉 rest
```

两个反直觉之处（面试爱问）：

```python
# ① except* 的 as 变量拿到的永远是「子组」，不是单个异常
try:
    raise ValueError("solo")         # ← 甚至不是 ExceptionGroup
except* ValueError as g:
    type(g)          #=> <class 'ExceptionGroup'>   ← 裸异常也会被自动包一层
    g.exceptions     #=> (ValueError('solo'),)

# ② 没被任何 except* 分支匹配的部分，会重新打包成新的 ExceptionGroup 继续外抛
```

### 7.10 反模式清单（自查用）

| 反模式 | 后果 | 正确写法 |
|---|---|---|
| `except:` / `except BaseException:` | 吞掉 Ctrl+C、`SystemExit`、`CancelledError` | `except Exception:` |
| `except Exception: pass` | 故障静默，最难排查 | `log.exception(...)` 或 `suppress(具体类型)` |
| `finally: return` | 异常凭空消失 | `finally` 只做清理 |
| `log.error(f"{e}")` | 丢 traceback、丢懒格式化 | `log.exception("...")` |
| `raise NewError(str(e))` | 丢类型、丢栈 | `raise NewError(...) from e` 或 `e.add_note(...)` |
| 用 `assert` 做校验 | `-O` 下整句被移除 | `if not ...: raise` |
| `except Exception` 罩住 await | 顺手吞了 `CancelledError`，优雅关闭挂死 | 先 `except CancelledError: raise` |
| 存异常对象做统计 | traceback 钉住整个栈的局部变量 | 存 `str(e)` / `format_exc()` |
| `create_task` 不留引用 | 异常延迟到 GC 才暴露，甚至丢失 | `TaskGroup`，或引用 + done callback |
| 自定义异常不调 `super().__init__(*args)` | 跨进程 unpickle 失败 | 传全参数，或写 `__reduce__` |

> **面试落点**：被问「你们线上怎么做错误处理」时，能分层回答就赢了——
> ①**边界**：每个入口兜底 + `sys.excepthook` / `TaskGroup`，保证异常不会消失；
> ②**分类**：一个异常基类 + 可重试/不可重试拆分，再映射成 HTTP 状态码；
> ③**可观测**：`log.exception` 保 traceback，`add_note` 挂 request_id；
> ④**成本意识**：知道 3.11 零成本异常让快乐路径免费、异常路径仍贵 4~5 倍，
> 也知道存异常对象会连带钉住整个调用栈。

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
