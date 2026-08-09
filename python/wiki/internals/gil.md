---
title: "GIL 全局解释器锁"
date: 2026-08-07
tags: [GIL, 并发, 多线程, free-threading, PEP703, 性能]
sources: ["pep-0703-free-threading.rst", "interview-python-cn.md"]
---

# GIL（Global Interpreter Lock）

Python 面试**出现频率最高的单个知识点**。多数人能说出"GIL 让多线程无法并行"，
但答不出**为什么存在、什么时候释放、3.13 之后怎么变了**——那才是区分度所在。

## 1. GIL 是什么

> **GIL 是 CPython 解释器级别的一把互斥锁，保证同一时刻只有一个线程在执行 Python 字节码。**

注意三个限定：

- **CPython 的实现细节**，不是 Python 语言规范。Jython、IronPython 没有 GIL。
- 锁的是**字节码执行**，不是"所有操作"——C 扩展、I/O 期间会释放。
- 是**解释器级**的一把锁，不是每个对象一把锁。

## 2. 为什么存在（核心考点）

**根因：引用计数的线程安全。**

CPython 用引用计数管内存（[[internals/cpython-object-model]]）。
`ob_refcnt++` / `--` 在多线程下是竞态操作：

```text
线程 A 读 refcnt=2 ──┐
线程 B 读 refcnt=2 ──┤ 两者都写回 3（本应是 4）
                     └─→ 计数偏低 → 对象被提前释放 → 段错误
```

三种解法：

| 方案 | 代价 |
|---|---|
| **每个对象一把锁** | 锁数量巨大、易死锁、单线程性能暴跌 |
| **原子操作 / 无锁计数** | 原子指令有开销，跨核缓存行争用（cache line ping-pong）严重 |
| **一把全局大锁（GIL）** | 单线程零额外开销、实现简单、**C 扩展作者不必考虑线程安全** ✅ |

Guido 的取舍：1992 年多核还不普及，GIL 让 CPython **单线程更快**、
让海量 C 扩展生态（NumPy 时代之前的 Python 靠它起家）**编写简单**。
后来多核普及了，但 GIL 已成为整个 C API 的隐含契约，移除它 = 打破所有 C 扩展。

> **面试落点**：能说出「GIL 的存在是为了保护引用计数等解释器内部状态，
> 并让 CPython 的 C 扩展可以不考虑并发安全；代价是牺牲了多核并行」——就答到点了。
> 加一句「它保护的是**解释器状态**，不是**你的业务数据**」更好。

## 3. GIL 什么时候释放

```text
① 执行 I/O 时（文件、网络、socket、sleep）—— C 层调用前主动释放，返回后重新获取
② 调用释放 GIL 的 C 扩展时（NumPy 的大数组运算、hashlib、zlib、部分数据库驱动）
③ 每隔 sys.getswitchinterval()（默认 5ms）由解释器强制让出
④ 线程结束 / 阻塞在锁上时
```

```python
import sys
sys.getswitchinterval()      #=> 0.005    单位秒
sys.setswitchinterval(0.001) # 调小 → 切换更频繁，响应更好但吞吐更低
```

> Python 3.2 之前的 GIL 是**按字节码计数**切换（每 100 条），会导致 CPU 密集线程
> 饿死 I/O 线程（David Beazley 的著名分析）。3.2 起 Antoine Pitrou 改为**按时间片**，
> 并加了"强制让出"机制。这段历史在深度面试中提到会很加分。

## 4. 后果：什么受影响，什么不受影响

```python
import time, threading
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor

def cpu_task(n):                 # CPU 密集
    return sum(i*i for i in range(n))

def io_task(url):                # I/O 密集
    return requests.get(url).text
```

| 任务类型 | 多线程效果 | 原因 |
|---|---|---|
| **CPU 密集** | ❌ **无加速，甚至更慢** | 只有一个线程能跑字节码，还多了切换开销 |
| **I/O 密集** | ✅ **显著加速** | 阻塞在 I/O 时 GIL 被释放，其它线程可以跑 |
| **NumPy / pandas 大运算** | ✅ 部分加速 | 底层 C 代码释放了 GIL |
| **多进程** | ✅ 真并行 | 每个进程有独立解释器和独立 GIL |

实测对比（4 核机器上典型结果）：

```python
N = 10_000_000
# 单线程        : 1.00x
# 2 线程        : ~1.0x  甚至 0.9x（切换开销）
# 2 进程        : ~1.9x
# 2 线程 + I/O  : ~2.0x
```

> **面试落点**：「GIL 让多线程完全没用吗？」——**不是**。
> 对 **I/O 密集型**任务多线程依然有效，因为 I/O 期间 GIL 被释放。
> Web 服务、爬虫、文件处理大多是 I/O 密集，这也是为什么 Python 做后端毫无问题。

## 5. GIL 不保证你的代码线程安全

这是最容易被误解的点。GIL 保护的是**解释器内部状态**，不是**你的逻辑**。

```python
counter = 0
def worker():
    global counter
    for _ in range(100_000):
        counter += 1          # ❌ 不是原子操作：三条字节码
```

因为 `counter += 1` 编译成多条字节码，**中间可能被切换**：

```python
import dis
dis.dis("counter += 1")
#   LOAD_NAME counter     ← 原则上这里可能被切换，另一个线程读到旧值
#   LOAD_CONST 1
#   BINARY_OP  +=
#   STORE_NAME counter
```

⚠️ **但这个教科书示例在现代 CPython 上实测跑不出丢失**（3.8 与 3.13 都精确得到 400000）。

原因是 **GIL 只在解释器检查 `eval_breaker` 的位置才可能切换**——紧凑循环里的检查点在
`JUMP_BACKWARD`，而上面三条指令之间没有检查点。只要循环体里加一次**函数调用**
（`CALL` 是检查点），竞态立刻显现：

