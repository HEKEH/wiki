---
title: "multiprocessing 与进程间通信"
date: 2026-08-07
tags: [multiprocessing, 进程, IPC, pickle, 共享内存, fork, spawn]
sources: ["interview-python-cn.md", "python-cheatsheet.md"]
---

# multiprocessing 与进程间通信

**绕过 GIL 实现真并行的主力方案**。核心代价是：**内存不共享、数据要序列化**。

## 1. 基本用法

```python
from multiprocessing import Process, Queue, Pool
from concurrent.futures import ProcessPoolExecutor

def cpu_task(n):
    return sum(i * i for i in range(n))

# ✅ 推荐：高层接口
if __name__ == "__main__":                     # ← 必须有！见下文
    with ProcessPoolExecutor(max_workers=4) as ex:
        results = list(ex.map(cpu_task, [10**7] * 4))

# 低层接口
if __name__ == "__main__":
    p = Process(target=cpu_task, args=(10**7,))
    p.start(); p.join()
    p.exitcode
```

## 2. 三种启动方式（必考）

```python
import multiprocessing as mp
mp.get_start_method()          #=> 'spawn'（macOS/Windows）/ 'fork'（Linux，3.13 前）
mp.set_start_method("spawn", force=True)
ctx = mp.get_context("spawn")  # 更推荐：不改全局
```

| 方式 | 平台 | 机制 | 特点 |
|---|---|---|---|
| **fork** | Unix | 复制父进程内存（写时复制 CoW） | **快**；但**与线程/锁混用极易死锁**，子进程继承一切（包括半开的连接） |
| **spawn** | 全平台，macOS/Win 默认 | 启动全新解释器，重新导入主模块 | **慢但安全**；要求所有传递对象可 pickle |
| **forkserver** | Unix | 预先 fork 一个干净的服务进程，由它来 fork | 兼顾速度与安全 |

> **重要变化**：Python **3.14 起 Linux 上的默认从 `fork` 改为 `forkserver`**，
> 因为 fork + 线程的组合（几乎所有现代程序都有后台线程）会导致难以排查的死锁。
> 能说出这个变化是紧跟版本的表现。

**`if __name__ == "__main__":` 为什么必须有**：spawn 模式下子进程会**重新导入主模块**，
没有守卫就会再次执行创建进程的代码 → **无限递归 fork 炸弹**。

```python
# ❌ macOS/Windows 上直接崩溃或无限创建进程
p = Process(target=f)
p.start()

# ✅
if __name__ == "__main__":
    p = Process(target=f)
    p.start()
```

fork 陷阱示例（Linux 上真实发生）：

```python
# ❌ 父进程持有锁的瞬间 fork，子进程继承了"已锁定"的锁，且没有持锁线程 → 永久死锁
# 典型场景：logging 的内部锁、数据库连接池、requests 的 session
# 解法：用 spawn/forkserver，或在 fork 后重建资源（os.register_at_fork）
```

## 3. 数据传递：pickle 是核心约束

**进程间的一切参数与返回值都要 pickle 序列化**。

```python
# ❌ 不可 pickle 的对象
lambda 表达式          # 用模块级函数或 functools.partial 代替
局部/嵌套函数
生成器、文件对象、socket、数据库连接、线程锁
类的实例若含上述成员

# ✅ 可 pickle
模块级函数与类
基本类型、list/dict/set/tuple
dataclass / pydantic 模型（成员可 pickle 时）
functools.partial(模块级函数, ...)
```

```python
# ❌ PicklingError: Can't pickle <function <lambda>>
ex.map(lambda x: x * 2, items)

# ✅
def double(x): return x * 2
ex.map(double, items)
```

**序列化成本**是多进程的主要开销：

```python
# ❌ 每个任务传一个 500MB 的 DataFrame → 序列化比计算还慢
ex.map(process, [huge_df] * 8)

# ✅ 传轻量参数，让子进程自己读数据
ex.map(process_file, ["part1.parquet", "part2.parquet", ...])

# ✅ 或用共享内存 / initializer 只传一次
```

> **面试落点**：「多进程一定比多线程快吗？」——**不一定**。
> 多进程有进程创建（几十毫秒）、数据 pickle、IPC 传输三重开销。
> **任务粒度太小（如每个任务 1ms）时，多进程会比串行还慢**。
> 经验阈值：单任务计算时间应远大于 10ms。

## 4. IPC 手段

```python
from multiprocessing import Queue, Pipe, Value, Array, Manager, shared_memory

# ① Queue —— 最常用，底层是管道 + pickle + 后台喂线程
q = Queue()
q.put(obj); q.get(timeout=1)

# ② Pipe —— 两个进程点对点，比 Queue 快
parent_conn, child_conn = Pipe()
parent_conn.send(obj); parent_conn.recv()

# ③ Value / Array —— 共享内存里的基本类型（无 pickle 开销）
counter = Value("i", 0)             # 'i'=int, 'd'=double
with counter.get_lock():            # ⚠️ 仍需加锁！
    counter.value += 1
arr = Array("d", [0.0] * 100)

# ④ Manager —— 共享的"代理对象"，支持 dict/list/Namespace
with Manager() as m:
    d = m.dict()                    # 每次访问都是一次 IPC，很慢，但最方便
    lst = m.list()
    lock = m.Lock()

# ⑤ shared_memory —— 3.8+，零拷贝共享大块内存（配合 numpy 最强）
shm = shared_memory.SharedMemory(create=True, size=1024*1024)
import numpy as np
arr = np.ndarray((128, 1024), dtype=np.float64, buffer=shm.buf)
# 子进程用 SharedMemory(name=shm.name) 挂载同一块
shm.close(); shm.unlink()
```

