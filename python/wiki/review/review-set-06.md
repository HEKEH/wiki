---
title: "复习题组 06 —— 三种并发模型与选型自测卷"
date: 2026-09-07
tags: [面试题, 复习, 并发, 并行, 线程, 进程, 协程, concurrent-futures]
sources: []
---

# 复习题组 06 —— 三种并发模型与选型自测卷

主题页是 [[concurrency/concurrency-models]]，对应 [[interview/roadmap]] **第 3 周 Day 1**。
本卷 **45 题**（选择 18 / 判断 15 / 填空 12），覆盖该页全部七节：概念辨析、三模型对比表、
选型决策树、四种写法的实测数字、`concurrent.futures` 接口、混合模式、答题模板。

**用法**：闭卷作答，把答案直接写在每题下方的 `**答：**` 后面。
判断题写 `✅` / `❌`（判 ❌ 的请补一句正确说法）。全部写完后交给我批改——
我会逐题给对错、补解析和面试落点，并把批改记录追加到本页末尾。

**不确定的题也要写**，并在答案后加 `(?)`，这样能区分「真会」和「蒙对」。

| 分区 | 题号 | 覆盖 |
|---|---|---|
| 选择题 | 1–18 | 概念定义、三模型量级对比、决策树选型、`concurrent.futures` API |
| 判断题 | 1–15 | 最容易记反的机制性结论（GIL 与 I/O、顺序语义、事件循环阻塞） |
| 填空题 | 1–12 | 术语、默认值公式、量级数字、惯用写法 |

**高频错点预告**（不看答案也能自查的四个坑）：
`ex.map` 与 `as_completed` 的顺序语义、两个 Executor 的 `max_workers` 默认值、
`to_thread` 是线程不是进程、asyncio 对 CPU 密集是「负收益」而不是「零收益」。

---

## 一、选择题（18 题，除标注多选外均为单选）

**1.** 关于并发（concurrency）与并行（parallelism），说法正确的是：

A. 单核 CPU 上无法实现并发　B. 并发指多个任务交替推进、逻辑上同时，单核也能并发
C. 并行是并发的一种实现手段，两者可以互换使用　D. Python 的多线程既并发也并行

**答：** B

**2.** 受 GIL 限制，Python 多线程**对 Python 字节码而言**：

A. 有并发也有并行　B. 有并发但没有并行　C. 没有并发但有并行　D. 线程数超过核数时才有并行

**答：** B

**3.** 三种模型中，调度者是**事件循环**、调度方式为**协作式**的是：

A. threading　B. multiprocessing　C. asyncio　D. 三者都是抢占式

**答：** C

**4.** 按**切换成本从低到高**排列，正确的是：

A. threading < asyncio < multiprocessing　B. asyncio(~ns) < threading(~µs) < multiprocessing(~ms)
C. asyncio(~µs) < multiprocessing(~ms) < threading(~s)　D. 三者同一量级

**答：** B

**5.** 单个执行单位的内存开销，量级对应正确的是：

A. 协程 ~KB / 线程 ~8 MB 栈（虚拟）/ 进程 ~10–50 MB
B. 协程 ~MB / 线程 ~KB / 进程 ~MB
C. 三者都是 KB 级，差别只在切换成本
D. 协程 ~KB / 线程 ~KB / 进程 ~MB

**答：** A

**6.** 关于可扩展数量，说法**错误**的是：

A. 线程数量通常能开到数百~数千　B. 进程数一般 ≈ CPU 核数
C. 协程可以开到数万~数十万　D. 协程数量的上限同样受 CPU 核数约束

**答：** B

**7.** 三种模型的数据共享方式，对应正确的是：

A. threading 需 IPC / multiprocessing 直接共享 / asyncio 需加锁
B. threading 共享地址空间（需加锁）/ multiprocessing 靠 pickle 序列化或共享内存 / asyncio 直接共享
C. 三者都需要 pickle 序列化
D. threading 与 multiprocessing 都共享地址空间，只有 asyncio 隔离

**答：** B

**8.** asyncio 中协程之间共享数据「不必用锁保护」，根本原因是：

A. 事件循环内部替你加了锁　B. 协程运行在单线程里，只在 `await` 点让出，不会被字节码级抢占
C. GIL 保证了所有操作原子　D. 协程之间的变量天然隔离

**答：** B

**9.** 一段**纯 Python** 写的 CPU 密集计算（大量循环 + 算术）要提速，首选：

A. `ThreadPoolExecutor`　B. `ProcessPoolExecutor`　C. `asyncio.gather`　D. 增加 `max_workers` 到 100

**答：** B

**10.** 同样是 CPU 密集，如果任务是**大规模数值计算**，决策树给出的首选是：

A. 多进程　B. NumPy / Polars（底层已释放 GIL 并向量化）
C. asyncio　D. 直接换语言

**答：** B

**11.** 场景：抓取几十个外部 API，只能用同步的 `requests`，团队不想大改代码。最合适的是：

A. `asyncio` + `httpx`（全链路改造）　B. `ThreadPoolExecutor`
C. `ProcessPoolExecutor`　D. 串行 + 加机器

**答：** B

**12.** 场景：网关服务要维持**数千个**长连接做转发，生态里有成熟的 async 客户端。最合适的是：

A. threading，开 5000 个线程　B. multiprocessing，开 5000 个进程
C. asyncio　D. `ThreadPoolExecutor(max_workers=5000)`

**答：** C

**13.** 判断一个慢任务是 CPU 密集还是 I/O 密集，正确的做法是：

A. 看代码里有没有 `for` 循环　B. 运行时看 CPU 占用：接近 100%（单核）→ CPU 密集，很低但耗时长 → I/O 密集
C. 看内存占用　D. 看线程数

