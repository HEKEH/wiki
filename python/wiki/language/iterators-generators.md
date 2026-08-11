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

### 「用两次」的典型翻车现场

这类 bug 的可怕之处在于**大多不报错**——只是静默地少做了事。

```python
# ① 先校验、再使用 —— 最常见，也最静默
def send_batch(records):
    if not any(r.valid for r in records):     # 第一次遍历：把 records 吃光了
        raise ValueError("no valid record")
    for r in records:                          # 第二次：空的
        send(r)                                # ❗ 一条都没发，还不抛异常

send_batch(filter(is_fresh, load_all()))       # 传进来的是 filter 对象
```

```python
# ② 求平均值：sum 之后再 len
nums = map(int, "1,2,3".split(","))
avg = sum(nums) / len(list(nums))   # ❌ ZeroDivisionError —— sum 已耗尽，list(nums) == []
```

```python
# ③ 记日志顺手数了一下
def process(items):
    log.info("待处理 %d 条", sum(1 for _ in items))   # 数完就没了
    for it in items:                                   # 循环体一次都不进
        handle(it)
```

```python
# ④ zip 校验长度后再建 dict
pairs = zip(keys, values)
assert len(list(pairs)) == len(keys)   # 通过了
d = dict(pairs)                        #=> {} ❗ 空字典
```

```python
# ⑤ 失败重试：重试的是被啃过的残骸
chunks = (buf for buf in split(big_file))
for attempt in range(3):
    try:
        upload(chunks)        # 第 1 次传到一半失败 → 第 2 次从断点继续 → 第 3 次直接传空
        break
    except NetworkError:
        continue
```

```python
# ⑥ in / any / next 只是"部分消耗"，比全空更难查
g = (x for x in range(5))
3 in g       #=> True     顺带吃掉了 0,1,2,3
list(g)      #=> [4]      只剩尾巴

# ⑦ 双重循环共用同一个迭代器
it = iter([1, 2, 3])
[(a, b) for a in it for b in it]   #=> [(1, 2), (1, 3)]   ❗ 期望 9 对
```

```python
# ⑧ itertools.groupby：外层一前进，之前的分组立刻作废（所有分组共享同一个源游标）
from itertools import groupby

gs = list(groupby([1, 1, 2, 2, 3]))                    # ❌ 先把分组物化
[(k, list(g)) for k, g in gs]                          #=> [(1, []), (2, []), (3, [])]

[(k, list(g)) for k, g in groupby([1, 1, 2, 2, 3])]    # ✅ 边遍历边消费
                                                       #=> [(1,[1,1]), (2,[2,2]), (3,[3])]
```

`groupby` 不预先分组，它只有**一个**源游标，边走边切。外层每次 `__next__` 做两件事：
换一个新的"当前组令牌"，然后把游标快进过当前组的剩余元素。旧的分组子迭代器
（grouper）循环条件里带着 `self.id is id` 的令牌校验，令牌一换就立刻结束——所以它不是
"数据被抢走后返回残缺结果"，而是**干脆利落地返回空**，一声不吭。

`list(groupby(...))` 连最后一组都是空的，是因为 `list()` 必须多调一次 `__next__` 才能拿到
`StopIteration`；那一次调用的第一步就是换令牌，第三组因此也被判死。这也解释了只前进一步时
的差异：`k1,g1 = next(it); k2,g2 = next(it)` → `g1` 空，`g2` 还活着。

同类"一次性"对象还有：文件对象、`csv.reader`、`os.scandir()`、DB cursor、
`requests.iter_lines()`、`re.finditer()`、异步生成器。

**三种修法**：

```python
# a) 物化 —— 在 API 边界把不确定的输入钉死（数据量可控时的首选）
def send_batch(records):
    records = list(records)     # 一行防住所有下游二次遍历

# b) 传「可再生的工厂」而不是迭代器 —— 数据量大、不能全进内存时
def send_batch(make_records):   # 传 callable
    if not any(r.valid for r in make_records()):
        raise ValueError(...)
    for r in make_records():    # 重新生成一个新迭代器
        send(r)

# c) itertools.tee —— 只需要两路并行消费时
a, b = itertools.tee(gen, 2)
# ⚠️ tee 内部有缓冲：两路进度差多少就缓存多少元素，
#    "一路跑完再跑另一路"等于把整个序列存进内存，还不如直接 list()
```

