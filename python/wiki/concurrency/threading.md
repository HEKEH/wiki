---
title: "threading 与线程安全"
date: 2026-08-07
tags: [threading, 锁, 死锁, 线程安全, Queue, 线程池]
sources: ["interview-python-cn.md", "python-cheatsheet.md"]
---

# threading 与线程安全

## 1. 基本用法

```python
import threading, time

def worker(name, n):
    for i in range(n):
        print(f"{name}: {i}")
        time.sleep(0.1)

t = threading.Thread(target=worker, args=("A", 3), daemon=True, name="worker-A")
t.start()
t.join(timeout=5)          # 等待结束
t.is_alive()

threading.current_thread().name
threading.active_count()
threading.enumerate()
threading.get_ident()      # 线程 ID
```

**daemon 线程**：主线程退出时**不等待** daemon 线程，直接杀掉。
非 daemon 线程会阻止进程退出。后台轮询、心跳一般设 `daemon=True`，
但**不要用 daemon 做需要清理的工作**（它被强杀时 finally 不执行）。

> 实践中**优先用 `ThreadPoolExecutor`**，而不是手工管理 Thread。见 [[concurrency/concurrency-models]]。

## 2. 竞态条件（race condition）

```python
counter = 0
def incr():
    global counter
    for _ in range(100_000):
        counter += 1        # 非原子：LOAD_GLOBAL / BINARY_OP / STORE_GLOBAL
```

⚠️ **但这个经典示例在现代 CPython 上跑不出丢失**（3.8 和 3.13 实测都是精确的 400000）。
原因值得搞懂——它恰恰是理解 GIL 切换时机的最好入口：

**GIL 只在解释器检查 `eval_breaker` 的位置才可能切换**，而在一个紧凑循环里，
检查点在 `JUMP_BACKWARD`（循环回跳），**`LOAD_GLOBAL → BINARY_OP → STORE_GLOBAL`
这三条之间没有检查点**，于是读-改-写"碰巧"跑完了。

只要循环体里出现**函数调用**（`CALL` 是检查点），竞态立刻显现：

```python
counter = 0
def bump(v): return v + 1

def incr():
    global counter
    for _ in range(200_000):
        counter = bump(counter)     # ← CALL 是 eval_breaker 检查点

# 4 线程 × 20 万，实测三次丢失量：401608 / 209196 / 214000
# 期望 800000，实际只有 30~70 万 —— 竞态非常显著
```

> **面试落点**（这段能明显拉开档次）：
> 「`counter += 1` **不是原子操作**——它是三条字节码，语言层面**从不保证**中间不被打断。
> 有意思的是在现代 CPython 上这个经典示例反而跑不出丢失，因为 **GIL 只在
> `eval_breaker` 检查点切换**，而紧凑循环里的检查点在回跳处，三条指令之间没有。
> 但**只要循环体里有函数调用就立刻复现**，free-threading 构建下更是必然。
> **所以这恰恰说明了为什么不能靠观察来判断线程安全——它是实现细节。
> 有共享可变状态就加锁。**」

任何 **read-modify-write** 都需要加锁——见 [[internals/gil]]。

## 3. 同步原语

### Lock（互斥锁）

```python
lock = threading.Lock()

with lock:                  # ✅ 永远用 with，异常时也能释放
    counter += 1

# 等价于
lock.acquire()
try:
    counter += 1
finally:
    lock.release()

lock.acquire(timeout=1.0)   # 返回 bool，避免永久阻塞
```

### RLock（可重入锁）

同一线程可以**多次获取**，用于递归或"公开方法调用内部方法"的场景。

```python
class Account:
    def __init__(self):
        self._lock = threading.RLock()
        self._balance = 0
    def deposit(self, x):
        with self._lock:
            self._balance += x
    def transfer(self, other, x):
        with self._lock:        # ← 已持有
            self.deposit(-x)    # ← 再次获取；用 Lock 会死锁，RLock 不会
            other.deposit(x)
```

### 其它原语

```python
sem = threading.Semaphore(5)          # 限制并发数（如同时最多 5 个连接）
with sem: ...

ev = threading.Event()                # 一次性信号（≈ Promise）
ev.set() / ev.clear() / ev.wait(timeout=5) / ev.is_set()

cond = threading.Condition()          # 条件变量：等待某个状态成立
with cond:
    cond.wait_for(lambda: len(queue) > 0)
    item = queue.pop()
with cond:
    queue.append(x)
    cond.notify()                     # 或 notify_all()

barrier = threading.Barrier(3)        # 等所有 3 个线程都到达
barrier.wait()

local = threading.local()             # 线程局部存储（每线程独立的属性）
local.request_id = "abc"              # 常用于日志上下文、DB session
```

> `threading.local()` 在 asyncio 里**不适用**（多个协程共用一个线程）——
> 异步场景要用 `contextvars.ContextVar`。见 [[concurrency/asyncio-patterns]]。

## 4. 死锁

**四个必要条件**：互斥、持有并等待、不可抢占、循环等待。