**答：** B

**14.** 4 个 `cpu_bound` 任务，串行 4.0s，`ThreadPool(4)` 反而变成 4.2s。原因是：

A. 线程池初始化慢　B. GIL 让字节码无法并行，多出来的是线程切换开销
C. 任务被重复执行了　D. `max_workers` 设小了

**答：** B

**15.** 「asyncio 能加速 CPU 密集任务吗？」标准答案是：

A. 能，和多进程效果相当　B. 能加速约 2 倍
C. 完全不能，而且更糟——不 `await` 的协程会卡死整个事件循环，其它请求全部超时
D. 不能加速但也无害，等同于串行

**答：** D

**16.** 在一台 8 核机器上执行 `ThreadPoolExecutor()`（不传 `max_workers`），worker 数是：

A. 1　B. 8　C. 12　D. 32

**答：** B

**17.** 关于 `ex.map(fn, items)` 与 `submit` + `as_completed(futures)`，说法正确的是（**多选**）：

A. `map` 的结果顺序与输入顺序一致
B. `as_completed` 按「谁先完成谁先产出」的顺序迭代
C. `map` 里任务抛的异常在**取结果时**才被抛出
D. `as_completed` 会保证按提交顺序产出 future

**答：** A B

**18.** 向 `ProcessPoolExecutor` 提交一个 `lambda`，实际结果是：

A. 正常执行　B. 静默跳过　C. 抛 `PicklingError`（函数必须是可 pickle 的模块级函数）　D. 自动降级为线程执行

**答：** C

---

## 二、判断题（15 题，写 ✅ 或 ❌；判 ❌ 的请补一句正确说法）

**1.** 并发必须有多核 CPU 才能发生，单核机器上只有串行。

**答：** 错

**2.** Python 线程在等待 I/O 时会释放 GIL，所以线程池确实能加速 I/O 密集任务。

**答：** 对

**3.** multiprocessing 的多个进程共享同一地址空间，直接读写同一个全局变量即可通信。

**答：** 错

**4.** 真正的并行只能靠多进程，或者靠会主动释放 GIL 的 C 扩展。

**答：** 对

**5.** `ThreadPoolExecutor` 和 `ProcessPoolExecutor` 的接口完全一致，从线程切到进程只需改一个类名。

> ⚠️ **出题缺陷**（2026-09-13 批改时发现）：这是个前半句真、后半句假的复合命题，
> 两种判法都讲得通。重出时应拆成两题：
> ① 「两种 Executor 的 `submit` / `map` / `shutdown` / `Future` 接口签名完全一致」（✅）
> ② 「把 `ThreadPoolExecutor` 换成 `ProcessPoolExecutor` 只需改类名，其余代码不用动」（❌）

**答：** 错

**6.** asyncio 是单线程协作式调度，协程不会在字节码级被抢占，因此共享数据不需要 `threading.Lock` 保护。

**答：** 对

**7.** `ThreadPoolExecutor` 的 `max_workers` 不应超过 CPU 核数，否则会因为切换开销变慢。

**答：** 错

**8.** `ProcessPoolExecutor` 的 `max_workers` 默认等于 `cpu_count()`，一般不建议超过它。

**答：** 错

**9.** 对 CPU 密集任务，asyncio 虽然不能加速，但至少不会比串行更差。

**答：** 错

**10.** `ex.map()` 返回的结果是「谁先算完谁在前」，所以不能依赖它的顺序。

**答：** 错

**11.** `as_completed()` 按 future 的提交顺序产出结果。

**答：** 错

**12.** `asyncio.to_thread(fn, arg)` 会新建一个**进程**来执行同步函数 `fn`。

**答：** 错

**13.** 在 FastAPI 的 `async def` 路由里直接调用同步阻塞的数据库驱动，会阻塞整个事件循环，导致其它请求全部变慢或超时。

**答：** 对

**14.** FastAPI 会把 `def` 定义的路由函数放进线程池执行，而 `async def` 路由直接在事件循环里跑。

**答：** 对

**15.** 线程切换要陷入内核态，成本比协程切换高好几个数量级；进程创建更贵，在毫秒量级。

**答：** 对

---

## 三、填空题（12 题）

> **批改于 2026-09-13　得分 9.5 / 12**。批注直接跟在每题答案下方。
> 实测环境 **CPython 3.11.9 / Windows / 8 核**。第 10 题**题目有缺陷**，见该题批注。

**1.** Rob Pike 的名言：*"Concurrency is about ______________ lots of things at once.
Parallelism is about ______________ lots of things at once."*

**答：** dealing with  doing

> ✅ **1/1**。两空全对。
> 这句话的分量在于：*dealing with* 是「应付得过来」——**结构**问题；
> *doing* 是「同时干」——**执行**问题。所以并发是设计，并行是能力。
> 中文答题时别翻成「处理 / 做」，直接引英文原句更有说服力。

**2.** 三种模型中，threading 和 multiprocessing 的调度者都是 ______________（______________式调度），
asyncio 的调度者是 ______________（______________式调度）。

**答：** 操作系统 抢占式 用户 协作

> ⚠️ **0.5/1**。第 1、2、4 空对；**第 3 空应为「事件循环」，不是「用户」**。
>
> 你想说的多半是「**用户态**调度」——方向没错（协程切换确实不进内核），
> 但题目问的是**调度者**（谁在做调度决策），答案必须是个**主体**：
>
> | | 调度者（主体） | 在哪一态 | 方式 |
> |---|---|---|---|
> | threading / multiprocessing | **操作系统内核** | 内核态 | 抢占式 |
> | asyncio | **事件循环**（`while True` 取就绪回调） | 用户态 | 协作式 |
>
> 面试时这个词要咬准：说「事件循环调度」考官知道你懂机制，
> 说「用户调度」会被追问「具体是谁？」。顺带一提，「用户态 vs 内核态」正是
> 第 11 题里切换成本差三个数量级（ns vs µs）的**原因**——两道题是一条线。

