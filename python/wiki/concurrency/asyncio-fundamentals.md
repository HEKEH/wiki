---
title: "asyncio 原理：事件循环与协程"
date: 2026-08-07
tags: [asyncio, 协程, 事件循环, async, await, coroutine]
sources: ["fastapi-doc/async.md", "pep-0492-async-await.rst"]
---

# asyncio 原理：事件循环与协程

**这一章是前端工程师的最大优势项**——你已经完全理解事件循环、微任务、Promise 链。
Python 的 asyncio 是同一套思想，差别在几个关键细节上。把这些差别搞清楚，
面试里讲 asyncio 会比科班 Python 出身的候选人更透彻。

## 1. 一句话本质

> **协程 = 可以在中途挂起、稍后从挂起点恢复的函数。**
> **事件循环 = 一个单线程调度器，不断从就绪队列取出协程恢复执行，遇到 `await` 就挂起它、换下一个。**

关键洞察：**并发不是靠多线程，而是靠"函数能暂停"**。
所以整个 asyncio 建立在 [[language/iterators-generators]] 的挂起/恢复机制上——
`async def` 编译出的 code object 和生成器共用同一套帧保存逻辑。

## 2. 与 JS 的核心对照

| | JavaScript | Python asyncio |
|---|---|---|
| 事件循环 | **运行时内置**，永远在跑 | **需要显式启动**：`asyncio.run(main())` |
| 异步函数 | `async function` | `async def` |
| 等待 | `await promise` | `await awaitable` |
| 立即执行 | **调用即开始执行**（eager） | **调用只创建协程对象，不执行**（lazy）★ |
| 并发原语 | `Promise` | `Task`（`Future` 是更底层的） |
| 并发聚合 | `Promise.all()` | `asyncio.gather()` / `TaskGroup` |
| 竞速 | `Promise.race()` | `asyncio.wait(..., FIRST_COMPLETED)` |
| 容错聚合 | `Promise.allSettled()` | `gather(..., return_exceptions=True)` |
| 任一成功 | `Promise.any()` | 无内置（手写或用 `wait`） |
| 定时 | `setTimeout` | `asyncio.sleep()` / `loop.call_later` |
| 取消 | `AbortController`（协作式） | `task.cancel()`（抛 `CancelledError`）★ |
| 阻塞 CPU | 卡死事件循环 | **同样卡死事件循环** |
| 多线程 | Worker（内存隔离） | `asyncio.to_thread`（共享内存） |

### 差异 ①：协程是惰性的（最容易踩的坑）

```python
async def work():
    print("running")
    return 1

coro = work()          # ❗ 什么都没发生，没有打印
await coro             # 这时才执行

# JS 对比：
# const p = work();    // 立即开始执行，打印 running
# await p;
```

**后果**：

```python
async def main():
    fetch(url)            # ❌ 忘了 await → 协程从未执行
                          #    RuntimeWarning: coroutine 'fetch' was never awaited
    await fetch(url)      # ✅
```

想要"立即开始执行"必须包成 Task：

```python
task = asyncio.create_task(fetch(url))     # ✅ 现在它被排进事件循环，开始跑
...其它工作...
result = await task                         # 稍后取结果
```

> **面试落点**：「`asyncio.create_task` 和直接 `await` 有什么区别？」
> **`await coro` 是顺序执行（等它完成才继续）；`create_task` 把协程交给事件循环调度，
> 立即返回 Task，实现真正的并发。** 这道题几乎必问。

```python
# ❌ 串行，总耗时 3 秒
async def bad():
    a = await fetch(1)      # 等 1 秒
    b = await fetch(2)      # 再等 1 秒
    c = await fetch(3)      # 再等 1 秒

# ✅ 并发，总耗时 1 秒
async def good():
    a, b, c = await asyncio.gather(fetch(1), fetch(2), fetch(3))

# ✅ 也可以手动
async def good2():
    t1 = asyncio.create_task(fetch(1))
    t2 = asyncio.create_task(fetch(2))
    a, b = await t1, await t2
```

### 差异 ②：取消是真取消

JS 的 Promise 一旦创建无法取消（AbortController 只是通知）；
Python 的 `task.cancel()` 会**在协程的挂起点抛出 `CancelledError`**，真正中断它。

