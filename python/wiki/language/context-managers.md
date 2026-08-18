---
title: "上下文管理器（with 语句）"
date: 2026-08-07
tags: [上下文管理器, with, contextlib, 资源管理, RAII]
sources: ["cpython-doc/datamodel.rst", "python-cheatsheet.md"]
---

# 上下文管理器（Context Manager）

`with` 解决的是**"无论如何都要执行的收尾动作"**——关文件、放锁、回滚事务、还连接。
前端没有直接对应物（最接近的是 `try/finally` 和 TS 5.2 的 `using` 声明）。

## 1. 协议

```python
class Managed:
    def __enter__(self):
        print("acquire")
        return self                # 返回值绑定给 as 后面的名字

    def __exit__(self, exc_type, exc_val, exc_tb):
        print("release")
        return False               # 返回真值 = 吞掉异常；返回假值/None = 异常继续传播

with Managed() as m:
    ...
```

`with A() as a:` 的等价展开：

```python
mgr = A()
a = type(mgr).__enter__(mgr)       # 注意在类型上查找
try:
    body
except:
    if not type(mgr).__exit__(mgr, *sys.exc_info()):
        raise
else:
    type(mgr).__exit__(mgr, None, None, None)
```

> **面试落点**：`__exit__` **返回 True 会吞掉异常**——这是最容易被忽略的细节。
> 不小心 `return True` 会让 bug 静默消失。默认写 `return False` 或干脆不写 return。

## 2. `@contextmanager`：用生成器写

```python
from contextlib import contextmanager

@contextmanager
def transaction(conn):
    tx = conn.begin()
    try:
        yield tx               # ← yield 之前 = __enter__，之后 = __exit__
    except Exception:
        tx.rollback()
        raise
    else:
        tx.commit()
    finally:
        conn.close()

with transaction(db) as tx:
    tx.execute(...)
```

规则：

- **必须恰好 yield 一次**（0 次或 2 次都会 RuntimeError）。
- **yield 必须包在 try 里**，否则 body 抛异常时清理代码不执行。
- body 里的异常会通过 `gen.throw()` 在 yield 处抛出。
- 想吞掉异常就在 `except` 里 `return`（不 re-raise）。

## 3. `contextlib` 工具箱

```python
from contextlib import (
    contextmanager, asynccontextmanager, closing, suppress,
    redirect_stdout, ExitStack, AsyncExitStack, nullcontext,
)

# 给只有 close() 但没有 with 支持的对象套上 with
with closing(urlopen(url)) as page: ...

# 忽略特定异常（比 try/except/pass 更明确）
with suppress(FileNotFoundError):
    os.remove(path)

# 重定向标准输出
buf = io.StringIO()
with redirect_stdout(buf):
    print("captured")

# 条件性上下文：不需要时用 nullcontext 占位，避免写两份代码
cm = open(path) if path else nullcontext(sys.stdin)
with cm as f: ...

# 动态数量的上下文管理器
with ExitStack() as stack:
    files = [stack.enter_context(open(p)) for p in paths]   # 全部会被正确关闭
    stack.callback(cleanup)                                  # 注册任意收尾回调
```

`ExitStack` 是写库时的利器——**数量在运行时才确定**的资源用它管理，比嵌套 `with` 灵活得多。

## 4. 异步上下文管理器

```python
class AsyncConn:
    async def __aenter__(self):
        self.conn = await connect()
        return self.conn
    async def __aexit__(self, et, ev, tb):
        await self.conn.close()

async with AsyncConn() as conn: ...

# 生成器版
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app):
    pool = await create_pool()          # 启动
    yield {"pool": pool}
    await pool.close()                  # 关闭
```

`@asynccontextmanager` 正是 **FastAPI lifespan** 的写法，见 [[web/fastapi-architecture]]。

## 5. 常见实战模式

```python
# ① 计时
@contextmanager
def timed(label):
    t0 = time.perf_counter()
    try:
        yield
    finally:
        log.info("%s took %.3fs", label, time.perf_counter() - t0)

# ② 临时修改全局状态并保证还原
@contextmanager
def setenv(**kw):
    old = {k: os.environ.get(k) for k in kw}
    os.environ.update({k: str(v) for k, v in kw.items()})
    try:
        yield
    finally:
        for k, v in old.items():
            if v is None:
                os.environ.pop(k, None)
            else:
                os.environ[k] = v

# ③ 锁与池（标准库对象大多已经是 CM，但三者的 __enter__/__exit__ 语义各不相同）
lock = threading.Lock()           # 锁必须是共享的同一个对象，不能在 with 里现场 new
with lock:                        # __enter__ = acquire()，__exit__ = release()（异常也放锁）
    ...

# ④ 多个 CM 一行（3.10+ 支持括号换行）
with (
    open("a") as fa,
    open("b") as fb,
):
    ...

# ⑤ 测试里断言异常
with pytest.raises(ValueError, match="bad"):
    parse("x")
```

### 5.1 标准库同步原语的 CM 语义各不相同（高频错点）

