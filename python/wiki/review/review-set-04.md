---
title: "复习题组 04 —— 异常体系一题（含完整解答）"
date: 2026-08-25
tags: [面试题, 复习, 异常, finally, 异常链, ExceptionGroup, add_note, pickle, BaseException]
sources: []
---

# 复习题组 04 —— 异常体系（控制流 → 链 → 组 → 边界）

对应 [[interview/roadmap]] 的 **Day 6**（异常体系），主题页是 [[language/exceptions]]。
**一道题串起 17 个输出 + 7 个追问 + 1 道加分题**，覆盖 try/else/finally 控制流、
`as` 变量的自动删除、异常链三件套、异常消息与 `args`、跨进程 pickle、`add_note`、
`ExceptionGroup` / `except*`、以及 `BaseException` 那一支的边界。
**每一行输出都在 CPython 3.11.9 上实测**。

用法：先只看「题目」自测，把 17 个输出全写下来（**会抛异常的行要写出异常类型**，
`print` 的副作用行顺序也算分），再对「参考答案」，最后读「解析」补机制。

| # | 主题 | 涉及页面 | 高频错点 |
|---|---|---|---|
| 1 | 异常体系（控制流 → 链 → 组 → 边界） | [[language/exceptions]] [[concurrency/asyncio-patterns]] [[concurrency/multiprocessing]] | `finally: return` 吞异常；`except*` 只接走一半；`except Exception` 以为兜住了一切 |

七段考点的分布：

| 考点 | 高频错点 |
|---|---|
| `finally` 里的 `return` | 异常被静默丢弃，连日志都没有 |
| `else` 的存在意义 | 以为只是「好看」，看不出它在**缩小保护范围** |
| `except X as e` 块末自动 `del` | 以为是作用域问题；不知道连 try 之前的同名值也会被删 |
| `__cause__` / `__context__` / `__suppress_context__` | 只知道 `from e`，说不清另外两个字段 |
| `str(e)` 与 `args` | 以为异常一定有消息；不知道 `KeyError` 的 `str` 是 `repr(key)` |
| 自定义异常的 pickle | `super().__init__(拼好的字符串)` → 跨进程重建直接炸 |
| `add_note` | 不知道 `__notes__` 要用 `getattr` 兜底 |
| `except*` 语义 | 以为 `as` 拿到的是单个异常；不知道未匹配部分会重新打包外抛 |
| `except Exception` 的边界 | 以为它兜住了一切 |

---

## 第 1 题 · 一段异常处理代码，十七个输出

### 题目

#### a. 控制流：`finally` / `else`

```python
def f1():
    try:
        raise ValueError("boom")
    finally:
        return 1

def trans(n):
    raise ValueError(f"unsupported {n}")

def f2(s):
    try:
        n = int(s)
        return trans(n)
    except ValueError:
        return "bad input"

def f3(s):
    try:
        n = int(s)
    except ValueError:
        return "bad input"
    else:
        return trans(n)

print(f1())          # (1)
print(f2("7"))       # (2)
print(f3("7"))       # (3)   ← 若抛出，写出异常类型与消息
```

#### b. `as` 变量的命运

```python
def f4():
    e = "上一轮的错误"
    try:
        raise ValueError("boom")
    except ValueError as e:
        pass
    return e

print(f4())          # (4)   ← 若抛出，写出异常类型
```

#### c. 异常链

```python
def load(raw):
    try:
        int(raw)
    except ValueError as e:
        raise KeyError("cfg") from e

def load2(raw):
    try:
        int(raw)
    except ValueError:
        raise KeyError("cfg")

for fn in (load, load2):
    try:
        fn("x")
    except KeyError as k:
        print(type(k.__cause__).__name__,
              type(k.__context__).__name__,
              k.__suppress_context__)      # (5) load / (6) load2
```

#### d. 消息、`args` 与跨进程

```python
print(repr(str(ValueError())), repr(str(KeyError("user_id"))))    # (7)

class Bad(Exception):
    def __init__(self, kind, id_):
        self.kind, self.id = kind, id_
        super().__init__(f"{kind} {id_} not found")

b = Bad("user", "42")
print(b.args)                                # (8)

import pickle
print(pickle.loads(pickle.dumps(b)))         # (9)   ← 若抛出，写出异常类型与消息
```

#### e. `add_note`（3.11+）

