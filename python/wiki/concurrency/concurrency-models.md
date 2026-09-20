---
title: "三种并发模型与选型"
date: 2026-08-07
tags: [并发, 并行, 线程, 进程, 协程, 选型]
sources: ["fastapi-doc/async.md", "interview-python-cn.md"]
---

# 三种并发模型与选型

「多线程、多进程、协程什么时候用哪个」——后端面试的核心综合题。
答不出选型依据，前面讲再多 GIL 也白搭。

## 1. 先分清概念

| 概念 | 定义 |
|---|---|
| **并发（concurrency）** | 多个任务**交替推进**，逻辑上同时；单核也能并发 |
| **并行（parallelism）** | 多个任务**物理上同时**执行；需要多核 |
| **同步/阻塞** | 调用后必须等结果才能继续 |
| **异步/非阻塞** | 调用立即返回，结果通过回调/await/轮询获取 |

Rob Pike 的名言：*"Concurrency is about dealing with lots of things at once.
Parallelism is about doing lots of things at once."*

**Python 里的关键事实**：受 GIL 限制，**多线程只有并发没有并行（对 Python 字节码而言）**；
真并行必须靠多进程或释放 GIL 的 C 扩展。见 [[internals/gil]]。

## 2. 三种模型对比

| | threading | multiprocessing | asyncio |
|---|---|---|---|
| 调度者 | 操作系统（抢占式） | 操作系统 | **事件循环（协作式）** |
| 并行 CPU | ❌（GIL） | ✅ | ❌ |
| 切换成本 | 中（~µs，内核态） | 高（~ms，进程创建） | **极低（~ns，函数调用级）** |
| 内存 | 共享（同一地址空间） | **隔离**（需 IPC） | 共享 |
| 单个单位开销 | ~8 MB 栈（虚拟） | ~10-50 MB | **~KB** |
| 可扩展数量 | 数百~数千 | ≈ CPU 核数 | **数万~数十万** |
| 数据共享 | 直接（需加锁） | pickle 序列化 / 共享内存 | 直接（**无需锁**，单线程） |
| 调试难度 | 高（竞态、死锁） | 中 | 中（但栈追踪难读） |
| 代码改造 | 小 | 小 | **大（全链路 async 化）** |
| 适用 | I/O 密集 + 用到同步库 | **CPU 密集** | **高并发 I/O** |

## 3. 决策树

```text
你的瓶颈是什么？
│
├── CPU 密集（计算、图像、加解密、序列化）
│   ├── 纯 Python 计算 → multiprocessing / ProcessPoolExecutor
│   ├── 数值计算       → NumPy / Polars（底层已释放 GIL 并向量化）
│   └── 极致性能       → Cython / Rust(PyO3) / 换语言
│
└── I/O 密集（网络、数据库、文件、外部 API）
    ├── 并发量小（< 几百）且依赖同步库 → ThreadPoolExecutor  ★最省事
    ├── 并发量大（数千+）且生态支持 async → asyncio          ★最高效
    └── 已经在 async 框架里（FastAPI）   → asyncio + 线程池兜底同步库
```

**判断瓶颈类型的方法**：任务运行时看 CPU 占用。
接近 100%（单核）→ CPU 密集；很低但耗时长 → I/O 密集。

```bash
py-spy top --pid <pid>      # 看时间花在哪个函数
```

## 4. 同一任务的四种写法

```python
import time, requests
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor

URLS = ["https://httpbin.org/delay/1"] * 20

# ① 串行：20 秒
def serial():
    return [requests.get(u).status_code for u in URLS]

# ② 线程池：~1 秒（I/O 时释放 GIL）
def threaded():
    with ThreadPoolExecutor(max_workers=20) as ex:
        return list(ex.map(lambda u: requests.get(u).status_code, URLS))

# ③ 进程池：~1 秒但开销大（这里是 I/O 任务，用进程是浪费）
def processed():
    with ProcessPoolExecutor(max_workers=8) as ex:
        return list(ex.map(fetch, URLS))     # fetch 必须是模块级函数（可 pickle）

# ④ asyncio：~1 秒，且可以轻松扩到 10000 个 URL
import asyncio, httpx
async def asyncio_way():
    async with httpx.AsyncClient() as client:
        rs = await asyncio.gather(*(client.get(u) for u in URLS))
        return [r.status_code for r in rs]
```

CPU 密集任务的对比：

