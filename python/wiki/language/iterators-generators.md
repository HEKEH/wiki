---
title: "迭代器与生成器"
date: 2026-08-07
tags: [迭代器, 生成器, yield, yield from, 惰性求值, 协程]
sources: ["cpython-doc/functional-howto.rst", "interview-python-cn.md", "python-cheatsheet.md"]
---

# 迭代器与生成器

生成器是 Python 最有特色的机制之一，也是**通往 asyncio 的必经之路**——`async def` 协程在
CPython 里就是生成器的近亲（共用同一套帧挂起/恢复机制）。

## 1. 三个概念的区别（必考）

| 概念 | 定义 | 检验方法 |
|---|---|---|
| **Iterable（可迭代对象）** | 实现了 `__iter__`（或旧式 `__getitem__`） | `iter(x)` 不报错 |
| **Iterator（迭代器）** | 实现了 `__iter__` **和** `__next__` | `next(x)` 不报错 |
| **Generator（生成器）** | 用 `yield` 或生成器表达式创建的**特殊迭代器** | `inspect.isgenerator(x)` |

```python
lst = [1, 2, 3]
iter(lst)        #=> <list_iterator>    list 是 iterable，不是 iterator
next(lst)        # ❌ TypeError: 'list' object is not an iterator

it = iter(lst)
next(it)         #=> 1
iter(it) is it   #=> True    迭代器的 __iter__ 返回自身（这样才能用于 for）
```

**关键区别**：iterable 可以被反复遍历（每次 `iter()` 得到新迭代器），iterator **一次性、有状态、耗尽即废**。

```python
it = iter([1, 2, 3])
list(it)         #=> [1, 2, 3]
list(it)         #=> []      ← 已耗尽
```

> 这是生产事故高发点：把 `zip()` / `map()` / `filter()` / 生成器的结果**用两次**，第二次是空的。
> Python 3 里这些内置函数全部返回惰性迭代器（Python 2 返回 list）。

### 手写迭代器

```python
class Countdown:
    def __init__(self, n): self.n = n
    def __iter__(self):  return self          # 返回自身
    def __next__(self):
        if self.n <= 0:
            raise StopIteration               # 结束信号
        self.n -= 1
        return self.n + 1

list(Countdown(3))   #=> [3, 2, 1]
```

`for` 循环的等价展开（面试常要求手写）：

```python
it = iter(obj)
while True:
    try:
        x = next(it)
    except StopIteration:
        break
    body(x)
```

## 2. 生成器函数

函数体里出现 `yield`，**调用它不执行任何代码**，只返回一个生成器对象。

```python
def gen():
    print("start")
    yield 1
    print("middle")
    yield 2
    print("end")

g = gen()        # 什么都不打印！
next(g)          # 打印 start，返回 1
next(g)          # 打印 middle，返回 2
next(g)          # 打印 end，抛 StopIteration
```

**执行模型**：每次 `yield` 挂起时，**整个函数帧（局部变量、指令指针）被保留**，
下次 `next()` 从挂起点继续。这正是协程的基础。

```python
g = gen()
g.gi_frame.f_lasti      # 挂起时的字节码位置
g.gi_frame.f_locals     # 挂起时的局部变量
inspect.getgeneratorstate(g)   #=> 'GEN_CREATED' / 'GEN_SUSPENDED' / 'GEN_RUNNING' / 'GEN_CLOSED'
```

### 为什么用生成器：惰性 + 常数内存

```python
# ❌ 一次性加载 10GB 日志到内存
lines = open("huge.log").readlines()
errors = [l for l in lines if "ERROR" in l]

# ✅ 流式处理，内存占用与文件大小无关
def read_errors(path):
    with open(path) as f:
        for line in f:                 # 文件对象本身就是迭代器
            if "ERROR" in line:
                yield line.rstrip()

for e in read_errors("huge.log"):
    handle(e)
```

内存对比：

```python
import sys
sys.getsizeof([x*x for x in range(10**6)])     #=> ~8.1 MB
sys.getsizeof((x*x for x in range(10**6)))     #=> 192 bytes（无论多大都是常数）
```

### 生成器表达式

```python
sum(x*x for x in range(10**6))          # 单参数时括号可省
any(is_valid(x) for x in items)         # 短路，找到即停
next((x for x in items if x.ok), None)  # "取第一个满足条件的，没有就 None"
```

> **面试落点**：列表推导式 `[...]` 立即构造完整列表；生成器表达式 `(...)` 惰性产出、常数内存、
> 只能遍历一次。聚合运算（`sum`/`any`/`max`/`join`）优先用生成器表达式。
> 例外：`sorted()`、`len()`、需要多次遍历时必须物化成 list。

## 3. 生成器的双向通信：`send` / `throw` / `close`

`yield` 是**表达式**，有返回值——这让生成器变成了协程。

```python
def accumulator():
    total = 0
    while True:
        x = yield total       # yield 出 total，并等待 send 进来的值
        if x is None:
            break
        total += x
    return total              # 3.3+ 生成器可以有返回值

acc = accumulator()
next(acc)          #=> 0      必须先"预激"（priming），运行到第一个 yield
acc.send(10)       #=> 10
acc.send(5)        #=> 15
acc.close()
```

| 方法 | 作用 |
|---|---|
| `next(g)` / `g.send(None)` | 恢复执行，`yield` 表达式求值为 `None` |
| `g.send(v)` | 恢复执行，`yield` 表达式求值为 `v`（首次必须先预激） |
| `g.throw(ExcType)` | 在挂起点抛出异常（可被生成器内部 try 捕获） |
| `g.close()` | 在挂起点抛 `GeneratorExit`，生成器应该清理并退出 |

