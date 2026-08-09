---
title: "IO 多路复用：asyncio 的底座"
date: 2026-08-07
tags: [epoll, select, poll, IO多路复用, C10K, 事件循环, 操作系统]
sources: ["interview-python-cn.md"]
---

# IO 多路复用：asyncio 的底座

[[concurrency/asyncio-fundamentals]] 里画的事件循环有一层叫 **Selector（epoll / kqueue / IOCP）**——
这一页解释它是什么。**「asyncio 底层是怎么实现的」是 asyncio 深挖时的必然追问**，
答到 epoll 才算讲完；中文面试题库里的"select、poll、epoll 的区别"也是十年常青题。

## 1. 先分清四组概念（面试爱绕）

**阻塞 vs 非阻塞**：发起 IO 后，**调用方要不要一直等**。
**同步 vs 异步**：**数据从内核拷贝到用户空间这一步，是谁做的**。

五种 IO 模型（POSIX 定义）：

| 模型 | 发起时 | 数据就绪后的拷贝 | Python 里的体现 |
|---|---|---|---|
| **阻塞 IO** | 挂起等待 | 内核拷完才返回 | `socket.recv()` 默认行为 |
| **非阻塞 IO** | 立即返回 `EAGAIN` | 需要轮询到就绪再拷 | `sock.setblocking(False)` + 忙等（浪费 CPU） |
| **IO 多路复用** ★ | **阻塞在 select/epoll 上**，一次监听 N 个 fd | 就绪后自己 `recv` 拷贝 | **asyncio / uvloop / nginx / Node** |
| 信号驱动 IO | 立即返回，就绪时发 SIGIO | 自己拷 | 极少用 |
| **异步 IO（AIO）** | 立即返回 | **内核拷完再通知你** | Linux `io_uring`、Windows IOCP |

> **面试落点**：**IO 多路复用严格说仍是"同步"的**——`epoll_wait` 返回后，
> 数据还是要你自己调 `recv()` 从内核缓冲区拷到用户空间。
> 只有 IOCP / io_uring 那种"内核帮你拷完再通知"才是真正的异步 IO。
> 能指出这一点，说明你不是只背了名词。
>
> 所以准确说法是：**asyncio 是"基于 IO 多路复用的单线程并发"，不是"异步 IO"**——
> 尽管名字里有 async。

## 2. 核心思想：一个线程盯住成千上万个连接

```text
传统方案：一连接一线程
  10000 连接 = 10000 线程 × 8MB 栈 = 80 GB 虚拟内存 + 海量上下文切换  ❌

IO 多路复用：
  1 个线程 → epoll_wait() → 内核返回"这 37 个 fd 现在可读"
                          → 依次处理这 37 个
  10000 连接 = 1 个线程 + 1 个 epoll 实例                            ✅
```

这就是 **C10K 问题**（单机一万并发连接）的标准解法，也是
nginx、Redis、Node.js、asyncio **全都采用**的架构。

## 3. select / poll / epoll 对比（标准考题）

| | select | poll | **epoll**（Linux） |
|---|---|---|---|
| fd 上限 | **1024**（`FD_SETSIZE` 编译期写死） | 无硬上限 | 无硬上限 |
| 数据结构 | 三个 fd_set 位图 | pollfd 数组 | **内核红黑树 + 就绪链表** |
| 每次调用的开销 | **把整个 fd 集合从用户态拷到内核态** | 同样要全量拷贝 | **只在 `epoll_ctl` 时注册一次**，`epoll_wait` 不拷贝集合 |
| 查找就绪 fd | **O(n) 轮询全部 fd** | O(n) | **O(1)**——内核把就绪的挂到就绪链表，直接返回 |
| 返回内容 | 要自己遍历判断哪个就绪 | 同 | **直接给你就绪的那些** |
| 触发模式 | 只有水平触发 | 只有水平触发 | **水平触发 LT + 边沿触发 ET** |
| 可移植性 | POSIX，到处能用 | POSIX | **仅 Linux** |

**一句话总结**：
**select/poll 的复杂度是 O(n)——每次都要把全部 fd 拷进内核并逐个检查；
epoll 是 O(1)——fd 只注册一次，内核用回调把就绪的 fd 主动挂到就绪队列。**
连接数越多、活跃比例越低，epoll 的优势越大。

各平台的等价物：

| 平台 | 机制 |
|---|---|
| Linux | **epoll**（`io_uring` 是更新的真异步方案） |
| macOS / BSD | **kqueue** |
| Windows | **IOCP**（真异步）/ select |
| 跨平台抽象 | Python 的 `selectors`、C 的 **libuv**（Node 和 uvloop 都用它） |

### 水平触发 LT vs 边沿触发 ET

```text
LT（Level Triggered，默认）：只要缓冲区【还有数据】就一直通知
    → 好写：可以一次只读一部分，下次还会通知
ET（Edge Triggered）：只在【状态发生变化】时通知一次
    → 高效：通知次数少
    → 难写：必须配合非阻塞 fd，一次性 read 到 EAGAIN 为止，否则剩下的数据再也不会被通知
```

> **面试落点**：**ET 必须搭配非阻塞 IO 并循环读到 `EAGAIN`**，
> 否则会丢数据（状态没再变化，内核不会再通知你）。nginx 用 ET，
> Python 的 `selectors` 和 asyncio 默认用 **LT**（更安全）。

## 4. Python 里直接用：`selectors` 模块