```python
def cpu_bound(n):
    return sum(i * i for i in range(n))

# 串行 4 次        : 4.0s
# ThreadPool(4)   : 4.2s   ❌ 更慢！GIL + 切换开销
# ProcessPool(4)  : 1.1s   ✅ 真并行
# asyncio         : 4.0s   ❌ 毫无帮助，还会阻塞整个事件循环
```

> **面试落点**：「asyncio 能加速 CPU 密集任务吗？」——**完全不能，还更糟**。
> asyncio 是单线程协作式调度，一个 CPU 密集协程**不 await 就永不让出**，
> 会把整个事件循环卡死（其它请求全部超时）。这是 FastAPI 生产事故的头号原因。

## 5. `concurrent.futures` —— 统一的高层接口

**优先用它**，而不是直接操作 Thread/Process。

```python
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor, as_completed

with ThreadPoolExecutor(max_workers=10, thread_name_prefix="worker") as ex:
    # 方式 1：map，保持输入顺序，异常在取结果时抛出
    for result in ex.map(fn, items, timeout=30):
        ...

    # 方式 2：submit + as_completed，谁先完成先处理 ★推荐
    futures = {ex.submit(fn, item): item for item in items}
    for fut in as_completed(futures):
        item = futures[fut]
        try:
            result = fut.result()
        except Exception as e:
            log.exception("处理 %s 失败", item)

# Future 接口
f = ex.submit(fn, x)
f.result(timeout=5)      # 阻塞等待，会重新抛出任务里的异常
f.done() / f.cancelled() / f.exception()
f.add_done_callback(cb)
```

两种 Executor 的接口**完全一致**——所以从线程切换到进程只需改一个类名。

`max_workers` 的经验值：

- **ThreadPoolExecutor**：I/O 密集时可以远大于核数（32、64、100 都行），
  默认是 `min(32, cpu_count + 4)`。
- **ProcessPoolExecutor**：默认 `cpu_count()`，一般不要超过它。

## 6. 混合模式（生产中最常见）

```python
# asyncio 主循环 + 线程池跑同步库 + 进程池跑 CPU 任务
import asyncio
from concurrent.futures import ProcessPoolExecutor

async def handler():
    # ① 同步阻塞库（比如老的 DB 驱动、requests、PIL）
    data = await asyncio.to_thread(blocking_db_query, sql)      # 3.9+ ★最简写法

    # ② CPU 密集
    loop = asyncio.get_running_loop()
    with ProcessPoolExecutor() as pool:
        result = await loop.run_in_executor(pool, cpu_heavy, data)

    return result
```

这正是 **FastAPI 的内部机制**：`def` 路由函数被放进线程池，`async def` 直接在事件循环跑。
见 [[web/fastapi-core]]。

## 7. 面试标准答题模板

> **问：Python 里多线程、多进程、协程怎么选？**
>
> 1. **先定位瓶颈**：CPU 密集还是 I/O 密集。
> 2. **CPU 密集** → 多进程（`ProcessPoolExecutor`），因为 GIL 让多线程无法并行；
>    数值计算优先 NumPy（底层释放 GIL）。
> 3. **I/O 密集** → 并发量不大或依赖同步库时用线程池（改造成本低）；
>    高并发（数千连接以上）用 asyncio，因为协程内存开销是 KB 级、切换是纳秒级，
>    而线程是 MB 级、微秒级。
> 4. **代价**：asyncio 要求**全链路异步**，一个同步阻塞调用就会卡死整个事件循环；
>    多进程有 IPC 序列化成本和内存开销。
> 5. **实践**：现代 Python 后端多是 asyncio 主循环 + `asyncio.to_thread` 兜底同步库
>    + 进程池处理 CPU 任务的混合模式。

## 相关

- [[internals/gil]] —— 为什么多线程不能并行
- [[concurrency/threading]] —— 线程与锁的细节
- [[concurrency/multiprocessing]] —— 进程与 IPC
- [[concurrency/asyncio-fundamentals]] —— 事件循环原理
- [[concurrency/asyncio-patterns]] —— asyncio 实战模式
- [[web/fastapi-cpu-bound]] —— FastAPI 里 CPU 密集任务的完整解法（实测数据）
- [[web/fastapi-production]] —— worker 与并发模型的生产配置
- [[bridge/async-js-vs-python]] —— 与 Node 单线程模型的对照