性能排序：`shared_memory` > `Value/Array` > `Pipe` > `Queue` > `Manager`。

## 5. Pool 的常用姿势

```python
from multiprocessing import Pool

if __name__ == "__main__":
    with Pool(processes=4, initializer=setup, initargs=(cfg,)) as pool:
        pool.map(f, items)                    # 阻塞，保序
        pool.imap(f, items)                   # 惰性，保序（大数据集用它）
        pool.imap_unordered(f, items)         # 惰性，先完成先返回 ★吞吐最高
        pool.starmap(f, [(1,2), (3,4)])       # 多参数
        r = pool.apply_async(f, (x,)); r.get(timeout=10)
        pool.map(f, items, chunksize=100)     # ★ 小任务务必设 chunksize，减少 IPC 次数
```

`initializer` 的价值：**每个 worker 进程只初始化一次**昂贵资源（模型、连接、配置），
避免每个任务都重新加载。

```python
_model = None
def setup(path):
    global _model
    _model = load_model(path)        # 每进程一次

def predict(x):
    return _model.infer(x)           # 直接用，无需传参
```

**`with Pool()` 的 `__exit__` 是 `terminate()`，不是 `close()` + `join()`**——立刻杀 worker，
不等未完成的任务。所以块内必须用阻塞式 API，或者在块内就把结果 `get()` 出来：

```python
# ❌ 提交完就离开 with，任务被静默杀掉
with Pool(2) as p:
    r = p.map_async(slow, range(4))
r.get(timeout=5)                     #=> TimeoutError

# ✅ get() 写在块内
with Pool(2) as p:
    r = p.map_async(slow, range(4))
    print(r.get())                   #=> [0, 1, 4, 9]
```

`concurrent.futures` 的池**语义正好相反**：`Executor.__exit__` 是 `shutdown(wait=True)`，
离开 `with` 会等所有已提交任务跑完。两种池混用时这是最容易踩的一脚，
详见 [[language/context-managers]] 的 §5.1。

## 6. 与线程池的接口一致性

```python
# 只需换一个类名，其余代码不变
from concurrent.futures import ThreadPoolExecutor as Executor   # I/O 密集
from concurrent.futures import ProcessPoolExecutor as Executor  # CPU 密集

with Executor(max_workers=4) as ex:
    results = list(ex.map(task, items))
```

这是**先用线程池跑通、再一行切换到进程池**的实用技巧。

## 7. 陷阱清单

```python
# ① 没有 __main__ 守卫 → fork 炸弹（macOS/Windows）
# ② 传 lambda / 局部函数 → PicklingError
# ③ 传大对象 → 序列化开销吞掉并行收益
# ④ 子进程里的异常：ProcessPoolExecutor 会在 result() 时重抛，但 traceback 可能不完整
# ⑤ 子进程崩溃（OOM/段错误）→ BrokenProcessPool，整个池不可用
# ⑥ 子进程的 print 输出可能交错、丢失 → 用 QueueHandler 集中日志
# ⑦ daemon 进程不能再创建子进程
# ⑧ fork 继承了父进程的随机数种子 → 各子进程生成相同随机数！
#    解法：子进程里 random.seed(os.getpid()) 或用 numpy 的 SeedSequence
# ⑨ 忘记 pool.close()/join() 或不用 with → 僵尸进程
#    但 with 也不是"优雅收尾"：Pool.__exit__ 是 terminate()，异步提交的任务会被杀（见 §5）
# ⑩ Ctrl+C 会同时发给所有子进程，处理不当会留下孤儿进程
```

多进程日志的标准做法：

```python
from logging.handlers import QueueHandler, QueueListener
log_q = mp.Queue()
# 子进程：logging.getLogger().addHandler(QueueHandler(log_q))
# 主进程：QueueListener(log_q, FileHandler("app.log")).start()
```

## 8. 什么时候不该用 multiprocessing

| 场景 | 更好的选择 |
|---|---|
| I/O 密集 | 线程池 / asyncio |
| 数值计算 | NumPy / Polars / Numba（底层已并行且释放 GIL） |
| 大数据处理 | Dask / Ray / Spark（分布式，还能跨机器） |
| Web 服务 | gunicorn/uvicorn 的多 worker（本质也是多进程，但由服务器管理） |
| 需要频繁共享状态 | 重新设计成无状态，或用 Redis |
| 任务粒度 < 10ms | 串行或批量化 |

> **生产实践**：Web 服务里几乎不直接用 `multiprocessing`——
> **进程级并行交给 gunicorn/uvicorn 的 worker 机制**，重计算任务交给
> **Celery / arq / Dramatiq 等任务队列**。见 [[web/fastapi-production]]。

## 相关

- [[internals/gil]] —— 为什么需要多进程
- [[concurrency/concurrency-models]] —— 选型
- [[concurrency/threading]] —— 对比与接口一致性
- [[language/modules-imports]] —— `__main__` 守卫的原理
- [[web/fastapi-production]] —— worker 模型与任务队列
- [[interview/question-bank-internals-concurrency]] —— 多进程面试题