标准库的 `selectors` 是对 select/poll/epoll/kqueue 的**跨平台封装**，
会自动选当前平台最优的实现——**这正是 asyncio 事件循环内部用的东西**。

```python
import selectors, socket

sel = selectors.DefaultSelector()      # Linux 上是 EpollSelector，macOS 上是 KqueueSelector
print(type(sel).__name__)

def accept(sock):
    conn, addr = sock.accept()
    conn.setblocking(False)             # ★ 必须非阻塞
    sel.register(conn, selectors.EVENT_READ, read)

def read(conn):
    data = conn.recv(1024)
    if data:
        conn.sendall(data)              # echo
    else:
        sel.unregister(conn)
        conn.close()

srv = socket.socket()
srv.bind(("localhost", 8000)); srv.listen()
srv.setblocking(False)
sel.register(srv, selectors.EVENT_READ, accept)

while True:                             # ← 这就是一个手写的事件循环
    for key, mask in sel.select(timeout=1):    # 阻塞在这里，等任意 fd 就绪
        callback = key.data
        callback(key.fileobj)
```

**这 20 行就是 asyncio 的骨架**。asyncio 在它之上加了：协程的挂起/恢复、
Future/Task 抽象、定时器堆、异常传播、取消语义——但**底层等待 IO 的那一步，就是上面这个 `sel.select()`**。

## 5. 串起整条链路

```text
你的代码
  await reader.read()                     ← 协程挂起，让出控制权
      ↓
asyncio Task / Future                     ← 注册"这个 Future 完成时恢复我"
      ↓
事件循环 BaseEventLoop._run_once()
      ├── 处理 ready 队列的回调
      ├── 检查定时器堆（heapq）
      └── selector.select(timeout)        ← ★ 阻塞在这里
              ↓
          selectors.EpollSelector
              ↓
          epoll_wait()  系统调用           ← 内核返回就绪的 fd 列表
              ↓
          内核：网卡收到数据 → 协议栈 → socket 接收缓冲区 → 把 fd 挂进就绪链表
```

**推论（能解释很多现象）**：

- **事件循环只有在 `selector.select()` 上才真正"睡着"**——CPU 占用为 0。
  这就是 asyncio 空闲时不耗 CPU 的原因。
- **一个不 await 的 CPU 密集协程会让循环永远走不到 `select()`**，
  于是所有 IO 事件都得不到处理 → 整个 worker 卡死。这就是
  [[concurrency/asyncio-fundamentals]] 里"阻塞事件循环"的物理解释。
- **`uvloop` 快，是因为它用 Cython 封装了 libuv**（Node 用的同一个库），
  把这条链路上的 Python 层开销换成了 C。

## 6. 与其它并发模型的位置关系

```text
                     一个连接一个线程        IO 多路复用
                     ──────────────────    ──────────────────
内存               每线程 ~8MB 栈           每连接几 KB
上下文切换         内核态，微秒级            函数调用级，纳秒级
可扩展连接数        数百 ~ 数千              数万 ~ 数十万
CPU 并行           受 GIL 限制（Python）     单线程，不并行
编程模型           顺序，直观               回调 / 协程
Python 里          threading / 线程池        asyncio / uvloop
```

见 [[concurrency/concurrency-models]] 的完整选型决策树。

## 7. 面试标准答法

> **问：select、poll、epoll 的区别？**
>
> 三者都是 IO 多路复用——用一个线程同时监听多个 fd。区别在效率：
>
> - **select**：fd 数量受 `FD_SETSIZE`（通常 1024）限制；每次调用都要**把整个 fd 集合从
>   用户态拷贝到内核态**，返回后还要**O(n) 遍历**才知道哪些就绪。
> - **poll**：用数组代替位图，**去掉了数量上限**，但拷贝和 O(n) 遍历的问题依旧。
> - **epoll**：fd 通过 `epoll_ctl` **只注册一次**（内核用红黑树管理），`epoll_wait`
>   **不需要传入整个集合**；内核在 fd 就绪时通过回调把它挂到**就绪链表**，
>   所以获取就绪 fd 是 **O(1)**，且直接返回就绪的那些。还多了**边沿触发**模式。
>
> **连接数大、活跃比例低时 epoll 优势最明显**——这正是 C10K 场景。
> epoll 只在 Linux 上有，macOS/BSD 对应 kqueue，Windows 是 IOCP。
>
> **补一句**：Python 的 `selectors` 模块封装了这些，asyncio 的事件循环就建立在它之上；
> uvloop 更快是因为换成了 libuv 的 C 实现。

> **问：asyncio 是异步 IO 吗？**
>
> 严格说**不是**。asyncio 用的是 **IO 多路复用 + 协程**，属于同步非阻塞——
> `epoll_wait` 返回后仍需自己 `recv()` 把数据从内核拷到用户空间。
> 真正的异步 IO 是内核拷贝完成后再通知你，对应 Windows 的 IOCP 和 Linux 的 io_uring。
> asyncio 的价值在于**用协程把回调式的多路复用写成了顺序代码的样子**。

## 相关

- [[concurrency/asyncio-fundamentals]] —— 事件循环建立在本页的 selector 之上
- [[concurrency/concurrency-models]] —— 与线程/进程模型的对比选型
- [[concurrency/threading]] —— "一连接一线程"方案的代价
- [[web/wsgi-asgi]] —— WSGI 无法支持长连接的根本原因
- [[interview/question-bank-internals-concurrency]] —— 并发题库
- [[sources/interview-python-cn]] —— 来源：中文题库操作系统/网络篇