**3.** 8 核机器上 `ThreadPoolExecutor()` 的默认 `max_workers`，公式是 ______________，数值是 ______________。

**答：** min(32, cpu + 4) 12

> ✅ **1/1**。公式和数值都对。
>
> **这是本卷最值得表扬的一处**：同一个知识点在选择题 #16（答 8）和判断题 #8（答错）
> 连续错了两次，到填空题已经完全纠正。原先列在回炉清单第一条的项目可以划掉了。
>
> 补一个实现细节：3.13 起源码里从 `os.cpu_count()` 换成了
> `os.process_cpu_count()`——后者尊重 CPU 亲和性设置（容器里限了 2 核时返回 2 而非宿主核数），
> 公式本身没变。

**4.** `ProcessPoolExecutor()` 的默认 `max_workers` 是 ______________。

**答：** cpu数量

> ✅ **1/1**。即 `os.cpu_count()`。
> 写代码时留意：它**可能返回 `None`**（罕见平台），标准库里写的是
> `os.cpu_count() or 1`。容器环境同第 3 题的提醒——`cpu_count()` 读的是宿主机核数，
> K8s 里给了 `limits.cpu: 2` 它照样返回 64，于是池子开 64 个进程抢 2 个核。
> 生产代码应显式传 `max_workers`，别依赖默认值。

**5.** 判断瓶颈类型：运行时单核 CPU 占用接近 100% → ______________ 密集；
CPU 占用很低但耗时很长 → ______________ 密集。想看时间花在哪个函数，可以用命令
`______________ top --pid <pid>`。

**答：** cpu IO py-spy

> ✅ **1/1**。三空全对。
> `py-spy` 的价值在于**不用改代码、不用重启、不侵入目标进程**——
> 线上服务 CPU 飙高时直接 `py-spy top --pid <pid>` 就能看热点函数，
> 这是比 `cProfile` 更常用的生产手段。面试聊性能时报出这个工具名是加分项。
> 另外两个常见搭配：`py-spy dump --pid` 看**当前每个线程卡在哪一行**（排查死锁神器）、
> `py-spy record -o out.svg` 出火焰图。

**6.** Python 3.9+ 把一个同步阻塞函数丢到线程池执行、并在协程里等待它的**最简写法**是
`data = await ______________(blocking_fn, arg)`。

**答：** asyncio.to_thread

> ✅ **1/1**。
> 两个容易被追问的点：① 它用的是**事件循环的默认 executor**
> （`min(32, cpu+4)`，和第 3 题同一个公式），大量并发的阻塞调用会在这里排队；
> ② 它会自动传播 `contextvars`，而手写 `run_in_executor` 不会——
> 这是 3.9 引入它的主要动机之一（链路追踪、请求 ID 能跟着进线程）。

**7.** 要在协程里用**自己指定**的 executor 跑 CPU 密集函数，写法是：
先 `loop = asyncio.______________()`，再 `result = await loop.______________(pool, cpu_heavy, data)`。

**答：** get_running_loop run_in_executor

> ✅ **1/1**。两空全对，而且 `get_running_loop` 选得准。
> 对照 `asyncio.get_event_loop()`：后者在**没有运行中的循环**时会隐式创建一个，
> 语义含糊，3.10 起在协程外调用会发 `DeprecationWarning`，3.12 起更严格。
> **协程内部一律用 `get_running_loop()`**（3.7+），没有循环就直接抛 `RuntimeError`，
> 失败得干脆。
>
> **另一个坑：`run_in_executor` 不接受关键字参数**。看四个入口的签名就明白了——
>
> ```python
> ThreadPoolExecutor.submit(self, fn, /, *args, **kwargs)     # ✅ 有 **kwargs
> asyncio.to_thread(func, /, *args, **kwargs)                  # ✅ 有 **kwargs
> loop.run_in_executor(self, executor, func, *args)            # ❌ 只有 *args
> ThreadPoolExecutor.map(self, fn, *iterables, timeout, chunksize)  # ❌ 只有 *iterables
> ```
>
> `run_in_executor` 只往后收**位置参数**，没有 `**kwargs` 兜底。
> 于是你写的关键字会被当成传给 **`run_in_executor` 自己**的参数：
>
> ```python
> def greet(name, greeting="Hello", punct="!"):
>     return f"{greeting}, {name}{punct}"
>
> await asyncio.to_thread(greet, "Kai", greeting="你好")
> #=> '你好, Kai!'                                   ← to_thread 可以
>
> await loop.run_in_executor(None, greet, "Kai", greeting="你好")
> #=> TypeError: BaseEventLoop.run_in_executor() got an unexpected keyword argument 'greeting'
> ```
>
> **报错信息是这个坑最难受的地方**：它说的是 `run_in_executor()` 收到了意外的关键字参数，
> 而不是 `greet()`——第一次遇到时很容易以为是自己调用 `run_in_executor` 的姿势错了，
> 其实问题出在「`greeting` 本该转发给 `greet`，但没有通道」。
>
> 三种写法：
>
> ```python
> # ❌ 不行
> await loop.run_in_executor(None, greet, "Kai", greeting="你好")
>
> # ⚠️ 能跑，但必须按顺序把中间的参数全补上，改默认值顺序就会静默错位
> await loop.run_in_executor(None, greet, "Kai", "你好")
> #=> '你好, Kai!'
>
> # ✅ 正解：用 partial 先把关键字「绑」进去，再交给 run_in_executor
> from functools import partial
> fn = partial(greet, greeting="你好", punct="。")
> await loop.run_in_executor(None, fn, "Kai")
> #=> '你好, Kai。'
> ```
>
> `partial(greet, greeting="你好")` 返回的是一个**新的可调用对象**，
> 它记住了那个关键字；之后只需要用位置参数调用它——正好满足 `run_in_executor` 的要求。
>
> **什么时候会撞上**：最常见的是**把 `to_thread` 改成进程池**的时候。
> `to_thread` 写法里带 kwargs 跑得好好的，一改成 `run_in_executor(pool, ...)` 就报 `TypeError`：
>
> ```python
> # 改造前（线程池，kwargs 没问题）
> await asyncio.to_thread(render, doc, dpi=300)
>
> # 改造后（进程池，直接搬会炸）
> await loop.run_in_executor(pool, partial(render, dpi=300), doc)   # ✅ 要包 partial
> ```
>
> 顺带接上第 9 题：**`partial` 是可 pickle 的**（`lambda` 不是），
> 所以这个写法对 `ProcessPoolExecutor` 同样成立。
> 如果图省事写成 `lambda: render(doc, dpi=300)`，线程池能跑，进程池会抛 `PicklingError`。