```python
import traceback

err = RuntimeError("boom")
print(getattr(err, "__notes__", "NO ATTR"))    # (10)
err.add_note("rid=abc")
try:
    raise err
except RuntimeError as x:
    print(traceback.format_exception_only(x))  # (11)
```

#### f. `ExceptionGroup` 与 `except*`（3.11+）

```python
try:
    raise ValueError("solo")                   # ← 注意：这不是一个组
except* ValueError as g:
    print(type(g).__name__, g.exceptions)      # (12)

try:
    try:
        raise ExceptionGroup("multi", [ValueError("v"), TypeError("t")])
    except* TypeError as g:
        print("handled:", g.exceptions)        # (13)
except BaseException as rest:
    print("escaped:", type(rest).__name__, rest.exceptions)   # (14)
```

#### g. `BaseException` 那一支

```python
import sys, asyncio

print(issubclass(asyncio.CancelledError, Exception))   # (15)

def worker():
    try:
        sys.exit(3)
    except Exception:
        return "caught"
    finally:
        print("cleanup")                       # (16)

try:
    print(worker())
except BaseException as x:
    print(type(x).__name__, x.code)            # (17)
```

### 追问

1. **(1)**：那个 `ValueError` 去哪了？`finally` 里除了 `return`，还有哪两个语句有同样的吞异常效果？
   为什么说这是最难查的一类 bug？
2. **(3)** 比 **(2)**「更好」在哪？把 `else` 去掉会带来什么真实事故？反过来，什么时候 `else` 是多余的？
3. **(4)**：`e` 为什么消失了，连 try 之前赋的值也没了？语言这么设计的动机是什么（提示：和**内存**有关）？
   要在 except 块外继续用异常对象，正确写法是什么？
4. **(5)(6)**：`__cause__` / `__context__` / `__suppress_context__` 三者的关系是什么？
   `raise X from None` 时三者各是什么值，traceback 会少打哪一段？对外 API 为什么常用 `from None`？
5. **(9)** 为什么炸？给出两种修法。为什么这个坑在 `ProcessPoolExecutor` 里才会变成线上事故？
   子进程的 traceback 能原样带回主进程吗？
6. **(12)(14)**：`except*` 的 `as` 变量为什么在 (12) 里也是个组？没被任何 `except*` 匹配的部分发生了什么？
   `except*` 能和普通 `except` 写在同一个 `try` 上吗？`ExceptionGroup("m", [KeyboardInterrupt()])` 会怎样？
7. **(15)(17)**：`except Exception` 罩不住哪几类异常？为什么 `CancelledError` 在 3.8 从
   `Exception` 改成继承 `BaseException`？这对重试逻辑意味着什么？

### 加分题

```python
assert (user.is_admin, "must be admin")
```

这一行有**两个独立**的 bug，分别是什么？正确写法？

---

## 参考答案

```text
(1)  1
(2)  bad input
(3)  ValueError unsupported 7
(4)  UnboundLocalError cannot access local variable 'e' where it is not associated with a value
(5)  ValueError ValueError True
(6)  NoneType ValueError False
(7)  '' "'user_id'"
(8)  ('user 42 not found',)
(9)  TypeError Bad.__init__() missing 1 required positional argument: 'id_'
(10) NO ATTR
(11) ['RuntimeError: boom\n', 'rid=abc\n']
(12) ExceptionGroup (ValueError('solo'),)
(13) handled: (TypeError('t'),)
(14) escaped: ExceptionGroup (ValueError('v'),)
(15) False
(16) cleanup
(17) SystemExit 3
```

逐条要点：

- **(1)** `finally` 里的 `return` 在异常向外传播的**途中**接管了函数出口，异常被**静默丢弃**，
  函数正常返回 `1`。
- **(2)** `trans()` 抛的 `ValueError` **也**落进了同一个 `except ValueError`——这正是
  「try 保护范围过大」的事故：想接的是 `int(s)` 的解析失败，实际把业务逻辑的错也吞了，
  还伪装成 `"bad input"`。
- **(3)** `else` 把 `trans(n)` 移出保护范围，它的 `ValueError` **原样外抛**。这才是想要的行为。
- **(4)** 不是 `NameError` 而是 **`UnboundLocalError`**：`e` 在函数里是局部名，
  `except ... as e` 块结束时语言隐式执行 `del e`，把**这个名字**删了（连 try 之前赋的字符串一起）。
