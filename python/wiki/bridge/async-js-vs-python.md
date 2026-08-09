---
title: "两种事件循环：JS vs Python asyncio"
date: 2026-08-07
tags: [asyncio, 事件循环, Promise, 协程, 对照]
sources: ["fastapi-doc/async.md"]
---

# 两种事件循环：JS vs Python asyncio

你已经有 JS 事件循环的完整心智模型。这一页只讲**差异**——把这些差异内化，
asyncio 就等于免费掌握了。

## 1. 五个必须知道的差异

### 差异 ①：事件循环不是自动跑的

```javascript
// JS：运行时永远在跑事件循环
fetchData();                      // 立即开始执行
```

```python
# Python：必须显式启动
asyncio.run(main())               # ← 没有这一句，任何协程都不会跑
```

**推论**：Python 程序默认是同步的，asyncio 是"一块区域"而不是"整个运行时"。
所以才会有"同步世界 ↔ 异步世界"的桥接问题（`asyncio.to_thread` / `run_coroutine_threadsafe`）。

### 差异 ②：协程是惰性的（最重要）★

```javascript
const p = work();       // 立即执行到第一个 await，返回 Promise
```

```python
c = work()              # ❗ 什么都不做，只是创建了一个协程对象
await c                 # 这时才开始执行
```

| | JS Promise | Python Coroutine |
|---|---|---|
| 创建时 | **eager**：立即开始执行 | **lazy**：不执行 |
| 谁驱动 | 运行时自动 | `await` 或 `create_task` |
| 可否多次 await | ✅ 可以（缓存结果） | ❌ **只能一次**（第二次抛 RuntimeError） |
| 忘了 await | 仍会执行（可能 unhandled rejection） | **完全不执行** + RuntimeWarning |

```python
# ❗ 这两行在 JS 里等价，在 Python 里天差地别
await a(); await b()                         # 串行，2 秒
await asyncio.gather(a(), b())               # 并发，1 秒

# JS 里：
// const pa = a(), pb = b();  ← 这里就已经并发开始了
// await pa; await pb;
```

**要在 Python 里模拟 JS 的 eager 行为，就用 `create_task`**：

```python
t1 = asyncio.create_task(a())     # 现在开始执行了（≈ JS 调用 async 函数）
t2 = asyncio.create_task(b())
await t1; await t2                # 并发
```

> **面试落点**：这是 asyncio 面试的第一道分水岭题。
> 「为什么 `await a(); await b()` 不是并发？」
> 答：**协程是惰性的，`await` 会等它完成才继续；要并发必须先把协程包成 Task
> 交给事件循环调度（`create_task` / `gather` / `TaskGroup`）。**

### 差异 ③：取消是真取消

```javascript
// JS：Promise 无法取消，AbortController 只是"请求"对方停止
const ctrl = new AbortController();
fetch(url, {signal: ctrl.signal});
ctrl.abort();          // fetch 内部检查 signal 后才停
```

```python
task = asyncio.create_task(work())
task.cancel()          # 在 work() 的挂起点【真的抛出】CancelledError
```

**后果**：Python 的协程必须考虑"我可能在任何 await 点被中断"，
所以 `try/finally` 和资源清理比 JS 里重要得多。

```python
async def work():
    try:
        await step1()
        await step2()      # 可能在这里被取消
    finally:
        await cleanup()    # 一定会执行
```

`CancelledError` 继承 `BaseException`，所以 `except Exception` 不会误吞它——
这个设计正是为了防止你"不小心把取消吃掉了"。

### 差异 ④：没有微任务队列的分层

```javascript
// JS 有明确的优先级
Promise.resolve().then(() => console.log("microtask"));   // 先
setTimeout(() => console.log("macrotask"), 0);            // 后
```

```python
# Python 里所有回调进同一个 ready 队列，没有 micro/macro 之分
await asyncio.sleep(0)      # ≈ "让出一次控制权"，没有优先级含义
loop.call_soon(cb)          # 下一轮
loop.call_later(0, cb)      # 用定时器堆，实际上比 call_soon 晚
```

**好处**：心智更简单，没有 JS 里"微任务饿死宏任务"那类问题。
**代价**：没法表达"这个回调要优先执行"。

### 差异 ⑤：阻塞的后果与解法不同

两者**都**会被同步 CPU 代码卡死事件循环。但解法不同：

| | JS | Python |
|---|---|---|
| 卸载 CPU 工作 | Web Worker / worker_threads（**内存隔离**，要序列化） | `asyncio.to_thread`（**共享内存**，但受 GIL）或进程池（真并行） |
| 阻塞 I/O | Node 内部用线程池（libuv），对用户透明 | **要自己处理**：`asyncio.to_thread` 或用异步库 |
| 现状 | 生态几乎全异步 | **同步库仍占多数**（requests、psycopg2、pandas…） |

```python
# Python 的万能兜底
result = await asyncio.to_thread(any_blocking_function, arg)
```

> **这是 Python asyncio 最大的实践痛点**：**生态分裂成同步和异步两套**。
> `requests` 不能在 asyncio 里用，要换 `httpx`；`psycopg2` 要换 `asyncpg`；
> 一个不小心引入的同步库就毁掉整个异步收益。JS 没有这个问题（fetch 天生异步）。
> 面试里能说出这一点，说明你真的用过。

