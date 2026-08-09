---
title: "性能剖析与优化"
date: 2026-08-07
tags: [性能, 剖析, cProfile, py-spy, 优化, numpy]
sources: []
---

# 性能剖析与优化

**原则：先测量，再优化。** Donald Knuth："过早优化是万恶之源"，
但完整的原话是"我们应该忘掉小的效率问题，约 97% 的时间如此；
**但不要放过那关键的 3%**"。

## 1. 测量工具

### 微基准：`timeit`

```python
import timeit
timeit.timeit("'-'.join(str(n) for n in range(100))", number=10000)
timeit.timeit(lambda: f(x), number=1000)

# 命令行
# python -m timeit -s "setup" "statement"
```

```python
# 更好用的：在 IPython/Jupyter 里
%timeit f(x)
%%timeit
...
```

### 函数级：`cProfile`

```bash
python -m cProfile -s cumtime app.py | head -40
python -m cProfile -o out.prof app.py
```

```python
import cProfile, pstats
with cProfile.Profile() as pr:
    run_workload()
pstats.Stats(pr).sort_stats("cumulative").print_stats(20)
```

```text
   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
     1000    0.500    0.001    4.200    0.004 app/service.py:42(process)
                ↑ 自身耗时（不含子调用）      ↑ 累计耗时（含子调用）★ 先看这列
```

可视化：`snakeviz out.prof`、`gprof2dot`、`tuna`。

### 行级：`line_profiler`

```python
# pip install line_profiler
@profile                        # 用 kernprof -l -v script.py 运行
def slow_function():
    ...
```

### 生产采样：`py-spy` ★ 最有用

```bash
py-spy top --pid 1234              # 实时 top，无侵入、无需改代码
py-spy record -o flame.svg --pid 1234 --duration 30   # 火焰图
py-spy dump --pid 1234             # 打印所有线程/协程的当前调用栈（查卡死神器）
```

**`py-spy` 的价值**：**不需要重启进程、不需要装依赖到目标环境、开销 <1%**，
可以直接对着生产容器跑。这是排查线上性能问题的首选。

### 内存

```python
import tracemalloc
tracemalloc.start()
snap = tracemalloc.take_snapshot()
for s in snap.statistics("lineno")[:10]: print(s)
```

```bash
memray run -o out.bin app.py && memray flamegraph out.bin
```

见 [[internals/memory-model]]。

## 2. 优化的优先级

```text
1. 算法与数据结构      —— 收益 10~1000 倍  ★★★★★
2. 消除 I/O 等待与 N+1 —— 收益 10~100 倍   ★★★★★
3. 缓存                —— 收益 10~100 倍   ★★★★
4. 并发/并行           —— 收益 ~核数倍     ★★★
5. 换库（orjson/polars/numpy）—— 2~50 倍   ★★★
6. Python 层微优化     —— 1.1~2 倍         ★
7. C/Rust 扩展         —— 10~100 倍（局部）★★（成本高）
```

**永远先做 1 和 2。** 把 `for x in list` 的 `in` 从 O(n) 改成 set 的 O(1)，
收益远超任何微优化。

## 3. 算法与数据结构

```python
# ❌ O(n²)
if x in big_list: ...
# ✅ O(1)
if x in big_set: ...

# ❌ O(n) 的头部删除
lst.pop(0)
# ✅ O(1)
deque.popleft()

# ❌ O(n²) 字符串拼接
s = ""; for x in xs: s += x
# ✅ O(n)
"".join(xs)

# ❌ 重复排序
for q in queries: sorted(data)
# ✅ 排一次
data.sort()
# 加上 bisect 做 O(log n) 查找
```

见 [[stdlib/builtin-data-structures]] 的复杂度表。

## 4. I/O 与并发

```python
# ① 批量化：把 N 次往返变成 1 次
# ❌
for id in ids: db.get(id)                     # N 次查询
# ✅
db.execute(select(User).where(User.id.in_(ids)))   # 1 次

# ② 并发化
results = await asyncio.gather(*(fetch(u) for u in urls))

# ③ 消除 N+1（ORM）
select(User).options(selectinload(User.posts))

# ④ 流式处理，避免一次性物化
for chunk in pd.read_csv(f, chunksize=100_000): ...
```

## 5. 缓存

```python
from functools import lru_cache, cache

@cache                                  # 进程内，无过期
def expensive(n): ...

# 带 TTL 的（第三方）
from cachetools import TTLCache, cached
@cached(TTLCache(maxsize=1000, ttl=300))
def get_config(): ...

# 分布式缓存
await redis.setex(f"user:{uid}", 300, json.dumps(data))
```

**缓存三大问题（面试常问）**：

| 问题 | 现象 | 解法 |
|---|---|---|
| **缓存穿透** | 查不存在的 key，每次都打到 DB | 缓存空值 / 布隆过滤器 |
| **缓存击穿** | 热点 key 过期瞬间大量请求涌向 DB | 互斥锁重建 / 逻辑过期 |
| **缓存雪崩** | 大量 key 同时过期 | 过期时间加随机抖动 |

