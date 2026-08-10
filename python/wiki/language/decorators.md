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

## 记忆点速查（复习先扫这里）

| # | 一句话 | 细节 |
|---|---|---|
| 1 | `@deco` ≡ `f = deco(f)`；`@deco(x)` ≡ `f = deco(x)(f)` | 开头 |
| 2 | **装饰器 = 闭包 + `*args/**kwargs` 转发**；`*`/`**` 在 `def` 处**打包**、在调用处**拆包** | §1 |
| 3 | `@wraps` 抄 `__name__`/`__doc__`/`__annotations__`+合并 `__dict__`+设 `__wrapped__`；**没它 `inspect.signature` 就瞎了**（FastAPI/pytest 直接坏） | §1 |
| 4 | `wraps` = `update_wrapper` 的 `partial`；后者**目标在前来源在后**、返回第一个参数、可用于类装饰器 | §1 |
| 5 | **层数**：无参两层，有参三层；兼容两种写法靠 `fn=None` + `partial` | §2 |
| 6 | 类装饰器装**方法**必须补 `__get__`（含 `obj is None` 分支），否则 `self` 丢失 | §3 |
| 7 | ⚠️ 装饰器实例是**类变量** → 状态**全实例共享**；要每实例就写进 `obj.__dict__` | §3 |
| 8 | `staticmethod`/`classmethod`/`property` **都只是描述符**，差别全在 `__get__` 返回什么 | §4 |
| 9 | `lru_cache` 三坑：参数须可哈希、装实例方法**泄漏实例**、无过期时间 | §4 |
| 10 | **顺序**：`@a @b @c f` = `a(b(c(f)))`——**装饰自下而上，调用自上而下** | §5 |
| 11 | 装饰 `async def`：wrapper 自己也要 `async def` + `await`；通用库用 `iscoroutinefunction` 分支 | §6 |
| 12 | 作用范围：装饰器=整个函数，CM=任意代码块，中间件=整个请求；`@contextmanager` 的产物**两者兼任** | §7 |

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

### `functools.update_wrapper`：`wraps` 底层真正干活的函数

`wraps(fn)` 只是 `update_wrapper` **预填了 `wrapped=fn` 的 `partial`**，所以两者等价：

```python
update_wrapper(wrapper, wrapped,                 # ⚠️ 目标在前，来源在后，写反会把原函数改名
               assigned=WRAPPER_ASSIGNMENTS,     # 覆盖：__module__ __name__ __qualname__
                                                 #       __doc__ __annotations__ __type_params__
               updated=WRAPPER_UPDATES)          # 合并：__dict__
# 外加一件常量表没写的事：设置 wrapper.__wrapped__ = wrapped
# 它原地修改并【返回第一个参数】，可以直接 return update_wrapper(w, fn)
```

**类装饰器只能用它**：`@wraps` 是装饰器语法，必须贴在 `def` 上；类装饰器没有内层函数，
包装器就是实例本身，只能在 `__init__` 里手动调（`setattr` 对实例同样有效）。

```python
class Counter:
    def __init__(self, fn):
        self.fn, self.calls = fn, 0
        functools.update_wrapper(self, fn)     # ← 没有 def 可以贴 @wraps

@Counter
def add(a, b=2):
    """把两个数相加"""
    return a + b

type(add)                  #=> <class 'Counter'>   ❗ 不是函数
add.__name__               #=> 'add'               不加则实例连 __name__ 都没有
inspect.signature(add)     #=> (a, b=2)            靠 __wrapped__ 穿透；不加则是 (*a, **k)
```

> **面试落点**：`wraps` 只能配 `def` 用，**类装饰器必须在 `__init__` 里手动调
> `update_wrapper(self, fn)`**。两者都只修「内省结果」，不改变 wrapper 实际接受的参数
> （仍是 `*args, **kwargs`）；静态类型层面还要配 `ParamSpec` 才完整。

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
        functools.update_wrapper(self, fn)   # 类版的 wraps，见上节
        self.fn = fn
        self.count = 0
    def __call__(self, *args, **kwargs):
        self.count += 1
        return self.fn(*args, **kwargs)
    def __get__(self, obj, objtype=None):    # ⚠️ 装饰实例方法时必须补描述符协议
        if obj is None:                      # 从【类】上访问：返回装饰器自身
            return self                      #   漏掉这个分支会把 None 当成 self
        return functools.partial(self.__call__, obj)