## 2. API 对照表

| JavaScript | Python asyncio |
|---|---|
| `async function f()` | `async def f()` |
| `await p` | `await awaitable` |
| `Promise.resolve(v)` | `asyncio.sleep(0)` 后返回 / `Future` + `set_result` |
| `Promise.all([...])` | `asyncio.gather(*coros)` |
| `Promise.allSettled` | `gather(*coros, return_exceptions=True)` |
| `Promise.race` | `asyncio.wait(tasks, return_when=FIRST_COMPLETED)` |
| `Promise.any` | 无内置（手写或 `wait` + 筛选） |
| `new Promise((res) => ...)` | `loop.create_future()` + `fut.set_result(v)` |
| `setTimeout(f, ms)` | `loop.call_later(s, f)` |
| `await sleep(ms)` | `await asyncio.sleep(s)`（**秒，不是毫秒！**） |
| `queueMicrotask(f)` | `loop.call_soon(f)` |
| `AbortController` | `task.cancel()` |
| `AbortSignal.timeout(ms)` | `async with asyncio.timeout(s)` |
| `for await (const x of gen)` | `async for x in agen` |
| `async function*` | `async def` + `yield` |
| `Symbol.asyncIterator` | `__aiter__` / `__anext__` |
| `await using` (TS 5.2) | `async with` |
| `EventEmitter` | 无内置；`asyncio.Queue` / `Event` |
| `p.then(f).catch(g)` | `task.add_done_callback(cb)`（更底层） |
| unhandled rejection | "Task exception was never retrieved" 警告 |
| Worker | `asyncio.to_thread` / `ProcessPoolExecutor` |

**时间单位是最容易犯的低级错误**：`asyncio.sleep(1)` 是 1 **秒**，
不是 1 毫秒。同理 `timeout=5` 也是秒。

## 3. 同一段逻辑的两种写法

```javascript
// JS
async function fetchAll(urls) {
  const results = await Promise.all(
    urls.map(async (u) => {
      const r = await fetch(u);
      return r.json();
    })
  );
  return results;
}
```

```python
# Python（对应写法）
async def fetch_all(urls: list[str]) -> list[dict]:
    async with httpx.AsyncClient() as client:
        async def one(u: str) -> dict:
            r = await client.get(u)
            return r.json()
        return await asyncio.gather(*(one(u) for u in urls))

# 更现代（3.11+，带自动取消）
async def fetch_all(urls):
    async with httpx.AsyncClient() as client:
        async with asyncio.TaskGroup() as tg:
            tasks = [tg.create_task(client.get(u)) for u in urls]
    return [t.result().json() for t in tasks]
```

带并发限制（JS 需要第三方库如 p-limit，Python 内置）：

```python
sem = asyncio.Semaphore(10)
async def one(u):
    async with sem:
        return await client.get(u)
```

## 4. 错误处理对照

```javascript
try {
  const [a, b] = await Promise.all([f1(), f2()]);
} catch (e) {           // 只拿到第一个失败的
  ...
}
```

```python
# gather：只抛第一个异常，其余任务【继续在后台跑】⚠️
try:
    a, b = await asyncio.gather(f1(), f2())
except Exception as e:
    ...

# TaskGroup（3.11+）：其余任务【自动取消】，异常聚合成 ExceptionGroup ✅
try:
    async with asyncio.TaskGroup() as tg:
        t1 = tg.create_task(f1())
        t2 = tg.create_task(f2())
except* ValueError as eg:
    print(eg.exceptions)          # 所有 ValueError
except* ConnectionError as eg:
    ...
```

> `TaskGroup` + `except*` 是 JS 完全没有的能力（`AggregateError` 只在 `Promise.any` 里出现，
> 且没有按类型分组的语法）。**这是 Python 结构化并发的优势**。

## 5. 一个能背的对照陈述

> 「我有 Node 的异步经验，asyncio 的模型是一致的：单线程事件循环 + 协作式调度。
> 主要差异有三点：
> **① Python 的协程是惰性的**——调用 `async def` 只创建协程对象，
> 必须 `await` 或 `create_task` 才执行，所以并发要显式用 `gather`/`TaskGroup`；
> **② 取消是真的抛异常**（`CancelledError`），资源清理必须写在 `finally` 里；
> **③ 生态还有大量同步库**，在事件循环里调用它们会阻塞整个 worker，
> 要用 `asyncio.to_thread` 卸载到线程池。
> 另外 3.11 的 `TaskGroup` + `except*` 提供了比 `Promise.all` 更好的结构化并发语义——
> 失败时会自动取消兄弟任务。」

## 相关

- [[concurrency/asyncio-fundamentals]] —— asyncio 完整原理
- [[concurrency/asyncio-patterns]] —— 实战模式
- [[concurrency/concurrency-models]] —— 何时用 asyncio
- [[bridge/js-to-python]] —— 语法对照
- [[web/wsgi-asgi]] —— ASGI 与 Node http 的对照
- [[interview/question-bank-internals-concurrency]] —— asyncio 面试题