- **(5)** `from e` → `__cause__` 是 `ValueError`，`__context__` 同时也被自动记为 `ValueError`，
  `__suppress_context__` 被置 `True`。
- **(6)** 不写 `from` → `__cause__` 是 `None`（打印类型名就是 `NoneType`），
  `__context__` 仍自动记录，`__suppress_context__` 为 `False`。
- **(7)** 异常消息不是必填项：`str(ValueError())` 是**空串**；
  `KeyError` 的 `__str__` 是 `repr(key)`，所以带引号。
- **(8)** `args` 只有一个元素——因为 `super().__init__()` 收到的是拼好的字符串。
- **(9)** `BaseException.__reduce__` 用 `type(e)(*e.args)` 重建，`args` 只有一个元素，
  少一个必填参数 → `TypeError`。
- **(10)** `__notes__` **只在第一次 `add_note()` 时才创建**，之前属性根本不存在。
- **(11)** note 作为**独立一行**紧跟在异常行之后，由 `format_exception_only` 一并产出。
- **(12)** `except*` 的 `as` 变量**永远是一个组**：裸 `ValueError` 会被自动包进
  `ExceptionGroup` 再交给分支。
- **(13)(14)** `except*` 只「筛走」匹配的那一半；剩下的 `ValueError` 被**重新打包成一个新的
  `ExceptionGroup`** 继续向外抛。
- **(15)** `asyncio.CancelledError` 继承 `BaseException`，**不是** `Exception` 的子类。
- **(16)(17)** `sys.exit(3)` 抛 `SystemExit`（`BaseException` 一支），`except Exception`
  **接不住**；但 `finally` 照常执行 → 先打印 `cleanup`，异常再向外传播。

---

## 解析（追问答案）

### 1. `finally` 吞异常

异常在传播途中要先跑完 `finally`。如果 `finally` 里出现**任何一个改变控制流的语句**，
原来的传播就被作废：

| 语句 | 效果 |
|---|---|
| `return` | 异常丢弃，函数正常返回 |
| `break` | 异常丢弃，跳出循环 |
| `continue` | 异常丢弃，进入下一轮 |

```python
def loop():
    for i in range(3):
        try:
            raise ValueError("x")
        finally:
            break          # ❌ 异常凭空消失
    return "done"

loop()      #=> 'done'
```

之所以最难查：**没有异常、没有日志、没有 traceback**，只有一个「看起来成功了」的返回值。
监控上表现为「成功率 100%，但数据不对」。规矩很简单：**`finally` 里只做清理，不放控制流语句**。
（真要在清理失败时改变结果，显式 `raise` 新异常，别用 `return`。）

### 2. `else` 的价值 = 缩小 try 的保护范围

(2) 的 `f2` 是个**真实事故模板**：`except ValueError` 本意是接 `int(s)` 的解析失败，
结果把下游 `trans()` 抛的同类异常一起吞了，并且返回了一个**语义错误**的 `"bad input"`——
调用方会以为是用户输入问题去查前端，而 bug 在业务逻辑里。

`else` 的判据：**try 块里只放「你正在保护的那一个调用」**，其余全放 `else`。

什么时候 `else` 多余：`except` 分支全都 `return` / `raise`（不会往下走）时，
`else` 和「跟在 try 后面」等价，写不写只是风格；一旦有分支会**穿过**去执行后续代码，
`else` 就是必需的。

### 3. `as` 变量为什么被删

Python 在 `except X as e:` 块末尾插入了等价于下面的代码：

```python
except X as e:
    try:
        <块体>
    finally:
        e = None
        del e          # ← 语言隐式生成
```

**动机是内存**：`e.__traceback__` → frame → 该帧的**全部局部变量**，形成强引用
（而且常是引用循环，只能等分代 GC）。如果 `e` 留在作用域里，整个调用栈的局部变量
都跟着活着：

```python
class Big:
    def __del__(self): print("Big 被回收")

def boom():
    big = Big()
    raise ValueError("boom")

saved = None
try:
    boom()
except ValueError as ex:
    saved = ex                 # ❌ 把 big 一起钉住
print("已离开 except 块")       # ← 此时 Big 还活着，什么都没打印
saved = None; gc.collect()     #=> Big 被回收   ← 到这里才析构
```

