---
title: "asyncio 实战模式"
date: 2026-08-07
tags: [asyncio, TaskGroup, gather, 超时, 取消, 信号量, contextvars]
sources: ["fastapi-doc/async.md"]
---

# asyncio 实战模式

原理见 [[concurrency/asyncio-fundamentals]]。这一页是**写生产代码时真正要用的东西**。

## 1. 并发聚合

### `gather` —— 经典写法

```python
results = await asyncio.gather(fetch(1), fetch(2), fetch(3))
# 保持输入顺序返回结果

# 容错：不因一个失败而全盘皆输
results = await asyncio.gather(*coros, return_exceptions=True)
for r in results:
    if isinstance(r, Exception):
        log.error("失败: %s", r)
```

> ⚠️ `gather` 默认行为：**任一任务抛异常，`gather` 立即抛出该异常，
> 但其余任务仍在后台继续运行**（不会被取消）。这是常见的资源泄漏来源。

### `TaskGroup` —— 3.11+ 的结构化并发 ★推荐

```python
async def main():
    async with asyncio.TaskGroup() as tg:
        t1 = tg.create_task(fetch(1))
        t2 = tg.create_task(fetch(2))
    # 退出 with 时：等待全部完成
    # 任一失败 → 【自动取消其余任务】，并抛出 ExceptionGroup
    print(t1.result(), t2.result())

# 配合 except* 处理
try:
    await main()
except* ConnectionError as eg:
    ...
except* ValueError as eg:
    ...
```

| | `gather` | `TaskGroup` |
|---|---|---|
| 失败时其余任务 | **继续跑**（泄漏风险） | **自动取消** ✅ |
| 异常形式 | 第一个异常 | `ExceptionGroup`（全部） |
| 结果获取 | 返回值列表 | `task.result()` |
| 版本 | 全版本 | 3.11+ |

> **面试落点**：能说出「优先用 `TaskGroup`，因为它是结构化并发——保证子任务不会
> 在父作用域结束后还活着，失败时自动取消兄弟任务」——这是 2026 年的正确答案。
> `gather` 的"孤儿任务"问题是真实的生产事故来源。

### 竞速与部分完成

```python
done, pending = await asyncio.wait(
    tasks, timeout=5, return_when=asyncio.FIRST_COMPLETED,
)
for t in pending:
    t.cancel()

# 谁先完成先处理（流式）
for coro in asyncio.as_completed(tasks):
    result = await coro
    handle(result)
```

> ⚠️ `asyncio.wait` 接受的是 **Task/Future**，3.12 起不再接受裸协程。
> 记得先 `create_task`。

## 2. 超时

```python
# ✅ 3.11+ 推荐：上下文管理器形式，可包住任意代码块
async with asyncio.timeout(5):
    await step1()
    await step2()          # 整段代码 5 秒内必须完成
# 超时 → 抛 TimeoutError

async with asyncio.timeout_at(loop.time() + 5): ...

# 兼容老版本
result = await asyncio.wait_for(coro(), timeout=5)     # 超时会取消该协程

# 可重新调整的超时
async with asyncio.timeout(5) as cm:
    await part1()
    cm.reschedule(loop.time() + 10)     # 延长
    await part2()
```

超时的实现就是"到点了给这个 Task 发 cancel"，所以**被超时的协程会收到 `CancelledError`**，
它的 `finally` 会执行。

## 3. 取消的正确处理

```python
async def worker():
    try:
        while True:
            await do_step()
    except asyncio.CancelledError:
        await flush_buffer()          # 清理
        raise                         # ✅ 必须重抛！
    finally:
        await close_conn()            # 也会执行

# 屏蔽取消（谨慎使用）：确保关键区段不被打断
await asyncio.shield(critical_operation())

# 3.11+：临时禁止取消（比 shield 更精确）
task = asyncio.current_task()
with task.uncancel():   # 3.11+ 的取消计数机制
    ...
```

**取消的三个铁律**：

1. `CancelledError` 继承 `BaseException`，`except Exception` **不会**误吞它。
2. 捕获后**必须重抛**，否则会破坏 TaskGroup/timeout 的语义（3.11+ 会报
   "Cancel scope was cancelled but exception was swallowed"类问题）。