上面 ③ 只写了 `threading.Lock` 一种，因为**另外两个各有各的坑**，不能照着套。
以下输出均在 CPython 3.11.9 实测（3.9–3.13 行为一致）。

**① `threading.Lock`：`__enter__` 返回的是 `True`，不是锁本身**

```python
lk = threading.Lock()
with lk as x:
    print(repr(x))                #=> True     ← 不是 <locked _thread.lock object>
```

所以锁一律写 `with lk:`，写 `with lk as l:` 拿到的 `l` 是个布尔值，后续 `l.release()` 会 `AttributeError`。

**② `asyncio.Lock`：只有 `__aenter__`，用 `with` 直接抛 TypeError**

```python
# ❌ 错误写法
with asyncio.Lock():
    ...
#=> TypeError: 'Lock' object does not support the context manager protocol

# ✅ 正确写法
lock = asyncio.Lock()
async def worker():
    async with lock:
        await do_something()
```

`asyncio.Lock` 从来没有 `__enter__`：3.7 起弃用了那个只会抛 `RuntimeError` 提示用 `async with` 的版本，
3.9 起彻底移除，于是报错从「友好的 RuntimeError」退化成了「协议缺失的 TypeError」。
语义与线程锁一样是进入 acquire、离开 release，区别在于等锁时是 `await` 挂起、交还事件循环，
而不是阻塞整个线程。参见 [[concurrency/asyncio-fundamentals]]。

**③ `multiprocessing.Pool`：`__exit__` 是 `terminate()`，不是 `close()` + `join()`**

```python
# 源码就这一行
def __exit__(self, exc_type, exc_val, exc_tb):
    self.terminate()              # 立刻杀掉 worker，不等未完成的任务
```

于是块内用异步 API 提交、块外取结果，任务会被**静默杀掉**：

```python
import multiprocessing as mp, time

def slow(n):
    time.sleep(1)
    return n * n

# 以下都要放在 if __name__ == "__main__": 里（spawn 平台的硬性要求）

# ❌ 错误写法
with mp.Pool(2) as p:
    r = p.map_async(slow, range(4))   # 不阻塞，提交完就离开 with
r.get(timeout=5)                      #=> TimeoutError    ← 任务已随 terminate 消失

# ✅ 正确写法：块内用阻塞式 API，或在块内就把结果取出来
with mp.Pool(2) as p:
    print(p.map(slow, range(4)))      #=> [0, 1, 4, 9]
with mp.Pool(2) as p:
    r = p.map_async(slow, range(4))
    print(r.get())                    #=> [0, 1, 4, 9]    ← get() 在块内
```

对照 `concurrent.futures` 的池，`Executor.__exit__` 是 `shutdown(wait=True)`——**语义正好相反**，
离开 `with` 会等所有已提交任务跑完：

```python
with ProcessPoolExecutor(2) as ex:
    futs = [ex.submit(slow, i) for i in range(4)]   # 不阻塞
# 这里已经等了 ~2s（4 个任务 / 2 个 worker）
[f.result() for f in futs]            #=> [0, 1, 4, 9]
```

> **面试落点**：`with` 只保证"离开时调用 `__exit__`"，不保证 `__exit__` 做的是"优雅收尾"。
> `multiprocessing.Pool` 收的是 `terminate`，`ProcessPoolExecutor` 收的是 `shutdown(wait=True)`——
> 同样一段 `with` 代码，前者丢任务后者不丢。用不熟的 CM 之前先看一眼它的 `__exit__`。

详见 [[concurrency/multiprocessing]]。

## 6. 为什么不用 `try/finally` 就好？

能，但 `with` 的价值是**把"正确的收尾逻辑"封装成可复用的对象**，调用方不可能忘记。
对比：

```python
# ❌ 每个调用点都要重复写，且容易漏
f = open(p)
try:
    data = f.read()
finally:
    f.close()

# ✅
with open(p) as f:
    data = f.read()
```

另外，CPython 的引用计数会让 `open(p).read()` 里的文件"碰巧"被及时关闭，
**但这是实现细节**——PyPy 用的是纯 GC，文件会延迟关闭直到耗尽 fd。
所以 `with` 不是风格问题，是**跨实现的正确性问题**。

> **面试落点**：「为什么一定要用 with 打开文件？」标准答案不是"优雅"，而是
> **不依赖 CPython 引用计数的即时析构，在异常路径下也保证释放**。

## 相关

- [[language/data-model]] —— `__enter__`/`__exit__`/`__aenter__`/`__aexit__` 协议
- [[language/iterators-generators]] —— `@contextmanager` 依赖生成器的 throw/close
- [[language/decorators]] —— ContextDecorator：同一对象既是 CM 又是装饰器
- [[language/exceptions]] —— `__exit__` 与异常传播
- [[web/fastapi-architecture]] —— lifespan 与依赖的 yield
- [[internals/garbage-collection]] —— 为什么不能依赖 `__del__` 做清理
