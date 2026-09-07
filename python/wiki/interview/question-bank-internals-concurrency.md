---
title: "题库：CPython 原理与并发"
date: 2026-08-07
tags: [面试题, GIL, GC, 内存, 并发, asyncio, 参考答案]
sources: ["interview-python-cn.md", "pep-0703-free-threading.rst"]
---

# 题库：CPython 原理与并发

**这是最能拉开差距的一块**——语言语法大家都会背，能把 GIL/GC/内存/asyncio
讲透的人很少。

## A. GIL

**Q1. 什么是 GIL？为什么存在？**

> GIL 是 **CPython 解释器的一把全局互斥锁**，保证同一时刻只有一个线程执行 Python 字节码。
> 它是 **CPython 的实现细节**，不是语言规范（Jython/IronPython 没有）。
>
> **存在的根本原因是引用计数的线程安全**：CPython 用 `ob_refcnt` 管内存，
> 多线程并发增减计数会导致对象被提前释放。三种方案里——每对象加锁（太多锁、易死锁）、
> 原子操作（缓存行争用严重）、一把大锁——CPython 选了 GIL，
> 换来**单线程高性能**和**C 扩展作者无需考虑并发安全**。

**Q2. GIL 什么时候释放？**

> ① **I/O 操作时**（文件、网络、sleep）——进入 C 层阻塞调用前主动释放；
> ② **调用释放 GIL 的 C 扩展时**（NumPy 大数组运算、hashlib、zlib）；
> ③ **每 `sys.getswitchinterval()`（默认 5ms）强制让出**；
> ④ 线程结束或阻塞在锁上时。
>
> 补充：3.2 之前是按字节码计数（每 100 条）切换，会导致 CPU 线程饿死 I/O 线程；
> 3.2 起改成按时间片 + 强制让出。

**Q3. GIL 让多线程完全没用吗？**

> **不是**。对 **I/O 密集型**任务多线程依然有效，因为 I/O 期间 GIL 被释放，
> 其它线程可以运行。Web 服务、爬虫、文件处理大多是 I/O 密集——
> 这就是 Python 做后端毫无问题的原因。
> 只有 **CPU 密集**任务才无法并行，那时要用多进程。

**Q4. GIL 能保证线程安全吗？**

> **不能**。GIL 保护的是**解释器内部状态**，不是你的业务数据。
> `counter += 1` 编译成 `LOAD / ADD / STORE` 多条字节码，中间可能被切换，
> 语言层面**从不保证**它是原子的。有共享可变状态就必须显式加锁。
>
> **加分细节**：这个教科书示例在现代 CPython 上其实**跑不出丢失**——
> 因为 GIL 只在解释器检查 `eval_breaker` 的位置切换，紧凑循环里的检查点在回跳处，
> 那三条指令之间没有。**但只要循环体里有一次函数调用（`CALL` 是检查点）就立刻复现**，
> free-threading 构建下更是必然。
> 这恰好说明：**"观察不到竞态"不等于"线程安全"——前者是实现细节，后者是语义保证。**
>
> （某些单条 C 调用完成的操作如 `list.append` 是原子的，同样**是实现细节，不要依赖**。）

**Q5. GIL 会被移除吗？**

> 正在进行中。**PEP 703 已被指导委员会接受**，CPython **3.13 起提供可选的
> free-threaded 构建**（`python3.13t`）。关键技术是
> **偏向引用计数**（owner 线程无需原子操作）、
> **不朽对象 PEP 683**（None/True/小整数的 refcount 不再修改，消除最热争用）、
> **每对象细粒度锁**。
> 代价是单线程性能有回退、C 扩展需要适配声明，短期内生产环境仍应按有 GIL 设计。
> 另一条路线是 **PEP 684 每解释器 GIL**（sub-interpreters），3.13 起有 Python 层 API。

## B. 内存与垃圾回收

**Q6. 说说 Python 的垃圾回收机制。**（背下来）

