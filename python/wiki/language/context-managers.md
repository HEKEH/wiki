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

# ③ 锁（标准库对象大多已经是 CM）
with threading.Lock(): ...
with asyncio.Lock(): ...          # async with
with multiprocessing.Pool() as p: ...

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