所以 (4) 报的是 `UnboundLocalError` 而不是 `NameError`：编译期 `e` 已经是函数局部名，
运行期那个槽位被 `del` 清空了。**要在块外用，先拷到别的名字**：

```python
err = None
try:
    ...
except ValueError as e:
    err = e            # ✅ 显式续命（自己承担钉住栈的代价）
```

统计 / 上报场景更推荐只留 `str(e)` 或 `traceback.format_exc()` 字符串；
要记日志就用 `log.exception()`，它在块内读 `sys.exc_info()`，根本不需要你持有 `e`。

### 4. 异常链三件套

| 字段 | 谁设置 | 含义 |
|---|---|---|
| `__context__` | **解释器自动** | 「抛出新异常时，正在处理的那个异常」 |
| `__cause__` | `raise ... from X` | 「我**声明**这是根因」 |
| `__suppress_context__` | 写了 `from`（含 `from None`）就置 `True` | 打不打印 `__context__` 那一段 |

三种写法的 traceback 措辞：

```python
raise New() from e     # __cause__=e, suppress=True   → "The above exception was the direct cause of..."
raise New()            # __cause__=None, suppress=False → "During handling of the above exception, another occurred"
raise New() from None  # __cause__=None, __context__ 仍在, suppress=True → 只打印 New
```

关键点：**`from None` 并不擦掉 `__context__`**，只是把它标记为不打印——调试时仍可
`e.__context__` 手动取回。

对外 API 常用 `from None` 的理由：上游细节属于**实现细节**（用了哪个 HTTP 库、
哪个 JSON 解析器），泄漏出去既是噪音也是攻击面，对外只给一个稳定的 `ConfigError`。
但**内部服务之间别这么干**——把根因藏了，值班的人要多花一小时。

### 5. 异常的 pickle

`BaseException.__reduce__` 的默认实现是 `(type(e), e.args)`，重建时执行
`type(e)(*e.args)`。所以**「`args` 必须能原样喂回 `__init__`」**是所有自定义异常的隐含契约。
`super().__init__(f"...")` 把多个字段压成一个字符串，契约就断了。

两种修法：

```python
# 修法一：把原始参数交给 super，消息交给 __str__
class Good1(Exception):
    def __init__(self, kind, id_):
        self.kind, self.id = kind, id_
        super().__init__(kind, id_)          # ✅ args == ('user', '42')
    def __str__(self):
        return f"{self.kind} {self.id} not found"

# 修法二：保留漂亮消息，自己写 __reduce__
class Good2(Exception):
    def __init__(self, kind, id_):
        self.kind, self.id = kind, id_
        super().__init__(f"{kind} {id_} not found")
    def __reduce__(self):
        return (self.__class__, (self.kind, self.id))     # ✅
```

为什么在 `ProcessPoolExecutor` 里才成事故：单进程内谁也不会去 pickle 一个异常，
本地测试全绿；一上多进程，**子进程真正的业务异常在回传时被一个 `TypeError` 顶掉**，
主进程看到的是「反序列化失败」而不是根因——排查方向从一开始就是错的。
（`multiprocessing`、Celery、以及任何把异常跨进程 / 跨网络传的框架同理。）

traceback 对象**不可 pickle**。`concurrent.futures` 的做法是在子进程侧把远端栈
**字符串化**后拼进异常消息里，所以主进程能看到远端栈的文字，但拿不到可编程访问的
`__traceback__`。

### 6. `except*` 的四个反直觉点

**① `as` 变量永远是组。** 即使抛的是裸异常，也会被自动包一层（见 (12)）。
所以处理逻辑一律写成遍历 `g.exceptions`，别假设拿到的是单个异常。

**② 未匹配的部分重新打包外抛**（见 (14)）。这是和 `except` 最大的差别：
`except` 是「要么全接、要么全不接」，`except*` 是**部分接管**——所以外层永远要留一道
兜底，否则「一半被处理、一半继续飞」会很意外。

**③ 一个组可能触发多个分支。** 组里既有 `ValueError` 又有 `ConnectionError` 时，
两个 `except*` 分支**都会执行**（普通 `except` 只命中第一个匹配分支）。

**④ 语法层面的两条硬限制**（实测报错原文）：