> 分三层：
>
> 1. **引用计数是主力**。每个对象有 `ob_refcnt`，归零立即释放。
>    优点是即时回收、无停顿；缺点是有计数开销，且**无法处理循环引用**。
> 2. **标记-清除处理循环引用**。只跟踪容器类对象（int/str 这类原子对象不跟踪）。
>    CPython 用**引用计数差值法**：把每个对象的计数拷贝一份，减去来自组内的引用，
>    剩下大于 0 的说明有外部引用 = 存活根，从根做可达性标记，未标记的就是垃圾环。
> 3. **分代是优化**。三代，基于"新对象更容易死"的分代假设。阈值默认 **`(700, 10, 10)`**——
>    净增容器对象超过 700 触发 gen0，每 10 次 gen0 触发一次 gen1，每 10 次 gen1 触发一次 gen2 全量。
>    （**版本注意**：**3.13 起第一个阈值改为 2000**；3.14.0–3.14.4 换成增量式 GC 时是 `(2000, 10, 0)`，
>    但**因生产环境内存压力，3.14.5 已回滚回 3.13 的分代 GC**，实测 3.14.6 又是 `(2000, 10, 10)`。
>    说 700 不算错——所有题库都是这个数——但主动补这句是明显的加分项，
>    而**说"3.14 是增量式"现在是错的**。）
>
> 补一句实践：**Python 仍会因"还持有引用"而泄漏**，常见于全局缓存、
> `lru_cache` 装饰实例方法、未取消的 asyncio Task；排查用 `tracemalloc`/`objgraph`/`memray`，
> 打破循环用 `weakref`。

**Q7. 循环引用一定会泄漏吗？带 `__del__` 呢？**

> 大多数情况会被循环 GC 回收，**但"一定"太强了**——三个条件缺一不可（见 [[internals/garbage-collection]] §4）：
>
> 1. 环的每个成员都在**本次被扫描的集合内**，且没有来自集合外的引用。
>    反例：环整体在 gen0，却被一个**自己也是垃圾**的 gen2 对象引用 → `collect(0)` 收不掉，
>    要等全量回收（**浮动垃圾**）。全局对象之间的环更极端：**运行期永远收不掉**，
>    因为 `module.__dict__` 一直引用着它们。
> 2. 每个成员的类型**正确实现了 GC 协议**。C 扩展忘了 `Py_TPFLAGS_HAVE_GC` → 环对 GC 不可见，
>    真正的永久泄漏。
> 3. **没有 finalizer 复活它**。
>
> 至于 `__del__`：Python 3.4（PEP 442）之前，循环里带 `__del__` 的对象 GC 不敢回收，
> 会进 `gc.garbage`；**3.4 起已修复**，现在 `gc.garbage` 通常永远是空的。
> 细节是 GC 会调 `tp_finalize` 并**打上 finalized 标记**，所以**`__del__` 一辈子只调一次**——
> 在 `__del__` 里复活自己只能推迟一轮，下一轮对象被静默回收。
> 但 `__del__` 仍不该用来清理资源（时机不确定、异常被忽略、PyPy 上可能不调用）——
> 用 `with` 或 `weakref.finalize`。

**Q8. 为什么 `del` 了大对象，进程 RSS 却不降？**

> 因为 CPython 的内存分配是分层的：对象被回收后内存回到 **pymalloc 的池（pool/arena）**里
> 供后续 Python 对象复用，**只有整个 1MB 的 arena 完全空闲才会还给操作系统**。
> 加上内存碎片和各类型的 freelist 缓存，RSS 更像"历史峰值"而不是当前使用量。
>
> 工程对策：**把高峰值内存的任务放到子进程**（进程退出必然归还）、流式处理、
> gunicorn 的 `--max-requests` 定期重启 worker。

**Q9. 怎么降低 Python 程序的内存占用？**

> 按收益排序：① `__slots__`（百万级小对象能省 40~60%）；
> ② 用 `array`/`numpy` 代替 `list` 存同质数值（list 存的是指针，每个 int 还是 28 字节对象）；
> ③ 生成器代替列表；④ `sys.intern` 大量重复字符串；
> ⑤ `dataclass(slots=True)` / NamedTuple；⑥ 列式存储（polars/pyarrow）。

**Q10. `sys.getsizeof([1,2,3])` 返回什么？为什么不等于三个 int 的大小之和？**

> 返回 88（3.13/64 位）= 列表对象头 56 + 3 个指针 × 8。
> **`getsizeof` 不递归**，列表只存指针，元素对象的大小不计入。
> 这也解释了为什么 numpy 快——它是连续的 C 数组，没有指针间接和对象头开销。

## C. 字节码与执行

**Q11. Python 是解释型还是编译型？**

> 先**编译成字节码**（存在 code object 和 `.pyc` 里），再由虚拟机（ceval 求值循环）解释执行，
> 模型类似 Java。完整流水线：源码 → tokens → AST（PEG parser）→ 字节码 → 求值循环。

**Q12. 为什么局部变量比全局变量快？**

> 局部变量在编译期分配**数组槽位**，用 `LOAD_FAST`（数组索引）；
> 全局变量用 `LOAD_GLOBAL`，要查模块 dict 再查 builtins dict。
> 3.11 起 `LOAD_GLOBAL` 加了内联缓存，差距缩小但仍在。

