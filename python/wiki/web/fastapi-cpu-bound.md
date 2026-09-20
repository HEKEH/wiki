---
title: "FastAPI 里的 CPU 密集任务：别阻塞事件循环"
date: 2026-09-13
tags: [FastAPI, asyncio, CPU密集, GIL, 进程池, 背压, 生产事故]
sources: []
---

# FastAPI 里的 CPU 密集任务：别阻塞事件循环

「某个 service 要占用长时间 CPU，怎么不阻塞其它请求？」——FastAPI 面试的必问追问，
也是 ASGI 服务最常见的生产事故。本页给出**实测数据 + 可直接抄的生产形态代码 + 四个坑**。

前置：[[concurrency/concurrency-models]]（三模型选型）、[[web/fastapi-core]] §4（`async def` vs `def`）。

## 1. 为什么在 FastAPI 里特别致命

事件循环本质是 `while True` 从就绪队列取回调**同步执行**。一个协程运行期间，
循环不在跑——**它就是那个协程的调用栈**。所以「卡住一个协程」＝「卡住整个 worker 的所有请求」。

对照多线程模型：OS 每 5ms（`sys.getswitchinterval()` → `0.005`）强制抢占，
别的线程仍能推进。asyncio 是**协作式**的，没有抢占权，协程不 `await` 就永不让出。

## 2. 三种方案的实测对比

跑真实 uvicorn 服务，并发打 4 个 CPU 请求（每个 ~0.7s 纯 Python 计算），
同时用另一条连接持续探 `/health`：

```console
$ python probe.py
空载基线                                   | /health 中位   4.5ms 最坏     5.9ms
① async def 直接算    4 请求  5.30s | 进程 1 | /health 中位 5262.9ms 最坏  5262.9ms
② await to_thread     4 请求  8.87s | 进程 1 | /health 中位   99.0ms 最坏   686.3ms
③ run_in_executor(进程池) 4 请求 2.50s | 进程 4 | /health 中位  10.1ms 最坏   161.0ms
```

三个要点，每个都是面试落点：

**① 直接算 → `/health` 中位延迟从 4.5ms 涨到 5262.9ms（1170 倍）。**
注意是**中位数**不是尾部——这段时间内**每一个**请求都在排队，不是「偶尔卡一下」。
这就是「asyncio 跑 CPU 密集比串行**更糟**」的含义：串行只慢自己，它慢的是所有人。

**② `to_thread` 解除了冻结，但没解决 GIL。** 中位 99ms（仍是基线 22 倍），
因为纯 Python 计算在线程里照样持有 GIL，事件循环线程每 5ms 才抢到一个时间片。
更反直觉的是**总耗时 8.87s 比直接算的 5.30s 还长**——4 个线程互抢 GIL，
切换开销纯属白给。**对纯 Python CPU 任务，线程池只是缓解，不是解法。**

**③ 进程池是唯一真解**：4 个请求落在 **4 个不同子进程**上真并行（2.50s），
`/health` 中位 10.1ms，基本回到基线。

> ⚠️ **② 的例外**：如果 CPU 任务底层**会释放 GIL**（NumPy、Pillow、bcrypt、
> 压缩/加解密、`orjson` 等 C 扩展），`to_thread` 就足够了，不必上进程池。
> 判据不是「像不像 CPU 任务」，而是「**这段计算是不是纯 Python 字节码**」。

## 3. 生产形态代码

### `cpu_tasks.py` —— 子进程执行的纯函数

```python
"""★ 必须是独立模块的模块级函数：进程池要 pickle 函数引用，lambda / 闭包 / 嵌套函数都不行。"""
import os
import signal


def init_worker() -> None:
    """子进程初始化：忽略 SIGINT，让父进程独占 Ctrl-C，实现优雅关闭。"""
    signal.signal(signal.SIGINT, signal.SIG_IGN)


def crunch(n: int) -> dict:
    total = sum(i * i for i in range(n))
    return {"n": n, "total": total, "pid": os.getpid()}
```

### `app.py` —— 池的生命周期 + 背压 + 超时