3. `finally` 里的 `await` 也可能被再次取消——需要极致可靠时用 `asyncio.shield`。

## 4. 限流：Semaphore

```python
sem = asyncio.Semaphore(10)          # 最多 10 个并发请求

async def fetch_limited(url):
    async with sem:
        return await client.get(url)

results = await asyncio.gather(*(fetch_limited(u) for u in urls))   # 1 万个 URL 也安全
```

> **面试落点**：「你要抓 10 万个 URL 怎么做？」
> 标准答案：**asyncio + `Semaphore` 限流 + `TaskGroup` 管理 + 分批处理 + 重试与超时**。
> 不加限流会瞬间打爆对方服务器和自己的文件描述符（`OSError: Too many open files`）。

其它并发原语（用法与 threading 版一致，但都是 `await` 的）：

```python
lock = asyncio.Lock();      async with lock: ...
event = asyncio.Event();    await event.wait(); event.set()
cond = asyncio.Condition(); async with cond: await cond.wait()
barrier = asyncio.Barrier(3)     # 3.11+
```

⚠️ **不要在 asyncio 里用 `threading.Lock`**——它会阻塞整个事件循环。

## 5. 生产者-消费者：`asyncio.Queue`

```python
async def producer(q: asyncio.Queue):
    for item in source:
        await q.put(item)         # 队列满时会挂起 → 背压
    for _ in range(N_WORKERS):
        await q.put(None)         # 哨兵

async def consumer(q: asyncio.Queue, wid: int):
    while True:
        item = await q.get()
        try:
            if item is None:
                break
            await handle(item)
        finally:
            q.task_done()

async def main():
    q = asyncio.Queue(maxsize=100)
    async with asyncio.TaskGroup() as tg:
        tg.create_task(producer(q))
        for i in range(N_WORKERS):
            tg.create_task(consumer(q, i))
```

## 6. 后台任务与"任务丢失"陷阱

```python
# ❌ 事件循环只持有 Task 的【弱引用】，没有强引用时任务可能被 GC 掉！
asyncio.create_task(background_job())

# ✅ 保存强引用
_background_tasks: set[asyncio.Task] = set()

def spawn(coro):
    t = asyncio.create_task(coro)
    _background_tasks.add(t)
    t.add_done_callback(_background_tasks.discard)
    return t
```

这是官方文档明确警告的坑，很多人不知道。

**未取回异常的警告**：

```python
t = asyncio.create_task(will_fail())
# 从不 await t → 进程退出时打印 "Task exception was never retrieved"
# ✅ 加回调处理
t.add_done_callback(lambda fut: fut.exception() and log.error("bg fail", exc_info=fut.exception()))
```

## 7. 同步/异步互操作

```python
# ① 异步中调同步阻塞函数 → 线程池
result = await asyncio.to_thread(blocking_fn, arg)                 # 3.9+ ★最简
result = await loop.run_in_executor(None, blocking_fn, arg)        # None = 默认线程池
result = await loop.run_in_executor(process_pool, cpu_heavy, arg)  # CPU 密集用进程池

# ② 同步中调异步函数
asyncio.run(async_fn())                     # 顶层入口
# 已有循环在跑的另一个线程里：
fut = asyncio.run_coroutine_threadsafe(async_fn(), loop)
fut.result(timeout=10)                      # 返回 concurrent.futures.Future

# ③ 从其它线程安全地调度回调
loop.call_soon_threadsafe(callback, arg)
```

> ⚠️ `asyncio.to_thread` 用的是**事件循环的默认 executor**，
> 即 `ThreadPoolExecutor()`，默认 `min(32, cpu_count + 4)` 个线程——
> **大量并发的阻塞调用会排队**。高负载场景要自建更大的 executor 并显式传给 `run_in_executor`。
>
> 注意 **FastAPI 跑 `def` 路由用的是另一个池**（Starlette 经 anyio，默认 40），
> 两者互不相干，见 [[web/fastapi-core]]。