**Q13. 你了解 Python 近几个版本的性能改进吗？**（加分题）

> **3.11 的 Faster CPython 计划**是分水岭：自适应特化解释器（PEP 659，热点指令按实际类型特化）、
> 零成本异常、内联 Python 函数调用、惰性帧创建，整体比 3.10 快 10~60%。
> 3.12 加了推导式内联（PEP 709）；3.13 有实验性的 copy-and-patch JIT 和 free-threading 构建。
> **所以"升级 Python 版本"往往是性价比最高的优化。**

## D. 并发选型

**Q14. 多线程、多进程、协程怎么选？**（背下来）

> **先定位瓶颈**：
>
> - **CPU 密集** → 多进程（`ProcessPoolExecutor`），每进程独立 GIL；
>   数值计算优先 NumPy（底层释放 GIL 且向量化）。
> - **I/O 密集 + 并发量不大 / 依赖同步库** → 线程池（改造成本最低）。
> - **I/O 密集 + 高并发（数千连接）** → asyncio：协程内存 KB 级、切换纳秒级，
>   而线程是 MB 级、微秒级。
>
> **代价**：asyncio 要求全链路异步，一个同步阻塞调用就卡死整个事件循环；
> 多进程有 pickle 序列化和进程创建开销，任务粒度小于 10ms 时可能比串行还慢。
>
> 实践中常是混合：asyncio 主循环 + `asyncio.to_thread` 兜底同步库 + 进程池处理 CPU 任务。

**Q15. 多进程一定比多线程快吗？**

> 不一定。多进程有三重开销：进程创建（几十毫秒）、参数/返回值 pickle 序列化、IPC 传输。
> **任务粒度太小时会比串行还慢**。另外 I/O 密集任务用多进程是浪费——线程池就够。

**Q16. `fork` 和 `spawn` 有什么区别？**

> `fork`（Unix）复制父进程内存（CoW），快，但**与线程/锁混用极易死锁**
> （子进程继承了处于锁定状态但没有持锁线程的锁）；
> `spawn`（macOS/Windows 默认）启动全新解释器并**重新导入主模块**，慢但安全，
> 要求传递的对象可 pickle，且**必须有 `if __name__ == "__main__"` 守卫**否则会 fork 炸弹。
> **Python 3.14 起 Linux 默认从 fork 改成了 forkserver**，正是因为 fork+线程的死锁问题。

**Q17. 什么对象不能传给子进程？**

> 不可 pickle 的：**lambda、局部/嵌套函数、生成器、文件对象、socket、数据库连接、线程锁**。
> 用模块级函数或 `functools.partial` 代替 lambda。

**Q18. 死锁的四个必要条件？怎么预防？**

> 互斥、持有并等待、不可抢占、循环等待。
> 预防：**① 全局固定的加锁顺序**（最常用，比如按 `id()` 排序）；
> **② 用带超时的 acquire**；③ 一次性获取全部锁；
> **④ 最好的办法是不共享状态——用 Queue 传消息。**

## E. asyncio

**Q19. asyncio 的原理？**

> 单线程 + 事件循环 + 协作式调度。**协程是可以在中途挂起并从挂起点恢复的函数**，
> 底层复用生成器的帧保存机制。事件循环维护 ready 队列、定时器堆和 selector（epoll/kqueue），
> 不断取出就绪的回调执行；协程 `await` 一个未完成的 Future 时挂起，
> 事件循环去跑别的任务，Future 完成后再把它放回 ready 队列。
>
> **追问准备**：往下一层就是 **IO 多路复用**——`selectors` 模块封装 epoll/kqueue，
> 事件循环真正"睡着"的地方就是 `selector.select()`。
> 加分句：**asyncio 严格说不是"异步 IO"，而是"IO 多路复用 + 协程"**（同步非阻塞），
> 因为 `epoll_wait` 返回后仍要自己 `recv()` 拷贝数据；真异步是 IOCP / io_uring。
> 见 [[concurrency/io-multiplexing]]。

**Q19b. select、poll、epoll 的区别？**

> 三者都是 IO 多路复用。**select** 有 1024 的 fd 上限，每次调用都要把整个 fd 集合
> 拷进内核，返回后还要 O(n) 遍历找就绪的；**poll** 去掉了上限，但拷贝和 O(n) 遍历依旧；
> **epoll** 用 `epoll_ctl` 一次性注册（内核红黑树），`epoll_wait` 不传集合，
> 内核用回调把就绪 fd 挂到就绪链表，取就绪是 **O(1)**，还多了边沿触发模式。
> **连接数大、活跃比例低时 epoll 优势最明显**（C10K 场景）。
> epoll 是 Linux 专有，macOS/BSD 是 kqueue，Windows 是 IOCP。
> 详见 [[concurrency/io-multiplexing]]。

