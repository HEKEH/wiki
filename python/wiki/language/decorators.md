---
title: "装饰器（Decorators）"
date: 2026-08-07
tags: [装饰器, 闭包, functools, wraps, AOP, 类装饰器]
sources: ["interview-python-cn.md", "python-cheatsheet.md", "python-patterns.md"]
---

# 装饰器（Decorators）

装饰器是 Python 面试的**必考题**，也是读懂 Flask/FastAPI/pytest/Django 源码的门槛。
本质极其简单：**装饰器是一个接收函数（或类）、返回新函数（或类）的可调用对象**，`@` 只是语法糖。

```python
@deco
def f(): ...

# 完全等价于：
def f(): ...
f = deco(f)
```

> 前端类比：≈ 高阶函数包裹（`withRouter(Component)`、Express 中间件），
> 也 ≈ TypeScript 的 `@Injectable()` 装饰器（TS 装饰器正是借鉴自 Python）。

## 1. 最简装饰器

```python
import functools, time

def timer(fn):
    @functools.wraps(fn)                    # ← 关键，见下
    def wrapper(*args, **kwargs):
        t0 = time.perf_counter()
        try:
            return fn(*args, **kwargs)
        finally:
            print(f"{fn.__name__} took {time.perf_counter() - t0:.3f}s")
    return wrapper

@timer
def work(n):
    """做一些事"""
    return sum(range(n))

work(10**6)
```

### `functools.wraps` 为什么必须加

不加 `wraps`，被装饰函数的元信息会被 wrapper 覆盖：

```python
work.__name__     # 不加 wraps →  'wrapper'；加了 → 'work'
work.__doc__      # 不加 wraps →  None；     加了 → '做一些事'
work.__wrapped__  # wraps 额外挂上原函数，便于取消装饰/内省
inspect.signature(work)   # 不加 wraps → (*args, **kwargs)，签名丢失
```

签名丢失会**直接打断依赖内省的框架**（FastAPI 靠签名解析参数、pytest 靠签名注入 fixture）。

> **面试落点**：「装饰器为什么要用 `functools.wraps`」——保留 `__name__`/`__doc__`/`__module__`/
> `__qualname__`/`__dict__` 和 `__wrapped__`，否则调试信息、文档工具、依赖签名内省的框架全部失效。

## 2. 带参数的装饰器（三层）

```python
def retry(times=3, exceptions=(Exception,), delay=0.1):
    def decorator(fn):
        @functools.wraps(fn)
        def wrapper(*args, **kwargs):
            for attempt in range(1, times + 1):
                try:
                    return fn(*args, **kwargs)
                except exceptions:
                    if attempt == times:
                        raise
                    time.sleep(delay * 2 ** (attempt - 1))   # 指数退避
        return wrapper
    return decorator

@retry(times=5, exceptions=(ConnectionError,))
def fetch(url): ...
```

层数记忆法：**`@deco` 无参数 → 两层；`@deco(...)` 有参数 → 三层**。
因为 `@retry(times=5)` 先执行 `retry(times=5)` 得到 `decorator`，再用它装饰。

### 兼容"带括号和不带括号"两种用法

```python
def logged(fn=None, *, level="INFO"):
    if fn is None:                          # 被当作 @logged(level="DEBUG") 使用
        return functools.partial(logged, level=level)
    @functools.wraps(fn)
    def wrapper(*a, **kw):
        print(f"[{level}] calling {fn.__name__}")
        return fn(*a, **kw)
    return wrapper

@logged                       # ✅
def a(): ...

@logged(level="DEBUG")        # ✅
def b(): ...
```

## 3. 类装饰器与装饰类

**用类实现装饰器**（便于保存状态）：

```python
class CountCalls:
    def __init__(self, fn):
        functools.update_wrapper(self, fn)   # 类版的 wraps
        self.fn = fn
        self.count = 0
    def __call__(self, *args, **kwargs):
        self.count += 1
        return self.fn(*args, **kwargs)
    def __get__(self, obj, objtype=None):    # ⚠️ 装饰实例方法时必须补描述符协议
        return functools.partial(self.__call__, obj)

@CountCalls
def ping(): ...
ping(); ping()
ping.count       #=> 2
```

**装饰类**（返回修改后的类）：

```python
def singleton(cls):
    instances = {}
    @functools.wraps(cls)
    def get(*a, **kw):
        if cls not in instances:
            instances[cls] = cls(*a, **kw)
        return instances[cls]
    return get

@singleton
class Config: ...

Config() is Config()     #=> True
```

`dataclass`、`functools.total_ordering` 都是类装饰器。

## 4. 标准库里必须会的装饰器