> **面试落点**：判断一个对象能不能重复遍历，看 `iter(x) is x`——为 `True` 说明它是迭代器，
> 一次性；为 `False`（如 list、dict、`dict.items()` 视图、`range`）才能反复遍历。
> 写公开 API 时，参数类型标注 `Iterable[T]` 就意味着**只能承诺遍历一次**，
> 需要多次遍历应该标 `Sequence[T]` 或在函数入口 `list()` 一下。

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

注意这是**一次性**的写法——遍历状态 `self.n` 被就地改掉，且 `__iter__` 返回自身：

```python
c = Countdown(3)
list(c), list(c)                 #=> ([3, 2, 1], [])
[(a, b) for a in c for b in c]   #=> [(3, 2), (3, 1)]   ❗ 期望 9 对（同坑 ⑦）
```

### 耗尽是不可逆的

协议规定：`__next__()` 一旦抛出 `StopIteration`，后续调用**必须**继续抛——
文档明确写着"不遵守这条的实现被视为 broken"。所以迭代器协议里没有 `reset()` / `rewind()`，
想从头再来只能**回到源头重新要一个迭代器**。

```python
l = [1, 2, 3]
it = iter(l)
list(it)         #=> [1, 2, 3]
l.append(4)
next(it)         # ❌ StopIteration —— 源变长了也救不回来
                 #    CPython 的 list_iterator 耗尽时把内部序列指针置为 NULL

# 但"未耗尽时"改源，迭代器是看得见的 —— 边遍历边改容器的坑
l2 = [1, 2, 3]
it2 = iter(l2); next(it2)
l2.append(4)
list(it2)        #=> [2, 3, 4]

# 生成器耗尽后执行帧直接销毁，连状态都不剩
g = (x for x in range(3)); list(g)
g.gi_frame                       #=> None
inspect.getgeneratorstate(g)     #=> 'GEN_CLOSED'
```

> ⚠️ 唯一常见的"能倒带"的迭代器是**文件对象**（`f.seek(0)`）。但 `seek` 不属于迭代器协议，
> 是文件对象靠可寻址的 OS 句柄额外提供的能力。换成 socket、`sys.stdin`、管道、
> `requests.iter_lines()`，同样是 file-like，`seek` 直接抛 `io.UnsupportedOperation`。
> 别把它当成"迭代器可以重置"的证据。

### 可重复遍历的写法

想让类能被反复 `for`，**`__iter__` 就不能 `return self`**，而要每次返回一个新的迭代器。
最简洁的做法是把 `__iter__` 本身写成生成器函数：

```python
class Countdown:                  # ✅ 可重复遍历
    def __init__(self, n): self.n = n
    def __iter__(self):
        cur = self.n              # 遍历状态是局部变量，不污染实例
        while cur > 0:
            yield cur
            cur -= 1

c = Countdown(3)
list(c), list(c)                 #=> ([3, 2, 1], [3, 2, 1])
iter(c) is c                     #=> False   每次调用产出一个新生成器
len([(a, b) for a in c for b in c])   #=> 9   嵌套遍历也正常
2 in c; list(c)                  #=> [3, 2, 1]   `in` 之后依然完整
```

> **设计判据**：遍历状态放在**实例属性**上 → 类是一次性的（自己既当 Iterable 又当 Iterator）；
> 放在 `__iter__` 的**局部变量**里 → 类可重复遍历。标准库走的是后者：`list` / `dict` / `set`
> 自己只是 Iterable，每次 `iter()` 现造一个独立的 `list_iterator` / `dict_keyiterator`。
> `collections.abc` 把 `Iterable` 和 `Iterator` 拆成两个 ABC，区分的正是这两种角色。
> 只有当"游标"本身就是要暴露给用户的东西时（如 DB cursor、`csv.reader`），才该让两者合一。

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
    r = yield from sub() # r 是 "done"
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