```python
# 不能混用
try: ...
except* ValueError: ...
except TypeError: ...
#=> SyntaxError: cannot have both 'except' and 'except*' on the same 'try'

# except* 块里不能有 break / continue / return
#=> SyntaxError: 'break', 'continue' and 'return' cannot appear in an except* block
```

（第二条的原因正是 ③：同一个 `try` 可能执行多个 `except*` 分支，`return` 该听谁的没有定义。）

关于 `ExceptionGroup("m", [KeyboardInterrupt()])`：

```python
ExceptionGroup("m", [KeyboardInterrupt()])
#=> TypeError: Cannot nest BaseExceptions in an ExceptionGroup

type(BaseExceptionGroup("m", [KeyboardInterrupt()]))   #=> BaseExceptionGroup
type(BaseExceptionGroup("m", [ValueError()]))          #=> ExceptionGroup  ← 自动降级
```

`ExceptionGroup` 只装 `Exception`，装 `BaseException` 必须用 `BaseExceptionGroup`；
而 `BaseExceptionGroup` 在内容全是 `Exception` 时会**自动返回 `ExceptionGroup`**。
这层设计是为了不破坏 `except Exception` 的老语义——一个装着 `KeyboardInterrupt`
的组，绝不能被 `except* Exception` 接住。

程序化拆分用 `split` / `subgroup`，**没有匹配项时返回 `None` 而不是空组**：

```python
eg = ExceptionGroup("m", [ValueError("v")])
eg.split(TypeError)      #=> (None, ExceptionGroup('m', [ValueError('v')]))
eg.subgroup(TypeError)   #=> None
```

### 7. `except Exception` 罩不住什么

`BaseException` 直属的几个子类，**全部**逃出 `except Exception`：

| 异常 | 来源 | 为什么必须放行 |
|---|---|---|
| `SystemExit` | `sys.exit()` | 吞了它 = 程序退不掉 |
| `KeyboardInterrupt` | Ctrl+C | 吞了它 = Ctrl+C 失效 |
| `GeneratorExit` | 生成器 `close()` | 吞了它 = 生成器无法清理 |
| `asyncio.CancelledError` | task 取消 / 超时 | 吞了它 = 优雅关闭挂死 |

`CancelledError` 在 **3.8** 从 `Exception` 改到 `BaseException` 正是因为第四条：
在 3.7 及更早，随便一个 `except Exception:` 兜底就会把取消信号吃掉，
`task.cancel()` 之后任务照常跑完，`asyncio.wait_for` 的超时形同虚设。

对重试逻辑的直接后果——**取消不是故障，绝不能重试**：

```python
# ❌ 兜底重试把取消信号也重试了，任务永远关不掉
try:
    return await call()
except Exception:
    return await call()

# ✅ 先放行取消，再谈重试
try:
    return await call()
except asyncio.CancelledError:
    raise                                  # 原样重抛，一个字都别改
except (ConnectionError, TimeoutError):
    return await call()
```

顺带：(16) 说明 **`finally` 对 `BaseException` 一样生效**——清理逻辑放 `finally`
（或上下文管理器）就能覆盖 Ctrl+C 和取消，放在 `except Exception` 里则不能。
这也是「清理用 `finally`，不用 `except`」的硬理由。

### 加分题：`assert (user.is_admin, "must be admin")`

**bug ①**：括号让它变成一个**非空元组**，真值恒为 `True`，断言**永远通过**。
CPython 会给 `SyntaxWarning: assertion is always true, perhaps remove parentheses?`
（实测 3.11.9 已有此警告），但警告默认不打眼，很容易漏。

**bug ②**：更严重的是——**安全检查根本不该用 `assert`**。`python -O` 会在编译期
直接移除整条 `assert` 语句（连字节码都不生成），上线加个 `-O`，权限校验就凭空消失了。

```python
if not user.is_admin:                 # ✅
    raise PermissionError("must be admin")

assert self._lock.locked()            # ✅ assert 只用于「代码有 bug 才会失败」的内部不变量
```

---

## 面试落点