`close()` 与 `finally` 配合是资源清理的关键：

```python
def managed():
    conn = connect()
    try:
        while True:
            yield conn.read()
    finally:
        conn.close()        # close() / 被 GC 时都会执行
```

> ⚠️ 不要在 `GeneratorExit` 之后再 `yield`，会 `RuntimeError: generator ignored GeneratorExit`。

## 4. `yield from`：委托与管道

```python
def chain(*iterables):
    for it in iterables:
        yield from it          # 等价于 for x in it: yield x，但还转发 send/throw/close
                               # 并把子生成器的 return 值作为 yield from 表达式的值

list(chain([1,2], (3,4), "ab"))   #=> [1, 2, 3, 4, 'a', 'b']
```

`yield from` 的完整语义（不只是循环）：

- 透传 `send()` 给子生成器
- 透传 `throw()` / `close()`
- **子生成器 `return v` 时，`result = yield from sub()` 中 `result == v`**

```python
def sub():
    yield 1
    return "done"

def outer():
    r = yield from sub()
    print("sub returned", r)     #=> sub returned done
    yield 2

list(outer())    #=> [1, 2]
```

> **面试落点**：`yield from` 是 PEP 380 引入的，正是它让"基于生成器的协程"成为可能
> （`asyncio` 早期用 `@coroutine` + `yield from`，3.5 后才有 `async`/`await` 语法）。
> `await` 在字节码层面与 `yield from` 高度相似。见 [[concurrency/asyncio-fundamentals]]。

递归展开嵌套结构：

```python
def flatten(items):
    for x in items:
        if isinstance(x, (list, tuple)):
            yield from flatten(x)
        else:
            yield x

list(flatten([1, [2, [3, [4]], 5]]))   #=> [1, 2, 3, 4, 5]
```

## 5. 生成器管道（Unix pipe 风格）

```python
def read(path):
    with open(path) as f:
        yield from f

def grep(pattern, lines):
    for l in lines:
        if pattern in l:
            yield l

def parse(lines):
    for l in lines:
        yield json.loads(l)

def take(n, it):
    return itertools.islice(it, n)

# 组合：全程惰性，内存恒定，只读到需要的行数就停
for rec in take(10, parse(grep("ERROR", read("app.log")))):
    print(rec["msg"])
```

这是数据处理的经典 Python 惯用法，比一层层 list 转换快且省内存。

## 6. 异步迭代器与异步生成器

```python
class AsyncRange:
    def __init__(self, n): self.n, self.i = n, 0
    def __aiter__(self): return self
    async def __anext__(self):
        if self.i >= self.n:
            raise StopAsyncIteration
        self.i += 1
        await asyncio.sleep(0)
        return self.i - 1

async def agen(n):                 # 异步生成器（3.6+）
    for i in range(n):
        await asyncio.sleep(0.01)
        yield i

async def main():
    async for x in agen(3):
        print(x)
    result = [x async for x in agen(3)]    # 异步推导式
```

数据库驱动的游标、HTTP 流式响应、Kafka 消费都是异步生成器的典型场景。
异步生成器需要 `aclose()` 清理，`asyncio` 有 `shutdown_asyncgens()` 兜底。

## 7. 常见陷阱

```python
# ① 生成器只能遍历一次
g = (x for x in range(3))
sum(g), sum(g)        #=> (3, 0)

# ② 生成器里的异常延迟到迭代时才抛
def g():
    raise ValueError
gen = g()             # 不报错
next(gen)             # 这时才抛

# ③ 生成器表达式同样有延迟绑定问题 —— 而且比 lambda 更隐蔽
gens = [(x for _ in range(2)) for x in range(3)]
[list(g) for g in gens]   #=> [[2, 2], [2, 2], [2, 2]]   ❗ 不是 [[0,0],[1,1],[2,2]]
fns  = [lambda: x for x in range(3)]
[f() for f in fns]        #=> [2, 2, 2]                  同样的坑

# 原因：生成器表达式只有【最外层可迭代对象】(range(2)) 在创建时求值，
#       自由变量 x 要到【迭代时】才从外层作用域取——那时外层推导式早已跑完，x == 2。
# 修法同 lambda：用默认参数把值冻结进去
gens = [(x for _ in range(2)) for x in range(3)]          # ❌
gens = [((lambda v: (v for _ in range(2)))(x)) for x in range(3)]   # ✅ 额外一层作用域

# ④ 在生成器里 return 值不会出现在迭代结果中
def g2():
    yield 1
    return 2
list(g2())            #=> [1]   ← 2 被放进 StopIteration.value 里了

# ⑤ StopIteration 在生成器内被吞掉会变 RuntimeError（PEP 479，3.7+）
def g3():
    yield next(iter([]))   # ❌ RuntimeError: generator raised StopIteration
```

## 相关

- [[language/comprehensions-functional]] —— 推导式与生成器表达式
- [[language/context-managers]] —— `@contextmanager` 用生成器实现
- [[language/data-model]] —— 迭代协议在数据模型中的位置
- [[stdlib/collections-itertools]] —— `itertools` 全套惰性工具
- [[concurrency/asyncio-fundamentals]] —— 协程与生成器的血缘关系
- [[internals/bytecode-execution]] —— 生成器帧的挂起/恢复
- [[interview/question-bank-language]] —— 迭代器/生成器面试题