**8.** `concurrent.futures` 里实现「谁先完成先处理」的辅助函数是 ______________，
配合 `ex.______________(fn, item)` 提交任务使用。

**答：** as_completed submit

> ✅ **1/1**。两空全对，和选择题 #17、判断题 #10/#11 的顺序语义串上了。
> 惯用法记牢那个字典：`futures = {ex.submit(fn, item): item for item in items}`，
> 然后 `item = futures[fut]` 反查——这是 `map` 做不到的事。

**9.** 提交给 `ProcessPoolExecutor` 的函数必须是 ______________ 级函数，
因为函数和参数都要经过 ______________ 序列化后传给子进程。

**答：** 非lambda pickle

> ⚠️ **0.5/1**。第 2 空 `pickle` 对；**第 1 空应为「模块」——模块级函数**。
>
> **什么叫「模块级函数」**：定义在 `.py` 文件**最外层**（缩进为 0）、
> 不嵌套在任何函数里的函数——也就是 `import mymodule` 之后能直接
> `mymodule.task` 取到的那种。对照 JS：≈ 顶层的 `export function task()`，
> 而**不是**写在另一个函数体内的函数。
>
> ```python
> # mymodule.py
> def task(x):              # ✅ 模块级：文件最外层
>     return x * 2
>
> def outer():
>     def inner(x):         # ❌ 不是模块级：嵌套在 outer 里
>         return x * 2
>     return inner
> ```
>
> **为什么 pickle 在意这个**：pickle 序列化函数时存的**不是代码，
> 是「模块名 + 限定名」这条查找路径**（`__module__` + `__qualname__`），
> 子进程拿到后重新 `import` 该模块、再按名字取属性。
> 所以真正的判据是**「能不能按名字从模块顶层重新找回来」**。
>
> 一眼判断的办法：**看 `__qualname__` 里有没有 `<locals>` 或 `<lambda>`**。
> 实测（`pickle.dumps` 逐个试）：
>
> ```python
> def task(x): ...                       # __qualname__ = 'task'                 ✅
> class Job:
>     def run(self, x): ...              # __qualname__ = 'Job.run'              ✅
>     @staticmethod
>     def helper(x): ...                 # __qualname__ = 'Job.helper'           ✅
> def outer():
>     def inner(x): ...                  # __qualname__ = 'outer.<locals>.inner' ❌ AttributeError
> square = lambda x: x * x               # __qualname__ = '<lambda>'             ❌ PicklingError
> ```
>
> 注意**第 2、3 行**：模块级**类**里的方法，`__qualname__` 是 `Job.run` 这样的带点路径，
> pickle 顺着 `模块 → Job → run` 一样找得回来，**所以是可以的**。
> 绑定方法 `Job(2).run` 也行——只要**那个实例本身可 pickle**
> （实例里存了文件句柄、socket、DB 连接就不行）。
>
> 真正过不去的只有一类：**`__qualname__` 里带 `<locals>` 或 `<lambda>`** ——
> 它们是运行时才存在的对象，模块里根本没有这个名字。
>
> 所以「模块级函数」是最省事的说法（覆盖绝大多数场景），
> 严格表述是「**可 pickle 的可调用对象**」。
>
> 实测第 18 题的那个错误：
>
> ```console
> PicklingError: Can't pickle <function <lambda> at 0x...>:
>                attribute lookup <lambda> on __main__ failed
> ```
>
> 注意报错里的 **`attribute lookup ... failed`**——它说的正是「按名字找不回来」，
> 而不是「lambda 这个类型不支持」。
>
> **要给函数固定参数怎么办**：用 `functools.partial(task, mode="fast")`
> （`partial` 可 pickle，这正是它相对 lambda 的关键优势），
> 或者把可变部分放进参数里传。

**10.** `Future` 对象上：阻塞取结果（并重新抛出任务内异常）用 `______________`，
不阻塞地拿异常对象用 `______________`，注册完成回调用 `______________`。

**答：** 不知道