## 8. 异步上下文与 `contextvars`

`threading.local()` 在 asyncio 里**不管用**（多个协程共享同一线程）。
用 `contextvars`：

```python
from contextvars import ContextVar

request_id: ContextVar[str] = ContextVar("request_id", default="-")

async def middleware(req):
    token = request_id.set(req.headers["X-Request-ID"])
    try:
        return await handler(req)
    finally:
        request_id.reset(token)

async def deep_inside():
    log.info("处理中", extra={"rid": request_id.get()})   # 自动拿到当前请求的 ID
```

**每个 Task 创建时会复制当前 context**——所以子任务能看到父任务的值，
但子任务里的修改不会影响父任务。这正是"请求级上下文"的正确实现方式
（FastAPI/Starlette 的中间件、SQLAlchemy 的 async session 都靠它）。

## 9. 异步迭代与上下文管理器

```python
# 异步生成器：流式处理数据库游标 / HTTP 流
async def stream_rows(conn):
    async with conn.transaction():
        async for row in conn.cursor("SELECT * FROM big_table"):
            yield row

async for row in stream_rows(conn):
    process(row)

# 异步推导式
results = [x async for x in agen() if await check(x)]

# 异步上下文管理器
from contextlib import asynccontextmanager

@asynccontextmanager
async def get_conn(pool):
    conn = await pool.acquire()
    try:
        yield conn
    finally:
        await pool.release(conn)

async with get_conn(pool) as conn: ...

# 动态数量的异步资源
from contextlib import AsyncExitStack
async with AsyncExitStack() as stack:
    conns = [await stack.enter_async_context(get_conn(p)) for p in pools]
```

## 10. 常用异步生态

| 用途 | 库 |
|---|---|
| HTTP 客户端 | `httpx`（同步+异步同一 API）、`aiohttp` |
| PostgreSQL | `asyncpg`（最快）、`psycopg` 3（支持 async） |
| MySQL | `aiomysql`、`asyncmy` |
| Redis | `redis.asyncio`（原 aioredis 已合并） |
| ORM | SQLAlchemy 2.0 async、`Tortoise ORM`、`SQLModel` |
| 文件 I/O | `aiofiles`（本质是线程池包装） |
| 任务队列 | `arq`、`celery`（有限支持）、`dramatiq` |
| 测试 | `pytest-asyncio`、`anyio` |
| 更快的循环 | `uvloop` |
| 跨后端抽象 | `anyio`（同时支持 asyncio 和 trio，Starlette 内部在用） |

## 11. 调试清单

```python
asyncio.run(main(), debug=True)          # 慢回调警告、未 await 的协程警告
PYTHONASYNCIODEBUG=1

asyncio.all_tasks()                       # 当前所有任务
asyncio.current_task()
task.get_stack() / task.print_stack()     # 协程的挂起位置
task.get_name() / task.set_name("fetch-user")   # 3.8+，给任务命名，日志友好

# 生产环境
py-spy dump --pid <pid>                   # 能看到协程栈
```

常见报错速查：

| 报错 | 原因 |
|---|---|
| `coroutine ... was never awaited` | 忘了 `await` 或 `create_task` |
| `no running event loop` | 在同步上下文里调用了需要循环的 API |
| `asyncio.run() cannot be called from a running event loop` | 嵌套 run（Jupyter 常见） |
| `Task exception was never retrieved` | 后台任务失败但没人取结果 |
| `Task was destroyed but it is pending!` | 循环结束时还有未完成任务，没有优雅关闭 |
| `Event loop is closed` | 循环关闭后还在用它（常见于测试用例间） |

## 相关

- [[concurrency/asyncio-fundamentals]] —— 原理
- [[concurrency/concurrency-models]] —— 选型
- [[language/exceptions]] —— ExceptionGroup 与 `except*`
- [[language/context-managers]] —— `@asynccontextmanager`
- [[web/fastapi-core]] —— FastAPI 中的落地
- [[web/fastapi-async-db]] —— 异步数据库会话管理
- [[bridge/async-js-vs-python]] —— 与 JS 的对照