```python
la, lb = threading.Lock(), threading.Lock()

def t1():
    with la:
        time.sleep(0.1)
        with lb: ...      # 等 lb

def t2():
    with lb:
        time.sleep(0.1)
        with la: ...      # 等 la  → 死锁
```

**四种预防**：

```python
# ① 全局固定的加锁顺序（最常用）—— 按 id() 或业务主键排序
def transfer(a, b, amt):
    first, second = sorted([a, b], key=id)
    with first._lock, second._lock:
        ...

# ② 用超时代替无限等待
if lock.acquire(timeout=1):
    try: ...
    finally: lock.release()
else:
    raise TimeoutError

# ③ 一次性获取所有锁（避免"持有并等待"）
# ④ 干脆不共享状态 —— 用 Queue 传消息（最推荐）
```

> **面试落点**：死锁的四个必要条件 + 至少两种预防手段（**固定加锁顺序**和**超时**）。
> 补一句「最好的办法是通过消息传递避免共享状态」体现设计品味。

## 5. `queue` —— 线程安全的通信

**生产者-消费者是线程编程的标准范式**，比共享变量 + 锁更不容易出错。

```python
from queue import Queue, LifoQueue, PriorityQueue, Empty, Full

q = Queue(maxsize=100)        # maxsize 提供背压（backpressure）

def producer():
    for item in source:
        q.put(item)           # 满了会阻塞 → 天然限流
    q.put(None)               # 哨兵，通知结束

def consumer():
    while True:
        item = q.get()
        try:
            if item is None:
                q.put(None)   # 传递哨兵给其它消费者
                break
            handle(item)
        finally:
            q.task_done()

q.join()                      # 等待所有 task_done
q.get(timeout=1)              # 超时抛 Empty
q.put_nowait(x)               # 满则抛 Full
```

`Queue` 内部用 `Condition` 实现，**所有操作都是线程安全的**。

## 6. 哪些操作是"原子"的

CPython 里由**单条字节码 / 单次 C 调用**完成的操作不会被 GIL 切换打断：

```python
# ✅ 原子
lst.append(x)          lst.extend(it)        lst.pop()
d[k] = v               d.update(other)       d.setdefault(k, v)
x = y                  x = obj.attr
sorted 的整体调用       next(itertools.count 实例)

# ❌ 非原子（read-modify-write / check-then-act）
i += 1
d[k] += 1
if k not in d: d[k] = []
x = lst[0]; lst[0] = x + 1
```

**不要依赖这个清单写代码**——它是实现细节，free-threading 构建下也在变。
**有共享可变状态就加锁**，这是唯一稳妥的规则。

## 7. 线程与异常

线程里的未捕获异常**不会传播到主线程**，只会打印到 stderr。

```python
def worker():
    raise ValueError("boom")

t = threading.Thread(target=worker)
t.start(); t.join()          # 主线程完全不知道出错了！

# ✅ 用 ThreadPoolExecutor，异常会在 future.result() 时重新抛出
with ThreadPoolExecutor() as ex:
    fut = ex.submit(worker)
    fut.result()             # ValueError: boom

# 或者设置全局钩子（3.8+）
threading.excepthook = lambda args: log.error("线程异常", exc_info=args.exc_value)
```

> **面试落点**：这是"为什么应该用 `concurrent.futures` 而不是裸 Thread"的**最有力理由**——
> Future 帮你把异常带回主线程。

## 8. 实战模式

```python
# ① 带限流的并发抓取
sem = threading.Semaphore(10)
def fetch(url):
    with sem:
        return requests.get(url)

# ② 单次初始化（线程安全的懒加载）
_lock = threading.Lock()
_instance = None
def get_instance():
    global _instance
    if _instance is None:            # 双重检查锁定
        with _lock:
            if _instance is None:
                _instance = expensive_init()
    return _instance
# 更简单：用模块级变量（模块只导入一次）或 functools.cache

# ③ 后台定时任务
def heartbeat(stop_event: threading.Event):
    while not stop_event.wait(timeout=30):    # wait 兼具 sleep 和"可中断"
        send_ping()
stop = threading.Event()
threading.Thread(target=heartbeat, args=(stop,), daemon=True).start()
# 关闭时：stop.set()

# ④ 超时执行
with ThreadPoolExecutor(1) as ex:
    try:
        result = ex.submit(slow_fn).result(timeout=5)
    except TimeoutError:
        ...     # ⚠️ 注意：任务并不会真的被终止，只是不等它了
```

> ⚠️ **Python 无法安全地强制终止线程**（没有 `Thread.kill()`）。
> 只能靠协作式退出（检查 Event 标志）。需要强制超时就用**子进程**。

## 相关

- [[internals/gil]] —— GIL 与线程安全的关系
- [[concurrency/concurrency-models]] —— 选型
- [[concurrency/multiprocessing]] —— 需要真并行时
- [[concurrency/asyncio-patterns]] —— asyncio 的对应原语与 contextvars
- [[stdlib/stdlib-essentials]] —— queue / concurrent.futures
- [[interview/question-bank-internals-concurrency]] —— 线程面试题