@CountCalls
def ping(): ...
ping(); ping()
ping.count       #=> 2
```

**`__get__` 补的是「函数自动绑定 `self`」这个能力**。普通函数是描述符（实现了 `__get__`），
`obj.method` 会自动把 `obj` 塞成第一个参数；`CountCalls` 实例默认不是描述符，`self` 就丢了：

```python
class Svc:
    @CountCalls
    def m(self, x): return x * 2

Svc().m(3)      # 没有 __get__ 时 ❌ TypeError: m() missing 1 required positional argument: 'x'
                # 有 __get__ 后   #=> 6    partial 预填了 obj，等于手工复刻函数的绑定行为
```

`obj is None` 分支对应「从类访问」（`Svc.m`），`property`/`staticmethod` 等内置描述符
都有这个分支。完整的描述符协议见 [[language/descriptors-properties]]。

### ⚠️ `count` 是所有实例共享的

装饰发生在**类体执行时**——那一刻还没有任何 `Svc` 实例，只创建了**一个** `CountCalls`
实例并存进类的命名空间。**它就是一个类变量**，`count` 存在它身上，与 `Svc` 的实例无关：

```text
    Svc.__dict__['m'] ──→ CountCalls 实例 { fn, count }   ← 计数器在这里，全局唯一
                              ↑         ↑
        a.m ─partial(obj=a)───┘         └───partial(obj=b)─ b.m
```

```python
a, b = Svc(), Svc()
a.__dict__                          #=> {}      实例字典里什么都没有
a.m.func.__self__ is b.m.func.__self__   #=> True   两个 partial 包的是同一个装饰器
a.m(1); a.m(2); b.m(3)
Svc.__dict__['m'].count             #=> 4      ❗ 连同上面那次 Svc().m(3)，全加在一起了
```

想要每实例独立，必须把状态写进 `obj.__dict__`（在 `__get__` 里用
`obj.__dict__.setdefault(...)`）——`cached_property` 正是这么做的。反面教材是
§4 的 `lru_cache` 装饰实例方法：缓存字典同样挂在类级的装饰器上，key 里含 `self`，
于是**强引用住每一个实例导致内存泄漏**，根因与此处完全相同。

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

### 自己实现 `staticmethod` / `classmethod` / `property`（高频手撕题）

这三个内置装饰器**都只是描述符**——差别全在 `__get__` 里返回什么：

```python
from types import MethodType

class my_staticmethod:                       # 非数据描述符
    def __init__(self, fn): self.fn = fn
    def __get__(self, obj, objtype=None):
        return self.fn                       # 原样返回，【不绑定任何东西】
    def __call__(self, *a, **k):             # 3.10+ 的真品也能直接调用
        return self.fn(*a, **k)

class my_classmethod:                        # 非数据描述符
    def __init__(self, fn): self.fn = fn
    def __get__(self, obj, objtype=None):
        if objtype is None: objtype = type(obj)
        return MethodType(self.fn, objtype)  # 绑定到【类】而不是实例

class my_property:                           # 有 __set__ → 数据描述符
    def __init__(self, fget=None, fset=None):
        self.fget, self.fset = fget, fset
        self.__doc__ = getattr(fget, "__doc__", None)
    def __get__(self, obj, objtype=None):
        if obj is None: return self          # 从类访问：返回描述符自身
        return self.fget(obj)                # 从实例访问：【调用】它
    def __set__(self, obj, value):
        if self.fset is None: raise AttributeError("can't set attribute")
        self.fset(obj, value)
    def setter(self, fset):
        return type(self)(self.fget, fset)   # 返回【新对象】，所以两个函数必须同名