## 6. Python 层微优化（知道即可，别滥用）

```python
# ① 局部变量比全局快（LOAD_FAST vs LOAD_GLOBAL）
def f(items):
    _len = len                       # 热循环里绑成局部
    return [_len(x) for x in items]

# ② 推导式比显式 append 循环快（少了方法查找和调用）
[f(x) for x in xs]        # 比 for + append 快约 20~30%

# ③ 内置函数走 C 层
sum(xs)  max(xs)  any(...)  map(str.strip, lines)

# ④ 避免不必要的属性查找
obj.method                # 每次都是一次属性查找
m = obj.method            # 循环外提取

# ⑤ __slots__ 省内存 + 加速属性访问
# ⑥ 用生成器避免物化中间列表
# ⑦ 字符串用 f-string（最快的格式化）
# ⑧ 尽量少用异常做控制流（异常路径很慢）
# ⑨ 3.11+ 本身就比 3.8 快 25~60%，升级版本是最省力的优化 ★
```

> **面试落点**：被问优化时，别一上来讲微优化。
> 正确顺序是：**"先用 py-spy/cProfile 找热点 → 看是不是算法或 I/O 问题 →
> 再考虑并发和缓存 → 最后才是语言层优化"**。
> 顺带提一句"升级到 3.12/3.13 通常能白拿 20~40%"很实用。

## 7. 换更快的库

| 场景 | 慢 | 快 |
|---|---|---|
| JSON | `json` | **`orjson`**（2~10x）/ `msgspec` |
| 数据帧 | `pandas` | **`polars`**（Rust，多线程，5~50x）/ `duckdb` |
| 数值 | 纯 Python 循环 | **`numpy`**（向量化，10~100x） |
| HTTP | `requests` | `httpx` / `aiohttp`（异步并发） |
| 事件循环 | asyncio 默认 | **`uvloop`**（2~4x） |
| 校验 | Pydantic v1 | **Pydantic v2**（5~50x） |
| 正则 | `re` | `regex` / 预编译 / 换成字符串方法 |
| 日期解析 | `strptime` | `ciso8601` |
| 序列化 | `pickle` | `msgspec` / `protobuf` |

**numpy 向量化的威力**：

```python
# ❌ 纯 Python：~1.5 秒
result = [x * 2 + 1 for x in range(10_000_000)]

# ✅ numpy：~50 毫秒（30x）
import numpy as np
result = np.arange(10_000_000) * 2 + 1
```

原理：numpy 数组是**连续的 C 数组**（不是指针数组），运算在 C 层循环完成，
无 Python 对象开销、无解释器循环、可用 SIMD。见 [[internals/cpython-object-model]]。

## 8. 编译加速

| 方案 | 说明 | 代价 |
|---|---|---|
| **Cython** | 加类型注解编译成 C 扩展 | 需要改代码和构建链 |
| **mypyc** | 用 mypy 的类型注解直接编译（mypy 自己就是这么加速的） | 类型必须完整 |
| **Numba** | `@njit` JIT 编译数值代码 | 只适合数值循环 |
| **PyO3 / maturin** | 用 Rust 写扩展 | 学习成本，但性能与安全最佳 |
| **PyPy** | 换解释器，JIT | C 扩展兼容性差 |
| **CPython 3.13 JIT** | 实验性，自动 | 尚未成熟 |

```python
from numba import njit

@njit
def mandelbrot(c, maxiter):     # 纯数值循环，能快 50~200 倍
    ...
```

## 9. 一个完整的排查案例（可以在面试里讲）

```text
现象：API P99 从 100ms 涨到 3s

1. 看监控：只有 /orders 端点变慢，QPS 没变，错误率正常 → 不是流量问题
2. 看链路追踪：DB span 占了 2.8s → 问题在数据库
3. 看慢查询日志：一条 SELECT 执行了 2.7s
4. EXPLAIN ANALYZE：全表扫描，因为新加的 status 过滤没有索引
5. 加复合索引 (status, created_at) → 降到 15ms

如果第 2 步显示 DB 很快、时间在应用层：
6. py-spy record 生成火焰图 → 发现 json.dumps 占了 60%
7. 换 orjson + 减少返回字段 → 降到 200ms

如果火焰图显示大量时间在 select/epoll 但 CPU 空闲：
8. 说明在等 I/O，检查是不是串行 await → 改成 gather 并发
```

## 相关

- [[stdlib/builtin-data-structures]] —— 复杂度表
- [[internals/memory-model]] —— 内存分析
- [[internals/bytecode-execution]] —— 3.11+ 的解释器优化
- [[internals/gil]] —— 并行的限制
- [[concurrency/concurrency-models]] —— 并发选型
- [[web/fastapi-production]] —— 服务端性能调优清单
- [[web/fastapi-async-db]] —— N+1 与查询优化