```python
import asyncio
import logging
from concurrent.futures import ProcessPoolExecutor
from concurrent.futures.process import BrokenProcessPool   # ★ 不在顶层导出
from contextlib import asynccontextmanager

from fastapi import FastAPI, HTTPException

from cpu_tasks import crunch, init_worker

log = logging.getLogger("uvicorn.error")

MAX_WORKERS = 4
CPU_TIMEOUT = 10.0
state: dict = {}


@asynccontextmanager
async def lifespan(app: FastAPI):
    # ① 进程池在启动时建一次，全生命周期复用
    #    ❌ 绝不要在请求处理函数里 `with ProcessPoolExecutor() as pool:`
    #       ——每个请求 fork/spawn 4 个进程，开销比任务本身还大
    state["pool"] = ProcessPoolExecutor(max_workers=MAX_WORKERS, initializer=init_worker)
    # ② 背压闸门：排队深度封顶，超了立刻 503，而不是无限堆积直到 OOM
    state["gate"] = asyncio.Semaphore(MAX_WORKERS * 2)
    log.info("process pool ready: %d workers", MAX_WORKERS)
    try:
        yield
    finally:
        state["pool"].shutdown(wait=True, cancel_futures=True)


app = FastAPI(lifespan=lifespan)


async def run_cpu(fn, /, *args):
    """把 CPU 密集函数踢进进程池，带背压 + 超时 + 池损坏处理。"""
    gate: asyncio.Semaphore = state["gate"]
    if gate.locked():                      # 已满：快速失败，别让客户端干等
        raise HTTPException(503, "server busy, retry later")
    async with gate:
        loop = asyncio.get_running_loop()
        fut = loop.run_in_executor(state["pool"], fn, *args)
        try:
            return await asyncio.wait_for(fut, timeout=CPU_TIMEOUT)
        except asyncio.TimeoutError:
            raise HTTPException(504, "computation timed out")
        except BrokenProcessPool:          # 子进程 OOM / 段错误 → 整个池不可用
            raise HTTPException(500, "worker crashed")


@app.get("/compute")
async def compute(n: int = 20_000_000):
    return await run_cpu(crunch, n)


@app.get("/health")
async def health():
    return {"status": "ok"}
```

三条路径实测（CPython 3.11.9 / FastAPI 0.141.1）：

```console
$ python probe.py
20 个并发 CPU 请求的状态码分布: {200: 8, 503: 12}      # 闸门 = MAX_WORKERS*2 = 8
被拒的响应体: {'detail': 'server busy, retry later'}
超长任务 -> 504 {'detail': 'computation timed out'}
```

## 4. 四个坑

### ① 超时**杀不掉**子进程 ★最容易踩

`asyncio.wait_for` 取消的是 asyncio 那层 future，
而 `concurrent.futures.Future.cancel()` 对**已经在跑**的任务返回 `False`——
子进程根本不知道有人放弃了它，会一直算到天荒地老：

```python
fut = loop.run_in_executor(pool, crunch, 400_000_000)   # ~25s 的活
try:
    await asyncio.wait_for(fut, timeout=1.0)
except asyncio.TimeoutError:
    print("wait_for 在 1.00s 超时返回")      #=> wait_for 在 1.00s 超时返回
print("future.cancelled() ->", fut.cancelled())  #=> True    ← 骗人的

# 池里只有 1 个 worker，再提交一个秒级任务：
await loop.run_in_executor(pool, crunch, 10)
#=> 实际等了 14.61s   ← 子进程仍在跑那个「已超时」的任务
```

**后果**：客户端收到 504 了，worker 还在烧 CPU；超时越多，池被僵尸任务占得越死，
最终所有请求都 503。**504 只是止损给调用方，没有止损给服务端。**

真正的解法（按代价排序）：

1. **任务内部自检 deadline**——把计算切成块，每块检查一次时间预算，主动返回。
   唯一无需额外依赖、且能真正释放 worker 的办法。
2. **`max_tasks_per_child=N`**（3.11+）——让 worker 定期重建，至少能回收内存泄漏，
   但**救不了当前正在跑的任务**。
3. **`pebble` 库**——`ProcessPool` 支持真超时，到点直接 `terminate()` 子进程。
4. **真任务队列**（Celery `time_limit` / `soft_time_limit`）——见下文 §5。

### ② `max_workers` 要和 uvicorn worker 数一起算

```bash
uvicorn app:app --workers 4      # 4 个 API 进程
# 每个 API 进程又建 MAX_WORKERS=4 的池 → 4 × (1 + 4) = 20 个进程！
```