**Q20. `await coro` 和 `asyncio.create_task(coro)` 的区别？**（必考）

> **`await coro` 是顺序执行**——等它完成才继续；
> **`create_task` 把协程交给事件循环调度并立即返回 Task**，实现真正的并发。
>
> 关键前提：**Python 的协程是惰性的**——调用 `async def` 只创建协程对象，什么都不执行。
> 所以 `await a(); await b()` 是串行 2 秒，要并发必须 `gather(a(), b())` 或 `create_task`。
> （这和 JS 不同，JS 的 Promise 一创建就开始执行。）

**Q21. 在 `async def` 里调用同步阻塞函数会怎样？**（必考）

> **整个事件循环被阻塞，该 worker 上所有并发任务全部停滞**，不只是当前请求。
> 因为 asyncio 是协作式调度，协程不主动让出就没人能抢占它。
>
> 解法：① 用异步库（httpx / asyncpg / redis.asyncio）；
> ② `await asyncio.to_thread(blocking_fn, arg)` 卸载到线程池；
> ③ CPU 密集用 `loop.run_in_executor(process_pool, ...)`；
> ④ 在 FastAPI 里，路由写成 `def` 而不是 `async def`，框架会自动放线程池。
>
> 检测：`asyncio.run(main(), debug=True)` 会警告慢回调；生产用 `py-spy` 采样。

**Q22. `gather` 和 `TaskGroup` 的区别？**

> `gather` 在任一任务抛异常时立即抛出该异常，但**其余任务仍在后台继续运行**（孤儿任务、资源泄漏）。
> **`TaskGroup`（3.11+）是结构化并发**：退出 with 块时保证所有子任务已结束，
> 任一失败会**自动取消兄弟任务**，并把异常聚合成 `ExceptionGroup`（配合 `except*` 分类处理）。
> **新代码优先 TaskGroup。**

**Q23. asyncio 的取消是怎么回事？**

> `task.cancel()` 会**在协程的挂起点抛出 `CancelledError`**（真中断，不像 JS 的 AbortController 只是通知）。
> 协程可以捕获它做清理，但**必须重新抛出**，否则会破坏 TaskGroup/timeout 的语义。
> `CancelledError` 继承 `BaseException`（3.8+），所以 `except Exception` 不会误吞它。

**Q24. 怎么并发抓取 10 万个 URL？**

> asyncio + httpx.AsyncClient（复用连接池）+ **`asyncio.Semaphore` 限流**（比如 100 并发）
> + TaskGroup 管理生命周期 + 每个请求加超时和重试 + 分批处理避免一次创建 10 万个 Task。
> 不限流会瞬间打爆对方服务和自己的文件描述符。
> 结果流式写出（`as_completed` 或队列），不要全部堆在内存里。

**Q25. `threading.local()` 在 asyncio 里能用吗？**

> **不能**——多个协程共用同一个线程，`threading.local` 会串。
> 要用 **`contextvars.ContextVar`**：每个 Task 创建时会复制当前 context，
> 所以能正确实现"请求级上下文"（request_id、当前用户、DB session）。

## F. 综合

| 问题 | 一句话 |
|---|---|
| `concurrent.futures` 比裸 Thread 好在哪 | **异常会通过 `future.result()` 带回主线程**；接口统一，换进程池只改一个类名 |
| 能强制终止一个线程吗 | **不能**，只能协作式退出（检查 Event）；需要强制超时就用子进程 |
| 协程比线程省在哪 | 内存 KB vs MB；切换是函数调用级（纳秒）vs 内核态上下文切换（微秒） |
| uvloop 是什么 | 基于 libuv（Node 用的那个）的事件循环实现，比标准快 2~4 倍 |
| Queue 为什么线程安全 | 内部用 Condition 变量，所有操作加锁；`maxsize` 还提供背压 |

## 相关

- [[internals/gil]] / [[internals/garbage-collection]] / [[internals/memory-model]] / [[internals/bytecode-execution]]
- [[concurrency/concurrency-models]] / [[concurrency/asyncio-fundamentals]] / [[concurrency/asyncio-patterns]]
- [[bridge/async-js-vs-python]] —— 用 JS 经验加速理解
- [[interview/question-bank-language]] —— 语言核心题库
- [[interview/roadmap]] —— 复习路线