> **面试落点**：被问「你们线上怎么做错误处理」时，分四层回答就赢了——
> ①**不丢**：清理只放 `finally` / 上下文管理器，绝不在 `finally` 里 `return`；
> try 只罩住「正在保护的那一个调用」，其余放 `else`；每个入口一道兜底屏障
> （`sys.excepthook` / `threading.excepthook` / `TaskGroup`）。
> ②**不歪**：转换异常一律 `raise X from e` 保住 `__cause__`，只在对外 API 边界用
> `from None`；补上下文优先 `add_note()`，因为它不改变异常类型、不破坏调用方的 `except` 契约。
> ③**不越界**：`except Exception` 是兜底的上限，`SystemExit` / `KeyboardInterrupt` /
> `GeneratorExit` / `CancelledError` 必须放行；异步代码里先写 `except CancelledError: raise`
> 再谈重试。
> ④**跨边界**：自定义异常要满足 `type(e)(*e.args)` 能重建（否则 `ProcessPoolExecutor`
> 会用一个 `TypeError` 顶掉真正的根因），traceback 本身不可 pickle。

> **面试落点**：`except*` 的一句话总结是「**部分接管**」——`as` 拿到的永远是子组，
> 一个组可能触发多个分支，未匹配的部分重新打包继续外抛。能补一句
> 「`ExceptionGroup` 不许装 `BaseException`，就是为了让 `except* Exception` 接不住
> `KeyboardInterrupt`」，说明是真读过设计动机而不是背 API。

## 批改记录

实际作答 **12 / 17**。全对 10 条、半对 4 条、全错 3 条。

| # | 判定 | 错在哪 |
|---|---|---|
| (1) | ⚠️ | 答「没有抛出」——**机制看懂了**（异常被 `finally` 吞掉），但题目问的是打印什么，漏了返回值 `1` |
| (2) | ✅ | |
| (3) | ✅ | 类型和消息都对 |
| (4) | ❌ | 答 `ValueError("boom")`：完全没意识到 `except ... as e` 块末有隐式 `del e`。正解是 **`UnboundLocalError`** |
| (5) | ❌ | `__suppress_context__` 答反了。`from e` 会把它置 **`True`** |
| (6) | ❌ | 三个字段错了两个，且是**系统性反转**：以为「不写 `from` → `__cause__` 有值、`__context__` 是 None」，实际正好相反 |
| (7) | ⚠️ | `''` 对；`KeyError` 那半答成 `"user_id"`，漏了**内层引号**——本题考的就是 `KeyError.__str__` 是 `repr(key)` |
| (8) | ⚠️ | 关键点（**只有一个元素**）答对了；但 `args` 是 **tuple** 不是 list，且消息里 `user` 手误成 `use` |
| (9) | ✅ | 认出了 `args` 与 `__init__` 签名不匹配 |
| (10) | ✅ | 知道 `__notes__` 首次 `add_note` 才创建 |
| (11) | ⚠️ | 形状对（异常行 + note 行两条）；但漏了 `format_exception_only` 返回的是 **list[str]**、每条带 `RuntimeError: ` 前缀和结尾 `
` |
| (12) | ✅ | 裸异常被自动包组，答对 |
| (13) | ✅ | |
| (14) | ✅ | 未匹配部分重新打包外抛，答对——本题最能体现理解深度的一问 |
| (15) | ✅ | |
| (16) | ✅ | |
| (17) | ✅ | |

### 错法归类

**① 异常链三件套整体反转（(5)(6)，唯一的知识性硬伤）。** 这不是记混，是**因果方向装反了**：

```text
__context__ = 解释器【自动】记录的「我抛新异常时，手上正在处理的那个异常」——永远会有
__cause__   = 你【手动】用 from 声明的根因——不写 from 就是 None
__suppress_context__ = 「写没写过 from」的标记——写了（含 from None）就是 True
```

一句话记法：**`context` 是自动的、总在；`cause` 是手动的、要你写；`suppress` 只是问你「写了 `from` 吗」。**
实测三种写法的完整取值（已补进 [[language/exceptions]] §4）：

| 写法 | `__cause__` | `__context__` | `__suppress_context__` | traceback 中间那句 |
|---|---|---|---|---|
| `raise K() from e` | `ValueError` | `ValueError` | `True` | "...was the **direct cause** of..." |
| `raise K()` | `None` | `ValueError` | `False` | "**During handling** of the above..." |
| `raise K() from None` | `None` | `ValueError` ← **仍在** | `True` | 无（那一整段不打印） |

注意第三行：`from None` **没有擦掉** `__context__`，只是标记为不打印，调试时仍可 `k.__context__` 取回。