```python
task = asyncio.create_task(long_running())
await asyncio.sleep(1)
task.cancel()
try:
    await task
except asyncio.CancelledError:
    print("已取消")
```

协程内部可以捕获并清理，但**必须重新抛出**：

```python
async def worker():
    try:
        await do_work()
    except asyncio.CancelledError:
        await cleanup()
        raise             # ✅ 必须重抛，否则取消语义被破坏
    finally:
        release()
```

> ⚠️ `asyncio.CancelledError` 在 3.8+ 继承自 **`BaseException`** 而不是 `Exception`，
> 所以 `except Exception:` **不会**误吞它。这是刻意的设计。

## 3. 事件循环做了什么

```text
┌─────────────────────────────────────────┐
│  Ready Queue（就绪回调）                  │
│   ├─ 恢复挂起完成的协程                    │
│   └─ call_soon 注册的回调                 │
├─────────────────────────────────────────┤
│  Scheduled（定时堆，heapq）               │
│   └─ call_later / sleep 到期的            │
├─────────────────────────────────────────┤
│  Selector（epoll / kqueue / IOCP）        │
│   └─ 等待 socket/fd 可读可写               │
└─────────────────────────────────────────┘

单轮循环：
1. 执行 ready 队列里的所有回调（本轮加入的留到下一轮）
2. 检查定时堆，到期的移入 ready
3. 用 selector 轮询 I/O（超时 = 最近的定时器时间）
4. 就绪的 I/O 对应的回调移入 ready
5. 回到 1
```

**这与 Node 的 libuv 事件循环几乎同构**（Node 多了 phases 的划分和 microtask 队列的细分）。

> 第三层的 **Selector 是整个 asyncio 的物理底座**——它就是 `select`/`poll`/`epoll`/`kqueue`。
> 「asyncio 底层怎么实现的」这个追问的终点在那里，
> 而且「asyncio 严格说不是异步 IO 而是 IO 多路复用」是很好的加分点。
> 见 [[concurrency/io-multiplexing]]。

核心区别：**Python 没有独立的"微任务队列"**。所有回调都在同一个 ready 队列，
`await` 让出后就是普通的 `call_soon`。所以 JS 里"微任务优先于宏任务"的那套排序规则，
在 Python 里不存在。

```python
# 让出控制权一次（相当于 JS 的 await Promise.resolve() 或 setTimeout(0)）
await asyncio.sleep(0)
```

## 4. 三个核心概念：Coroutine / Future / Task

```text
Awaitable（可 await 的东西）
├── Coroutine   —— async def 调用的产物；惰性；只能被 await 一次
├── Future      —— 一个"将来会有结果的容器"（≈ JS Promise），底层原语
└── Task        —— Future 的子类，包装一个 Coroutine 并交给事件循环调度 ★
```

```python
import asyncio, inspect

async def f(): return 1

c = f()
inspect.iscoroutine(c)                 #=> True
asyncio.iscoroutinefunction(f)         #=> True

# 在事件循环里
async def main():
    t = asyncio.create_task(f())       # Coroutine → Task
    isinstance(t, asyncio.Future)      #=> True
    t.done() / t.result() / t.cancelled() / t.exception()
    t.add_done_callback(cb)

    fut = asyncio.get_running_loop().create_future()   # 手工 Future
    # 别处：fut.set_result(x)  → 让 await fut 的协程恢复
```

`Future` 用于**桥接回调式代码到 async/await**：

```python
# 把一个回调 API 包装成可 await 的
async def wait_for_callback():
    loop = asyncio.get_running_loop()
    fut = loop.create_future()
    def on_done(result):
        loop.call_soon_threadsafe(fut.set_result, result)   # 跨线程要用这个
    legacy_api.register(on_done)
    return await fut
```

## 5. `await` 到底做了什么

```python
async def outer():
    r = await inner()
```

字节码层面 `await x` ≈ `yield from x.__await__()`：

1. 调用 `type(x).__await__(x)` 拿到一个迭代器。
2. 反复 `next()` 它。每次 yield 出来的值**穿透所有 await 层，直达事件循环**。
3. 事件循环拿到的通常是一个 `Future`，于是注册"这个 Future 完成时恢复该 Task"。
4. Future 完成 → 事件循环把 Task 放回 ready 队列 → 从挂起点继续。