```python
class C:
    @staticmethod                # 不接收 self/cls，就是挂在类命名空间的普通函数
    def util(x): return x

    @classmethod                 # 接收 cls，常用于替代构造器
    def from_json(cls, s): return cls(**json.loads(s))

    @property                    # 把方法伪装成属性（见 descriptors 页）
    def area(self): return self.w * self.h

    @area.setter
    def area(self, v): ...

    @functools.cached_property   # 只算一次，结果写进实例 __dict__
    def heavy(self): return expensive()

@functools.lru_cache(maxsize=128)     # 记忆化，参数必须可哈希
def fib(n): return n if n < 2 else fib(n-1) + fib(n-2)

@functools.cache                       # 3.9+，等价 lru_cache(maxsize=None)
def f(x): ...

fib(100)                               # 瞬间返回
fib.cache_info()   #=> CacheInfo(hits=98, misses=101, maxsize=None, currsize=101)
fib.cache_clear()
```

`lru_cache` 的坑（常考）：

- 参数必须**可哈希**（`list`/`dict` 参数会 TypeError）。
- 用在**实例方法**上会**持有 self 的强引用 → 内存泄漏**，实例永远不被回收。
  实例级缓存应该用 `cached_property` 或每实例的 `lru_cache`。
- 缓存无过期时间、非线程安全边界（CPython 里 `lru_cache` 本身是线程安全的，但被缓存函数的副作用不是）。

```python
class Service:
    @lru_cache                # ❌ 缓存字典挂在类上，key 含 self，实例泄漏
    def get(self, k): ...

    @cached_property          # ✅ 结果存在实例 __dict__，随实例一起回收
    def config(self): ...
```

## 5. 多个装饰器的顺序

```python
@a
@b
@c
def f(): ...
# 等价于 f = a(b(c(f)))
```

**从下往上应用，从上往下执行**（最靠近函数的先包裹，最外层先被调用）。

```python
@app.route("/x")      # 最外层：注册路由，拿到的是已鉴权、已限流的函数
@require_auth         # 中间
@rate_limit           # 最内层：最贴近业务函数
def handler(): ...
```

> **面试落点**：顺序问题几乎每次都问。记住 `@a @b f` = `a(b(f))`：
> **装饰是自下而上，调用栈是自上而下**。

## 6. 装饰器的真实应用场景（AOP）

| 场景 | 例子 |
|---|---|
| 路由注册 | Flask `@app.route`、FastAPI `@app.get` |
| 缓存 | `lru_cache`、Django `@cache_page` |
| 权限/鉴权 | `@require_login` |
| 重试 / 限流 / 熔断 | `tenacity.retry` |
| 事务 | `@transaction.atomic` |
| 日志 / 指标 / 链路追踪 | OpenTelemetry `@tracer.start_as_current_span` |
| 测试 | `@pytest.fixture`、`@pytest.mark.parametrize`、`@mock.patch` |
| 类型/校验 | `@validate_call`（Pydantic） |

**装饰异步函数**要区分对待：

```python
def async_timer(fn):
    @functools.wraps(fn)
    async def wrapper(*a, **kw):        # ← wrapper 本身必须是 async def
        t0 = time.perf_counter()
        try:
            return await fn(*a, **kw)   # ← 必须 await
        finally:
            print(time.perf_counter() - t0)
    return wrapper
```

通用装饰器要用 `inspect.iscoroutinefunction(fn)` 分支处理同步/异步两种情况——
这是写库时的高频考点。

```python
def timed(fn):
    if inspect.iscoroutinefunction(fn):
        @functools.wraps(fn)
        async def aw(*a, **kw): ...
        return aw
    @functools.wraps(fn)
    def w(*a, **kw): ...
    return w
```

## 7. 装饰器 vs 上下文管理器 vs 中间件

| | 作用范围 | 典型写法 |
|---|---|---|
| 装饰器 | 整个函数 | `@timer def f()` |
| 上下文管理器 | 任意代码块 | `with timer():` |
| 中间件 | 整个请求 | ASGI middleware |

`contextlib.ContextDecorator` 让一个对象**同时**能当两者用：

```python
from contextlib import contextmanager

@contextmanager
def timer(label):
    t0 = time.perf_counter()
    try:
        yield
    finally:
        print(label, time.perf_counter() - t0)

with timer("block"):        # ✅ 当上下文管理器
    work()

@timer("func")              # ✅ 也能当装饰器！@contextmanager 返回的对象继承了 ContextDecorator
def work2(): ...
```

## 相关

- [[language/scope-closure]] —— 装饰器 = 闭包
- [[language/functions-arguments]] —— `*args/**kwargs` 转发与签名内省
- [[language/descriptors-properties]] —— `property`/`staticmethod` 其实是描述符
- [[language/context-managers]] —— `@contextmanager` 与 ContextDecorator
- [[stdlib/collections-itertools]] —— `lru_cache` / `partial` / `singledispatch`
- [[web/fastapi-core]] —— `@app.get` 路由装饰器的实现思路
- [[interview/question-bank-language]] —— 装饰器面试题