**② `as` 变量的自动 `del` 是盲区（(4)）。** 这条不是冷知识，它是**内存问题的语言级对策**：
`e.__traceback__` → frame → 该帧全部局部变量，留住 `e` 就钉住整个调用栈。
顺带记住两种作用域报错不同（实测）：

```python
# 函数内 → e 是局部名，槽位被清空
UnboundLocalError: cannot access local variable 'e' where it is not associated with a value
# 模块级 → 名字从 globals 里删掉
NameError: name 'e' is not defined
```

**③ 「答语义，不答字面输出」——和 [[review/review-set-03]] 批改记录里同一个习惯的复发。**
(1)(7)(8)(11) 四条半对全属此类：机制都想对了，丢分在**没把打印出来的那串字符原样写下来**。
(1) 说「没有抛出」而不写 `1`、(7) 漏内层引号、(11) 只写两条消息的大意。
这类题的判分标准是「你能不能预测 stdout 上**逐字符**出现什么」，机制对但形式错，
在真实的 debug 场景里就等于「以为自己知道，实际对不上日志」。

**④ tuple / list 混写（(8)(12)(13)(14)）。** 四处把元组写成了 `[...]`。这次没扣分（内容都对），
但异常体系里这两处**恰好都是 tuple**：`e.args` 是 tuple、`eg.exceptions` 也是 tuple（实测）。
`args` 是 tuple 这一点还有实际后果——`type(e)(*e.args)` 的解包重建正建立在它是 tuple 上（见 (9)）。

### 掌握扎实的

(9)(14)(15)(16)(17) 这一串答得干净：**跨进程 pickle 契约**、**`except*` 的部分接管语义**、
**`BaseException` 一支的边界**——恰好是三个最"工程"、最容易在真实事故里遇到的点，
也是本题里最难的三问。(12)「裸异常也被包组」能答对，说明 `except*` 不是背 API 背来的。

真正的缺口只有两块：**异常链三件套**（必考，且是「转换异常」这个日常操作的正确性基础）
和 **`as` 变量的生命周期**（背后是 traceback 钉住栈的内存问题）。两块都已回补进主题页。

## 一条主线：异常的「消失」有六种方式

本题 17 个输出里，有一半在回答同一个问题——**异常是怎么悄悄不见的**：

| 输出 | 消失方式 | 症状 |
|---|---|---|
| (1) | `finally` 里的 `return` / `break` / `continue` | 无异常、无日志，返回值看起来正常 |
| (2) | try 保护范围过大，被同类型的 `except` 误捕 | 报了错，但**报的是错的原因** |
| (4) | 想留住 `e` 却发现它被自动 `del` | 块外拿不到对象（这条是语言在**帮**你省内存） |
| (6) | 不写 `from e`，只落在 `__context__` 里 | traceback 还在，但因果关系降级成「凑巧同时发生」 |
| (9) | 跨进程重建失败，真异常被 `TypeError` 顶掉 | 看到的是序列化错误，根因彻底丢失 |
| (14) | `except*` 只接走一半 | 另一半重新打包继续飞，外层没兜底就炸在别处 |
| (17) | `except Exception` 罩不住 `BaseException` 一支 | 反过来：**该逃的逃掉了**，这是对的 |

答题时不要停在「我 try 起来了」，要说清**异常从抛出点到日志之间会经过哪些环节、
每个环节可能把它弄丢**。这和 [[review/review-set-03]] 的「协议是查得到而不是写得对」
是同一种思路：都在追**解释器的实际执行路径**，而不是代码「看起来的意图」。

配套的工程结论在 [[language/exceptions]] §7.2（每个入口一道兜底屏障）与
§7.10（反模式清单）。

## 相关

- [[interview/roadmap]] —— 本组题对应 Day 6（异常体系那一半）
- [[review/review-set-01]] —— 语言核心五题（Day 1–4）
- [[review/review-set-02]] —— 类、MRO 与 ABC 两题（Day 5）
- [[review/review-set-03]] —— 数据模型 dunder 全景（Day 6 的另一半）
- [[language/exceptions]] —— 主题页
- [[language/context-managers]] —— `finally` 的替代品：`__exit__` 与异常传播
- [[concurrency/asyncio-patterns]] —— TaskGroup / CancelledError / ExceptionGroup
- [[concurrency/multiprocessing]] —— 异常跨进程 pickle
- [[interview/traps]] —— `finally: return` 等陷阱题