> ❌ **0/1**。答案是 `result()` / `exception()` / `add_done_callback()`。
>
> ⚠️ **但这题的题干我写错了**：`exception()` **同样是阻塞的**。实测——
>
> ```python
> def boom():
>     time.sleep(0.5); raise ValueError("kaboom")
>
> f = ex.submit(boom)
> e = f.exception()
> #=> exception() 返回耗时 0.50s -> ValueError('kaboom')     ← 等满了 0.5 秒
> f.exception(timeout=0)
> #=> TimeoutError                                           ← 确实会等
> ```
>
> 两者的真正区别**不是阻塞与否，是抛出与返回**：
>
> | 方法 | 是否阻塞 | 任务抛了异常时 |
> |---|---|---|
> | `f.result(timeout=None)` | **是** | **重新抛出**那个异常 |
> | `f.exception(timeout=None)` | **是** | **返回**异常对象（不抛） |
> | `f.done()` | **否** | 只回 `True`/`False` |
> | `f.cancelled()` | **否** | 是否被取消 |
> | `f.add_done_callback(cb)` | **否** | 完成后回调，`cb(future)` |
>
> 真正「不阻塞」的查询是 `done()` / `cancelled()`。题目重出时第 2 空应改为
> 「**取回异常对象而不抛出**用 `______`」。
>
> 完整心智模型（和 JS 对照）：
>
> ```python
> f = ex.submit(fn, x)
> f.result(timeout=5)        # ≈ await p          （重抛异常；超时抛 TimeoutError）
> f.exception()              # ≈ p.catch(e => e)  （拿到异常本身）
> f.add_done_callback(cb)    # ≈ p.then(cb)       ★ 唯一非阻塞的「拿结果」方式
> f.done() / f.cancelled()   # JS 没有对应物（Promise 状态不可同步查询）
> f.cancel()                 # 只能取消【还没开始】的任务，运行中的返回 False
> ```
>
> 最后一行是高频追问，也是聊 FastAPI 超时时的核心坑：
> **`cancel()` 对已经在跑的任务返回 `False`，杀不掉它**——所以 `wait_for` 超时后
> 子进程还在烧 CPU，见 [[web/fastapi-cpu-bound]] §4。
>
> > **面试落点**：`result()` 重抛异常、`exception()` 返回异常，**两者都会阻塞**；
> > 只有 `done()` / `cancelled()` / `add_done_callback()` 不阻塞。
> > `cancel()` 只对未启动的任务有效。

**11.** 量级速记：协程的内存开销是 ______ 级、切换是 ______ 级；
线程的内存开销是 ______ 级、切换是 ______ 级。

**答：** KB ns MB 微秒

> ✅ **1/1**。四空全对。
> 把**原因**接上就是完整答案（也和第 2 题的「调度者」串起来了）：
>
> | | 内存为什么是这个量级 | 切换为什么是这个量级 |
> |---|---|---|
> | 协程 | 只是一个**堆上的对象**，保存 frame 和局部变量 | **纯函数调用级**，不进内核、不换栈、不刷 TLB |
> | 线程 | 内核要分配 ~8 MB **栈**（虚拟地址空间，按需提交物理页） | **陷入内核**做上下文切换：保存寄存器、换栈、可能刷 cache |
>
> 这组数字是「为什么高并发选 asyncio」的**量化依据**：
> 10 万连接 × KB 级 = 几百 MB，可行；10 万线程 × 8 MB = 800 GB 虚拟地址空间，不可行。
> 面试报数字时带上这个推算，比单说「协程更轻」有力得多。

**12.** 现代 Python 后端最常见的混合模式是：______________ 主循环 +
`asyncio.to_thread` 兜底 ______________ 库 + ______________ 处理 CPU 密集任务。

**答：** asyncio.run 同步 多进程

> ⚠️ **0.5/1**。第 2、3 空对（「同步（阻塞）库」「进程池 / 多进程」都算对）；
> **第 1 空应为「asyncio」，不是「asyncio.run」**。
>
> 差别不只是抠字眼——`asyncio.run()` 是**进程入口函数**，不是「主循环」本身：
> 它做的是「建循环 → 跑完这一个协程 → 关循环」，是一次性的。
> 「asyncio 主循环」指的是那个**长期运行的事件循环**。
>
> 而且在最典型的场景里，**你根本不会写 `asyncio.run()`**：
>
> ```bash
> uvicorn app:app --workers 4      # ← 循环由 uvicorn 创建并持有
> ```
>
> FastAPI 项目的入口是 ASGI 服务器，`app` 只是个 ASGI callable；
> 要挂进程级的启动/关闭逻辑（比如建那个 `ProcessPoolExecutor`）用的是 `lifespan`。
> 真正会写 `asyncio.run(main())` 的是 **CLI 工具、爬虫脚本、后台 worker、机器人**，
> 而且**整个进程只写一次、放在最外层**。
>
> 完整的混合模式（见 [[web/fastapi-cpu-bound]]）：
>
> ```python
> # asyncio 事件循环（由 uvicorn 持有）
> async def handler():
>     data = await asyncio.to_thread(blocking_db_query, sql)   # ① 同步阻塞库 → 线程池
>     loop = asyncio.get_running_loop()
>     return await loop.run_in_executor(pool, cpu_heavy, data) # ② CPU 密集 → 进程池
> ```

### 填空题小结

**9.5 / 12**。三处扣分全是**用词精度**问题，没有一处是机制理解错误：

| # | 你写的 | 应为 | 性质 |
|---|---|---|---|
| 2 | 用户 | **事件循环** | 方向对，但答的是「在哪一态」而非「谁在调度」 |
| 9 | 非lambda | **模块** | 只排除了一种反例，没答出正面判据 |
| 12 | asyncio.run | **asyncio** | 把「入口函数」当成了「主循环」 |