8 核机器上这会严重超卖。经验值：**`uvicorn workers × MAX_WORKERS ≈ CPU 核数`**。
容器化部署推荐 `--workers 1` + `MAX_WORKERS = cpu_count()`，副本数交给 K8s
（见 [[web/fastapi-production]] §1）。内存也要一起算：每个子进程 10–50 MB。

### ③ pickle 往返是有成本的

参数和返回值都要序列化跨进程。**传 200 MB 的 DataFrame 进去，序列化时间可能超过计算时间**——
那还不如不并行。此时的做法是传**引用**（文件路径、S3 key、DB 主键），让子进程自己去读。

### ④ Windows / macOS 的 spawn 语义

默认 start method 在 Windows 和 macOS 上是 `spawn`：子进程会**重新 import 主模块**。
所以启动代码必须放在 `if __name__ == "__main__":` 里，否则递归创建进程，
直接 `RuntimeError: An attempt has been made to start a new process before the
current process has finished its bootstrapping phase.`
用 uvicorn 命令行启动天然满足（模块被 import 而非执行），
但写脚本自测时极易踩。详见 [[concurrency/multiprocessing]]。

## 5. 什么时候该升级到任务队列

进程池的适用边界是**秒级、请求-响应模型内能等完**的计算。超出就该异步化：

| 任务时长 | 方案 |
|---|---|
| < 100ms | 直接算，不值得 offload（pickle 开销比计算还大） |
| 0.1s – 数秒 | **进程池 + `run_in_executor`**（本页方案） |
| 数十秒以上 / 需要重试、进度、持久化 | **Celery / RQ / Dramatiq / arq**：接口立即返回 `202 + task_id`，客户端轮询或 WebSocket 推送 |

```python
# 长任务的正确形态：接口不等结果
@app.post("/render", status_code=202)
async def render(payload: RenderIn):
    task = render_video.delay(payload.model_dump())   # Celery
    return {"task_id": task.id, "status_url": f"/tasks/{task.id}"}
```

> ⚠️ 别用 `BackgroundTasks` 跑 CPU 任务——它在**响应返回后、同一个事件循环里**执行，
> 同步函数会走 Starlette 的线程池（40 个 token），`async def` 的则直接在循环里跑。
> 两种都不解决 GIL，只是把阻塞推迟到了响应之后。

## 6. 面试落点

> **问：FastAPI 里某个接口要跑几秒的 CPU 计算，怎么办？**
>
> 1. **先说后果**：asyncio 是协作式调度，CPU 协程不 `await` 就永不让出，
>    卡的是整个 worker 的**所有**请求——实测 `/health` 中位延迟能从 4.5ms 涨到 5.2s。
> 2. **再分情况**：底层会释放 GIL 的（NumPy、Pillow、bcrypt）→ `await asyncio.to_thread(...)`
>    就够；**纯 Python 计算 → 必须进程池**，因为线程池里 GIL 照样争抢
>    （实测中位延迟仍有 22 倍劣化，总耗时甚至比不并行更长）。
> 3. **给工程形态**：进程池在 `lifespan` 里建一次复用，用 `Semaphore` 做背压
>    （满了返 503 而不是无限排队），`wait_for` 加超时，捕获 `BrokenProcessPool`。
> 4. **主动补坑**：`wait_for` 超时**杀不掉子进程**，`cancelled()` 返回 `True` 是假象，
>    worker 会继续烧 CPU——要么任务内部自检 deadline，要么上 `pebble` / Celery。
> 5. **划边界**：超过数十秒、需要重试和进度的，就不该用请求-响应模型，
>    改 `202 + task_id` + 任务队列。

第 4 点是拉开区分度的地方——大部分候选人答到第 3 点就停了。

## 相关

- [[concurrency/concurrency-models]] —— 三模型选型与决策树
- [[concurrency/asyncio-fundamentals]] —— 事件循环为什么会被卡住
- [[concurrency/multiprocessing]] —— 进程池、pickle 限制、fork vs spawn
- [[web/fastapi-core]] —— `async def` vs `def` 与两个线程池
- [[web/fastapi-production]] —— worker 数、内存与部署形态
- [[internals/gil]] —— GIL 何时释放
- [[review/review-set-06]] —— 相关自测题（选择 15、判断 9/13）