**关键推论**：**只有真正会挂起的 `await` 才让出控制权**。

```python
async def not_really_async():
    return 1                     # 没有任何挂起点

await not_really_async()         # 全程占用事件循环，不会给别人机会
```

这就是为什么"把函数标成 `async def` 不会让它变快"——**异步的收益来自 I/O 等待期间去做别的事**。

## 6. 头号事故：阻塞事件循环

```python
# ❌ 这些会把【整个进程的所有并发请求】卡死
async def handler():
    time.sleep(1)                    # 同步 sleep
    requests.get(url)                # 同步 HTTP
    conn.execute(sql)                # 同步 DB 驱动
    json.dumps(huge_object)          # 大 CPU 操作
    open("f").read()                 # 同步文件 I/O（大文件时）
    hashlib.pbkdf2_hmac(...)         # 密码哈希，故意很慢

# ✅ 正确做法
async def handler():
    await asyncio.sleep(1)                          # 异步 sleep
    await httpx_client.get(url)                     # 异步 HTTP 库
    await async_conn.execute(sql)                   # 异步驱动（asyncpg/aiomysql）
    await asyncio.to_thread(json.dumps, huge)       # 丢给线程池 ★万能兜底
    result = await loop.run_in_executor(proc_pool, cpu_heavy, x)   # CPU → 进程池
```

**检测阻塞**：

```python
asyncio.run(main(), debug=True)     # 慢回调（>100ms）会打印警告
# 或环境变量 PYTHONASYNCIODEBUG=1
# 生产用 aiodebug / 自定义 loop 监控
```

> **面试落点**：「asyncio 里调用一个同步阻塞函数会怎样？」——
> **整个事件循环被阻塞，该进程上所有并发任务全部停滞**（不只是当前请求）。
> 因为 asyncio 是单线程协作式调度，协程不主动让出就没人能抢占它。
> 解法是 `asyncio.to_thread()` 或 `run_in_executor`。

## 7. 入口与运行

```python
# ✅ 推荐（3.7+）：自动创建循环、运行、清理（含 shutdown_asyncgens）
asyncio.run(main())
asyncio.run(main(), debug=True)

# 3.11+ 更精细的控制
async def main(): ...
with asyncio.Runner() as runner:
    runner.run(main())

# ❌ 老写法，别再用
loop = asyncio.get_event_loop()      # 3.12 起在无运行循环时会 DeprecationWarning
loop.run_until_complete(main())

# 在协程内部拿循环
loop = asyncio.get_running_loop()    # ✅ 3.7+
```

**一个线程同一时刻只能有一个运行中的事件循环**，`asyncio.run` 不能嵌套：

```python
async def f():
    asyncio.run(g())      # ❌ RuntimeError: asyncio.run() cannot be called from a
                          #    running event loop
    await g()             # ✅
```

（Jupyter/IPython 里已经有循环在跑，所以要直接 `await` 或用 `nest_asyncio`。）

## 8. 更快的事件循环

```python
import uvloop
uvloop.install()          # 或 asyncio.set_event_loop_policy(uvloop.EventLoopPolicy())
# 3.12+ 推荐：
# asyncio.run(main(), loop_factory=uvloop.new_event_loop)
```

**uvloop** 基于 libuv（就是 Node 用的那个），用 Cython 写，比标准事件循环快 **2~4 倍**。
uvicorn 默认就会用它（如果装了）。

## 相关

- [[concurrency/asyncio-patterns]] —— gather/TaskGroup/超时/取消/队列等实战
- [[concurrency/concurrency-models]] —— 何时该用 asyncio
- [[language/iterators-generators]] —— 协程与生成器的血缘
- [[bridge/async-js-vs-python]] —— 与 JS 事件循环的深度对照
- [[web/wsgi-asgi]] —— ASGI 与 asyncio 的关系
- [[web/fastapi-core]] —— `async def` vs `def` 路由
- [[interview/question-bank-internals-concurrency]] —— asyncio 面试题
- [[sources/cpython-docs]] —— 来源：PEP 492