这三处有个共同点：**你知道是怎么回事，但没用上那个准确的名词**。
填空题恰好是放大这个弱点的题型——面试口述时，这类模糊表述会招来追问，
而追问正是失分点。对策是刻意记「主体名词」：调度者叫**事件循环**，
可 pickle 的判据叫**模块级**，长期运行的那个东西叫**事件循环 / 主循环**。

**最大的亮点是第 3 题**：选择 #16、判断 #8 连错两次的 `min(32, cpu+4)`，
在填空题里公式和数值都写对了——原回炉清单的第一条可以划掉。

### 全卷总结

| 分区 | 得分 | 率 |
|---|---|---|
| 选择题 | 14.5 / 18 | 81% |
| 判断题 | 14 / 15 | 93% |
| 填空题 | 9.5 / 12 | 79% |
| **合计** | **38 / 45** | **84%** |

**扣分分布**（7 分）：真正的机制理解错误只有 **1 处**（选择 #15，asyncio 对 CPU 密集
是负收益而非零收益）——而它在判断题 #9 已经自行纠正了。其余是审题（选择 #6）、
纯记忆（选择 #16，已在填空 #3 纠正）、推论没走完（选择 #17 漏 C）、
用词精度（填空 2 / 9 / 12）、以及一处完全空白（填空 #10）。

**还需要回炉的只剩两条**：

1. **`Future` 的五个方法**（填空 #10 空白）——`result` / `exception` 都阻塞、
   区别在抛出与返回；`done` / `cancelled` / `add_done_callback` 不阻塞；
   `cancel()` 杀不掉运行中的任务。
2. **三个主体名词**（填空 2 / 9 / 12）——事件循环、模块级、事件循环（主循环）。

**两处出题缺陷已在题目下方标注**：判断 #5（复合命题两种判法都成立）、
填空 #10（`exception()` 并非不阻塞）。

---

## 批改记录（一）· 选择题　[2026-09-13]

> 体例说明：本卷选择题与判断题的批改写成了独立区块；**填空题起改为把批注直接写在每题答案下方**，
> 后续题组一律沿用新体例。

**得分：14.5 / 18**（全对 14 题，错 3 题，多选漏选 1 项算半分）。
本节所有输出在 **CPython 3.11.9 / Windows / 8 核** 实测。

| # | 你的答案 | 正确 | 判定 |
|---|---|---|---|
| 1–5 | B B C B A | B B C B A | ✅✅✅✅✅ |
| **6** | B | **D** | ❌ 看反了题干的「**错误**」 |
| 7–14 | B B B B B C B B | 同左 | ✅✅✅✅✅✅✅✅ |
| **15** | D | **C** | ❌ 零收益 vs 负收益 |
| **16** | B | **C** | ❌ `min(32, cpu+4)` 记成了 `cpu` |
| **17** | A B | **A B C** | ⚠️ 漏 C（0.5 分） |
| 18 | C | C | ✅ |

### 第 6 题 · 题干问的是「错误」的那一项

四个选项里 A、B、C 全是主题页对比表的原文，**只有 D 是错的**——协程数量的上限由
**内存**（每个协程 ~KB）和**事件循环单次遍历的开销**决定，跟 CPU 核数没有关系；
asyncio 本来就只用一个核，开 1 个协程还是 10 万个协程，用的都是那一个核。

真正受 CPU 核数约束的是 **B 说的进程数**——所以 B 是**正确陈述**，你把它当成错项选了。
这类「选出错误项」的题，作答前先在题干的「错误」两个字上画个圈。

顺带把三个上限的成因记牢，这是面试时区分「背过表」和「懂」的地方：

| 模型 | 上限 | 卡在哪 |
|---|---|---|
| 线程 | 数百~数千 | 每线程 ~8 MB 栈（虚拟地址空间）+ 内核调度开销 |
| 进程 | ≈ CPU 核数 | 每进程 ~10–50 MB 常驻内存；超过核数后只是徒增切换与 IPC |
| 协程 | 数万~数十万 | 每协程 ~KB 堆内存；**与核数无关** |

### 第 15 题 · asyncio 对 CPU 密集是「负收益」，不是「零收益」

你选的 D（「不能加速但也无害，等同于串行」）——**前半句对，后半句是这道题的陷阱**。

单机跑个脚本时，D 确实近似成立（主题页实测：串行 4.0s，asyncio 4.0s）。
但面试官问这句话，考的是**服务场景**：asyncio 是单线程**协作式**调度，
一个 CPU 密集协程**不 `await` 就永不让出**，事件循环在它跑完之前**无法调度任何其它协程**——
包括已经建立的几千个连接、正在等响应的所有请求。结果是整个服务在这段时间里**全部超时**，
而不是「这一个任务慢一点」。

```python
# ❌ 错误写法：一个协程把全服务拖死
@app.get("/hash")
async def hash_it(data: str):
    return bcrypt.hashpw(data.encode(), bcrypt.gensalt(14))   # 纯 CPU，~1s 不让出

# ✅ 正确写法：踢进线程池/进程池，事件循环立刻恢复调度
@app.get("/hash")
async def hash_it(data: str):
    return await asyncio.to_thread(bcrypt.hashpw, data.encode(), bcrypt.gensalt(14))
```

对照串行脚本：串行时「慢 4 秒」只影响这一个任务；asyncio 里「慢 4 秒」影响**所有并发请求**。
同样的 4 秒，代价完全不是一个量级——这就是 C 里「更糟」二字的分量，
也是主题页说它是「FastAPI 生产事故头号原因」的原因。

> **面试落点**：「asyncio 能加速 CPU 密集吗？」——**不能，而且比串行更糟**。
> 串行只慢自己，asyncio 卡的是整个事件循环，其它请求全部超时。

### 第 16 题 · 两个 Executor 的默认值不一样