```python
def bump(v): return v + 1
def worker():
    global counter
    for _ in range(200_000):
        counter = bump(counter)
# 4 线程实测：期望 800000，三次实际丢失 401608 / 209196 / 214000
```

> **这个反直觉的实测结果本身就是最好的论据**：
> **"能不能观察到竞态"是实现细节，"是不是线程安全"是语义问题。**
> 语言从不保证 `+=` 原子；free-threading 构建下必然竞态；
> 换个写法、换个版本、加一次函数调用就会暴露。
> **有共享可变状态就加锁，不要靠跑一遍没出错来判断。**

**哪些操作是原子的**（依赖实现，不要背，但要知道存在这个区别）：

```python
lst.append(x)        # 原子（单条 C 层调用）
d[k] = v             # 原子
x = y                # 原子
lst.sort()           # 原子
d.setdefault(k, [])  # 原子

i += 1               # ❌ 非原子
lst[i] = lst[j] + 1  # ❌ 非原子
if k not in d: d[k]=v  # ❌ 非原子（check-then-act 竞态）
```

正确做法永远是**显式加锁**：

```python
lock = threading.Lock()
with lock:
    counter += 1

# 或用天生线程安全的结构
from queue import Queue
q = Queue()          # 内部有锁
import itertools
counter = itertools.count()   # next() 是原子的
```

见 [[concurrency/threading]]。

## 6. 绕过 GIL 的五条路

| 方案 | 适用 | 说明 |
|---|---|---|
| **multiprocessing / ProcessPoolExecutor** | CPU 密集 | 每进程独立 GIL；代价是 IPC 序列化和内存 |
| **asyncio** | 高并发 I/O | 单线程协程，根本不与 GIL 竞争 |
| **C 扩展 / NumPy / Cython `nogil`** | 数值计算 | 在 C 层释放 GIL 后并行 |
| **Rust 扩展（PyO3）/ Mojo** | 新项目 | 同上，现代选择 |
| **free-threading 构建（3.13+）** | 未来 | 真正移除 GIL，见下 |
| 换实现 | 特定场景 | Jython/IronPython 无 GIL，但落后主线 |

## 7. PEP 703：无 GIL 的 CPython（3.13+）

Sam Gross 的 PEP 703 已被指导委员会**接受**，3.13 起提供**可选的 free-threaded 构建**
（`python3.13t`，编译时 `--disable-gil`）。

关键设计：

- **偏向引用计数（biased reference counting）**：区分"对象的所有者线程"与其它线程，
  owner 线程的计数操作无需原子指令。
- **不朽对象（immortal objects, PEP 683）**：`None`/`True`/小整数/interned 字符串的
  refcount 设为特殊值，永不修改 → 消除最热的计数争用。
- **每对象锁**：list/dict 等内建容器加细粒度锁，保持基本原子性。
- **线程安全的 pymalloc**（mimalloc）。

现状与代价：

```console
$ python3.13t -c "import sys; print(sys._is_gil_enabled())"
False
```

- 单线程性能**有回退**（3.13 时约 ~10-20%，3.14 已大幅改善，目标是接近持平）。
- **C 扩展需要适配**（声明 `Py_mod_gil = Py_MOD_GIL_NOT_USED`），生态迁移需要数年。
- 3.13/3.14 阶段仍是**实验性**，默认构建仍有 GIL。

> **面试落点**（2026 年的加分回答）：
> 「GIL 正在被移除。PEP 703 已被接受，CPython 3.13 起提供可选的 free-threaded 构建，
> 靠偏向引用计数 + 不朽对象 + 每对象锁来替代全局锁。但单线程有性能回退、C 扩展生态需要迁移，
> 短期内生产环境仍应按有 GIL 来设计——CPU 密集用多进程，I/O 密集用 asyncio 或线程池。」

另一条并行路线是 **PEP 684：每解释器 GIL（sub-interpreters）**，3.12 已在 C API 层实现，
3.13 通过 `interpreters` 模块（3.14 转正为 `concurrent.interpreters`）暴露给 Python 层——
同一进程内多个解释器各有自己的 GIL，比多进程轻量、比线程隔离。

## 8. 面试标准答题模板

> **问：什么是 GIL？**
>
> 1. **定义**：CPython 解释器的一把全局互斥锁，同一时刻只有一个线程执行 Python 字节码。
>    是 **CPython 的实现细节**，不是语言规范。
> 2. **为什么**：CPython 用引用计数管内存，多线程改计数需要保护。GIL 是"一把大锁"方案，
>    换来单线程高性能和 C 扩展的简单性。
> 3. **影响**：CPU 密集型多线程无法并行；I/O 密集型不受影响，因为 I/O 时 GIL 会释放。
> 4. **注意**：GIL **不保证业务代码线程安全**，`i += 1` 仍需加锁。
> 5. **对策**：CPU 密集 → 多进程 / C 扩展；I/O 密集 → 线程池 / asyncio。
> 6. **未来**：PEP 703 free-threading，3.13 起可选构建。

## 相关

- [[internals/cpython-object-model]] —— 引用计数是 GIL 的根因
- [[internals/garbage-collection]] —— 三者的关系
- [[concurrency/concurrency-models]] —— 三种并发模型的选型
- [[concurrency/threading]] —— 线程安全与锁
- [[concurrency/multiprocessing]] —— 绕过 GIL 的主流方案
- [[concurrency/asyncio-fundamentals]] —— 单线程高并发
- [[interview/question-bank-internals-concurrency]] —— GIL 面试题
- [[sources/cpython-docs]] —— 来源：PEP 703