```

代码里两个名字的含义：

- **`objtype`** 是「**从哪个类访问到的**」，不是「定义在哪个类上」——`Sub.create` 传进来的是
  `Sub`，哪怕 `create` 定义在 `C` 里。**这正是 `classmethod` 配合继承能拿到子类的原因。**
  `if objtype is None` 只为手动调 `descr.__get__(obj)`（省略第二个参数）时兜底。
- **`MethodType(fn, obj)`** 手工造一个**绑定方法**——`type(d.m)` 就是 `types.MethodType`，
  调用时自动把 `obj` 塞成第一个参数，**≈ JS 的 `fn.bind(obj)`**。比 `partial` 多了
  `__self__`/`__func__`，`repr` 也好读，`inspect.ismethod` 认得它；代价是只能绑第一个位置参数。

```python
class C:
    def __init__(self, w=3): self._w = w
    @my_staticmethod
    def util(x): return x * 10
    @my_classmethod
    def create(cls, w): return cls(w)
    @my_property
    def w(self): return self._w
    @w.setter
    def w(self, v):
        if v <= 0: raise ValueError("必须为正")
        self._w = v

class Sub(C): pass

C.util(2)                #=> 20      type(C.util) 就是 function，没有 self
Sub.create(1)            #=> Sub 实例  ❗ cls 是 Sub 不是 C —— 这正是 classmethod 的价值
c = C(); c.w = 8; c.w    #=> 8
c.w = -1                 # ❌ ValueError: 必须为正
c.__dict__["w"] = 999
c.w                      #=> 8       数据描述符优先级高于实例字典，遮不住
```

| | `__get__` 返回 | 描述符类型 |
|---|---|---|
| 普通函数 | `MethodType(fn, obj)` —— 绑定**实例** | 非数据 |
| `staticmethod` | `fn` 本身 —— **不绑定** | 非数据 |
| `classmethod` | `MethodType(fn, cls)` —— 绑定**类** | 非数据 |
| `property` | `fget(obj)` —— 直接**调用** | **数据** |

> **面试落点**：「`staticmethod` 是怎么实现的」——**它就是个 `__get__` 里原样返回原函数的
> 描述符**，「不接收 self」不是什么特殊语法，只是没有走绑定这一步。`classmethod` 绑的是
> `objtype` 所以子类继承时 `cls` 是子类；`property` 因为定义了 `__set__` 是**数据描述符**，
> 优先级高于实例字典。三个加起来能手写出来，descriptors 那一整章就算通了。

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

**「每实例的 `lru_cache`」的写法**——不在类体上装饰，而是在 `__init__` 里为每个实例单独造一个：

```python
class Good:
    def __init__(self):
        self.get = functools.lru_cache(maxsize=None)(self._get)   # ← 包的是【已绑定】的方法
    def _get(self, k): return k * 2

g1, g2 = Good(), Good()
g1.get(1); g1.get(1); g2.get(1)
g1.get.cache_info()   #=> CacheInfo(hits=1, misses=1, ...)
g2.get.cache_info()   #=> CacheInfo(hits=0, misses=1, ...)   两个缓存完全独立
g1.get is g2.get      #=> False
```

因为包的是 `self._get`，缓存 key 里**不再含 `self`**，缓存对象作为实例属性存在
`g1.__dict__` 里。代价是造出了**循环引用**（实例 → `self.get` → 绑定方法 → 实例），
引用计数回收不了，要等分代 GC——但**能回收，只是晚一点**，与类级 `lru_cache` 的
「永远回收不了」有本质区别：

```python
import gc, weakref
r = weakref.ref(g1); del g1
r() is None           #=> False   仅 del 还在（循环引用）
gc.collect()
r() is None           #=> True    ✅ GC 之后回收

# 对照：类级 lru_cache 的实例，del + gc.collect() 之后依然存活
```

| 场景 | 用什么 |
|---|---|
| 方法无参数（只是懒加载） | `@cached_property` —— 进 `obj.__dict__`，无循环引用，最干净 |
| 方法有参数、实例不多 | 每实例 `lru_cache`（上面这招） |
| 实例极多 / 要精确控制 | `cachetools.cachedmethod` + 每实例 dict，或自己用 `WeakKeyDictionary` |

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