```python
import os, concurrent.futures as cf
os.cpu_count()                         #=> 8
cf.ThreadPoolExecutor()._max_workers   #=> 12   ← min(32, cpu_count + 4)
cf.ProcessPoolExecutor()._max_workers  #=> 8    ← cpu_count()
```

你答的 8 是 **ProcessPoolExecutor** 的默认值。两者不同**不是随意定的**，理由正是这张卷子的主线：

- **线程池**服务的是 **I/O 密集**任务，worker 大部分时间在等 socket、**不占 CPU**，
  所以 worker 数可以远大于核数；`+4` 是给「就算单核也至少给点并发度」留的余量，
  `min(32, ...)` 是防止在 96 核机器上一口气建 100 个线程（每个 ~8 MB 栈）。
- **进程池**服务的是 **CPU 密集**任务，每个 worker 都要**真占满一个核**，
  超过核数纯属徒增内存与上下文切换——所以默认恰好是 `cpu_count()`，且「一般不要超过它」。

一句话记法：**线程池默认 `cpu+4`（还封顶 32），进程池默认 `cpu`**。

### 第 17 题 · 漏掉的 C —— `map` 的异常是**惰性**抛出的

`ex.map()` 返回的是**生成器**，不是 list。提交任务是立刻的，但**异常要等你迭代到那一项才抛**：

```python
def fn(x):
    if x == 2:
        raise ValueError(f"boom {x}")
    return x * 10

with ThreadPoolExecutor(max_workers=4) as ex:
    it = ex.map(fn, [1, 2, 3])
    print("map() 已返回，还没抛异常")   #=> map() 已返回，还没抛异常
    for r in it:
        print("got", r)               #=> got 10       ← 第一项正常产出
                                      #=> ValueError: boom 2   ← 迭代到第二项才炸
```

**工程后果**：`map` 一旦抛异常，**整个循环就中断了**，第 3 项及之后的结果你一个也拿不到，
而且无法知道是哪个输入出的错（异常里没带 item）。这正是主题页把
`submit` + `as_completed` 标成 ★推荐 的原因——它能**逐个任务**捕获、且能反查是哪个 item：

```python
futures = {ex.submit(fn, item): item for item in items}
for fut in as_completed(futures):
    item = futures[fut]               # ← 反查输入，map 做不到
    try:
        result = fut.result()
    except Exception:
        log.exception("处理 %s 失败", item)   # ← 单个失败不影响其余
```

顺序语义的实测（A、B 你都答对了，这里补一组对照数据加深印象）：

```python
def sleep_then(x):
    time.sleep(x / 10); return x

ex.map(sleep_then, [3, 1, 2])                    #=> [3, 1, 2]   ← 输入序
[f.result() for f in as_completed(futs)]         #=> [1, 2, 3]   ← 完成序
```

> **面试落点**：`map` 保输入序、异常惰性抛出且会中断整个迭代；
> `submit` + `as_completed` 按完成序、可逐任务捕获异常、可用 `{future: item}` 字典反查输入。
> 生产代码默认用后者。

### 小结

**14 题全对**覆盖了概念辨析（1–5）、数据共享与无锁原因（7、8）、
整条选型决策树（9–13）、GIL 导致线程池反而更慢（14）、进程池的 pickle 约束（18）——
**机制理解是通的**，这是最难补的部分。

3 处失分的性质各不相同，按优先级：

1. **第 15 题是真正要回炉的**——「无害」和「更糟」差的是对协作式调度的理解，不是记忆。
   重读主题页 §4 末尾的引用块。
2. **第 16 题纯记忆**——把 `min(32, cpu+4)` / `cpu` 这组默认值背下来即可，附带记住理由。
3. **第 6 题是审题**——不是知识问题，考场上留意「错误 / 不属于 / 除外」这类反向题干。
4. **第 17 题漏 C**，属于「知道结论、没推到后果」：知道 map 保序，但没想过异常什么时候抛、
   抛了之后剩下的任务怎么办。

判断题与填空题填好后发我，我接着批改并补到「批改记录（二）」。

---

## 批改记录（二）· 判断题　[2026-09-13]

**得分：14 / 15**。唯一的实错是第 8 题；第 5 题判你对，但那是**我出题的问题**，下面说明。
本节代码在 **CPython 3.11.9** 实测。

| # | 你的答案 | 正确 | 判定 |
|---|---|---|---|
| 1–4 | ❌ ✅ ❌ ✅ | 同左 | ✅✅✅✅ |
| **5** | ❌ | **题目有缺陷** | ✅ 判对（见下） |
| 6, 7 | ✅ ❌ | 同左 | ✅✅ |
| **8** | ❌ | **✅** | ❌ 唯一实错 |
| 9–15 | ❌ ❌ ❌ ❌ ✅ ✅ ✅ | 同左 | ✅✅✅✅✅✅✅ |

### 第 8 题 · 和选择题第 16 题是同一个知识点，错法一致

「`ProcessPoolExecutor` 的 `max_workers` 默认等于 `cpu_count()`，一般不建议超过它」——
这句**完全正确**，是主题页 §5 的原文。

值得注意的是**错法的一致性**：选择题 #16 问 8 核机器上 `ThreadPoolExecutor()` 的默认值，
你答了 `8`；这里问「进程池默认 = `cpu_count()`」，你答了「错」。
两次都指向同一个内部模型——**你把两个默认值记反了**。实测：

```python
import os, concurrent.futures as cf
os.cpu_count()                         #=> 8
cf.ThreadPoolExecutor()._max_workers   #=> 12   ← min(32, cpu_count + 4)
cf.ProcessPoolExecutor()._max_workers  #=> 8    ← cpu_count()
```

记法不要硬背数字，记**理由**（这条理由本身就是面试答案）：

| | 默认 | 为什么 |
|---|---|---|
| **线程池** | `min(32, cpu+4)` | 服务 **I/O 密集**任务，worker 大部分时间在等 socket、**不占 CPU**，所以可以超配核数；`+4` 给单核机器留并发度，`min(32, …)` 防止 96 核机器上建 100 个线程（每个 ~8 MB 栈） |
| **进程池** | `cpu_count()` | 服务 **CPU 密集**任务，每个 worker 要**真占满一个核**，超过核数纯属徒增内存与切换 |

一句话：**线程池能超配（`cpu+4`，封顶 32），进程池不超配（`cpu`）。**

### 第 5 题 · 我的题出坏了，判你对

原题干是个**复合命题**：

- 前半「两种 Executor 的接口完全一致」——**真**。
  `submit` / `map` / `shutdown` / `as_completed` / `Future` 的签名与语义确实一模一样，
  这正是 `concurrent.futures` 设计的核心价值。
- 后半「从线程切到进程**只需改一个类名**」——**假**。

复合命题中只要有一个合取项为假，整句就为假，所以你答 ❌ 是**技术上更正确**的。
我原本按主题页原文预期 ✅，是我照抄了页面里那句面向「接口统一」的修辞，没意识到
「只需改一个类名」会被当作严格断言——这题已在上面标注为出题缺陷，重出时拆成两题。

顺带把「改类名之后还要动什么」列全，这是第 18 题（lambda → `PicklingError`）的完整版：

| 换成进程池后必须处理 | 原因 |
|---|---|
| 函数必须是**模块级**可 pickle 对象 | `lambda` / 闭包 / 嵌套函数 / 实例方法都会炸 |
| 参数和返回值必须可 pickle，且**有序列化成本** | 传 200 MB DataFrame 可能比计算还慢——改传文件路径 / DB 主键 |
| **全局变量、单例、连接池不再共享** | 子进程是独立地址空间，DB 连接、缓存都要在 `initializer` 里各建一份 |
| Windows / macOS 需 `if __name__ == "__main__"` 守卫 | 默认 start method 是 `spawn`，子进程会重新 import 主模块 |
| 日志要重新配置 | 子进程不继承父进程的 logging handler（`spawn` 下） |
| 异常 traceback 可能残缺 | 跨进程传递时被重建 |

> **面试落点**：「`concurrent.futures` 两种 Executor 能无缝互换吗？」
> 答：**接口能，代码不能**。接口完全一致是它的设计目标；
> 但换成进程后要额外满足 pickle 约束、放弃共享内存状态、处理 spawn 语义。

### 第 6 题 · 判对了，但它只讲了一半（重要补充）

「协程不会在字节码级被抢占，所以不需要 `threading.Lock`」——正确。
**但这不等于 asyncio 里没有竞态**。协程在**每个 `await` 点**都可能让出，
跨 `await` 的 check-then-act 照样会出错：

```python
balance = 100

async def withdraw(amount, tag):
    global balance
    if balance >= amount:          # ① 检查
        await asyncio.sleep(0)     # ② 让出！别的协程插进来了
        balance -= amount          # ③ 扣款（此时条件可能已不成立）

await asyncio.gather(withdraw(100, "A"), withdraw(100, "B"))
#=> [A] 取走 100, 余额 0
#=> [B] 取走 100, 余额 -100
#=> 最终余额 -100   ← 单线程也能出负数
```

解法是 `asyncio.Lock`（**不是** `threading.Lock`——那会阻塞整个事件循环）：

```python
lock = asyncio.Lock()

async def safe(amount, tag):
    global balance
    async with lock:               # ★ 临界区跨越 await 也安全
        if balance >= amount:
            await asyncio.sleep(0)
            balance -= amount
#=> [A] 取走 100, 余额 0
#=> [B] 余额不足
#=> 最终余额 0
```

> **面试落点**：asyncio 的无锁只保证**单个协程内、两个 await 之间**的代码是原子的。
> **临界区一旦跨越 `await`，就必须用 `asyncio.Lock`**。
> 这是「单线程就不用加锁」这个常见误解的正确边界。

### 第 9 题 · 上一轮的错点已经改过来了

选择题 #15 你答了「不能加速但无害」，这里问「至少不会比串行更差」你判了 ❌——
说明「负收益 vs 零收益」这个点已经吃透了。这是本卷进步最明显的一处。

### 小结

**14/15，判断题基本满分**。整套 15 题覆盖了 GIL 与 I/O、内存模型、顺序语义、
事件循环阻塞、FastAPI 执行模型五个维度，只错在一个**纯记忆点**上。

回炉清单只剩一条：

1. **两个 Executor 的默认值**（判断 8 + 选择 16 连续两次错在同一处）——
   背下「线程池 `cpu+4` 封顶 32，进程池 `cpu`」以及背后的理由。

无需回炉但值得读的两处补充：第 5 题的「换进程池还要动什么」六行表、
第 6 题的 `asyncio.Lock` 边界——后者是判断题没考到、但面试很容易追问的一步。

填空题填好后发我，补「批改记录（三）」。

---

## 相关

- [[concurrency/concurrency-models]] —— 本卷的主题页
- [[internals/gil]] —— 为什么多线程不能并行
- [[concurrency/threading]] —— 线程与锁的细节
- [[concurrency/multiprocessing]] —— 进程与 IPC
- [[concurrency/asyncio-fundamentals]] —— 事件循环原理
- [[web/fastapi-core]] —— `def` / `async def` 路由的内部机制
- [[interview/roadmap]] —— 第 3 周 Day 1
