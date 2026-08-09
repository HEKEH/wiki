---
title: "题库：语言核心"
date: 2026-08-07
tags: [面试题, 语言核心, 装饰器, 生成器, MRO, 参考答案]
sources: ["interview-python-cn.md", "interview-bible-cn.md"]
---

# 题库：语言核心

**用法**：先自己答，再看参考答案。参考答案是"面试现场可以直接说出来"的长度，
展开细节点原理页链接。

## A. 对象与可变性

**Q1. Python 的参数传递是值传递还是引用传递？**

> 都不是，是 **call by object reference（传对象引用）**。
> 函数拿到的是实参对象引用的副本。**能不能影响外部取决于你做的是就地变更还是重新绑定**：
> `lst.append(x)` 外部可见，`lst = [...]` 外部不可见。
> 因为 `int`/`str`/`tuple` 不可变，对它们的任何"修改"都是重新绑定，所以看起来像值传递。
> 详见 [[language/objects-mutability]]。

**Q2. `is` 和 `==` 的区别？为什么 `a = 256; b = 256; a is b` 是 True 而 257 不是？**

> `is` 比较身份（`id()`），`==` 比较值（走 `__eq__`）。
> CPython 预先缓存了 `-5~256` 的小整数对象，所以 256 共享同一个对象。
> **但这是 CPython 的实现细节**，不是语言规范，业务代码绝不能依赖。
> `is` 的正当用途只有 `is None` / `is True/False` / 哨兵比较。

**Q3. 深拷贝和浅拷贝的区别？`deepcopy` 怎么处理循环引用？**

> 浅拷贝（`x[:]`、`copy.copy`）只复制最外层容器，内层对象仍是共享引用；
> 深拷贝递归复制所有层级。`deepcopy` 内部用一个 **memo 字典**记录"已复制对象的 id → 副本"，
> 遇到已复制过的对象直接返回副本，因此能正确处理循环引用而不栈溢出。

**Q4. 可变默认参数为什么危险？**

> 默认值在**函数定义时求值一次**，保存在 `func.__defaults__` 里，所有调用共享同一个对象。
> 所以 `def f(lst=[])` 的 lst 会跨调用累积。用 `None` 哨兵解决。
> `dataclass` 会直接报错逼你用 `default_factory`，Pydantic 则会为每个实例深拷贝默认值。

## B. 作用域与闭包

**Q5. 说说 LEGB。Python 有块级作用域吗？**

> Local → Enclosing（外层**函数**）→ Global（模块）→ Builtin。
> **没有块级作用域**——只有函数、类、模块、推导式创建作用域，
> `if`/`for`/`while`/`with` 不创建，所以循环变量会泄漏。相当于 JS 只有 `var` 没有 `let`。

**Q6. 什么是闭包？`nonlocal` 和 `global` 的区别？**

> 闭包 = 内层函数 + 它引用的外层函数变量（存在 `__closure__` 的 cell 对象里）。
> 捕获的是**变量本身**不是值，所以多个闭包共享同一个 cell。
> `global` 绑定到模块级，`nonlocal` 绑定到最近的 enclosing **函数**作用域（不能是全局）。
> **只读不需要声明，重新绑定才需要。**

**Q7. `[lambda: i for i in range(3)]` 全部返回 2，为什么？怎么修？**

> 延迟绑定：lambda 捕获的是变量 `i`，循环结束时 `i == 2`。
> 修法：`lambda i=i: i`（默认参数在定义时求值）、工厂函数、`functools.partial`。
> 这就是 ES5 `var` + `setTimeout` 的同一个问题，JS 用 `let` 的每轮新绑定解决了它。

**Q8. 下面为什么报 UnboundLocalError？**

```python
x = 1
def f():
    print(x)
    x = 2
```

> 因为函数体内有 `x = 2` 这个赋值，**编译期**就把 `x` 判定为局部变量
> （写进 `co_varnames`，生成 `LOAD_FAST`），所以 `print(x)` 时局部 x 还没赋值。
> 要读全局就加 `global x`。

## C. 装饰器

**Q9. 装饰器的本质是什么？`@a @b def f` 的执行顺序？**

> 本质是"接收可调用对象、返回可调用对象"的高阶函数，`@` 只是 `f = deco(f)` 的语法糖。
> `@a @b def f` 等价 `f = a(b(f))`——**装饰自下而上，调用自上而下**。

**Q10. 为什么装饰器要用 `functools.wraps`？**

> 不加的话被装饰函数的 `__name__`/`__doc__`/`__qualname__`/`__module__`/`__dict__`
> 都会变成 wrapper 的，**签名也会退化成 `(*args, **kwargs)`**。
> 这会破坏调试信息、文档工具，以及**任何依赖 `inspect.signature` 的框架**——
> 比如 FastAPI 的依赖注入、pytest 的 fixture 注入。`wraps` 还会挂上 `__wrapped__` 方便取原函数。

**Q11. 写一个带参数的重试装饰器。**

```python
import functools, time

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
                    time.sleep(delay * 2 ** (attempt - 1))
        return wrapper
    return decorator
```

> 追问"怎么同时支持异步函数"：用 `inspect.iscoroutinefunction(fn)` 分支，
> 异步分支的 wrapper 本身是 `async def` 且内部 `await fn(...)`。

**Q12. `@staticmethod`、`@classmethod`、`@property` 分别是什么？**

> 三者都是**描述符**。`staticmethod` 不绑定任何东西；`classmethod` 绑定到类（`cls`），
> 用于替代构造器和多态工厂；`property` 是**数据描述符**，把方法伪装成属性，
> 让你能在不改调用方代码的前提下加校验/惰性计算。详见 [[language/descriptors-properties]]。

## D. 迭代器与生成器

**Q13. 可迭代对象、迭代器、生成器的区别？**

> **Iterable** 实现 `__iter__`（可以被 `iter()`）；
> **Iterator** 同时实现 `__iter__` 和 `__next__`，**一次性、有状态、耗尽即废**；
> **Generator** 是用 `yield` 或生成器表达式创建的特殊迭代器。
> `list` 是 iterable 不是 iterator——`next(list)` 会报错。

**Q14. 生成器的好处？`yield` 时函数发生了什么？**

> 惰性求值 + 常数级内存。`yield` 时**整个函数帧（局部变量 + 指令指针）被保留**，
> 控制权交回调用方；下次 `next()` 从挂起点继续。这正是协程的基础机制。

**Q15. `yield from` 除了简写循环还做了什么？**

> 它还**透传 `send()`/`throw()`/`close()` 给子生成器**，
> 并把子生成器 `return` 的值作为 `yield from` 表达式的值。
> 这是 PEP 380 的核心，也是 asyncio 早期"基于生成器的协程"的实现基础——
> `await` 在字节码层面与 `yield from` 高度相似。

**Q16. 列表推导式和生成器表达式怎么选？**

> 需要多次遍历、需要索引、需要 `len` → 列表推导式；
> 只遍历一次、数据量大、做聚合（`sum`/`any`/`max`/`join`）→ 生成器表达式。
> 后者内存是常数级：`sys.getsizeof` 一个百万元素的生成器只有 200 字节。

## E. 类与面向对象

**Q17. 说说 MRO。`super()` 是不是父类？**

> MRO 是 C3 线性化算法算出的方法解析顺序，保证子类优先、保持声明顺序、单调性。
> **`super()` 不是"父类"，而是 MRO 中当前类的下一个类**——
> 在菱形继承 `D(B, C)` 里，`B.method` 中的 `super()` 指向 **C** 而不是 A。
> 这就是协作式多重继承。详见 [[language/classes-mro]]。

**Q18. `__new__` 和 `__init__` 的区别？**

> `__new__` 是构造器，负责**创建并返回实例**（隐式 staticmethod，首参是 `cls`）；
> `__init__` 是初始化器，给已创建的实例填状态（首参 `self`，必须返回 `None`）。
> `__new__` 返回的不是 cls 的实例时 `__init__` 不会被调用。
> **子类化不可变类型（int/str/tuple）必须在 `__new__` 里定制**。

**Q19. Python 有私有变量吗？`_x` 和 `__x` 的区别？**

> **没有真正的私有**。`_x` 是纯约定（`import *` 不导出）；
> `__x` 会触发**名字改写**变成 `_ClassName__x`，
> 目的是**避免多重继承时的命名冲突**，不是访问控制——`obj._A__x` 照样能访问。

**Q20. 类变量和实例变量的查找顺序？**

> 读：先实例 `__dict__`，再沿 MRO 查类——**但数据描述符优先于实例字典**。
> 完整顺序：数据描述符 > 实例 `__dict__` > 非数据描述符 > 普通类属性 > `__getattr__`。
> 写：默认写进实例 `__dict__`（除非有数据描述符的 `__set__`）。
> 所以可变类变量（`items = []`）会被所有实例共享，是经典 bug。

**Q21. 什么是元类？什么时候用？**

> 元类是"类的类"，控制类的创建过程（`type` 是默认元类）。
> `class` 语句本质是调用 `metaclass(name, bases, namespace)`。
> **但 99% 的场景不该用**——3.6 起 `__init_subclass__`（子类注册）、
> `__set_name__`（描述符命名）、类装饰器已覆盖绝大多数需求，
> 而且不会引入元类冲突。我只在写框架、需要 `__prepare__` 或控制实例化（`__call__`）时才考虑。

**Q22. 鸭子类型是什么？怎么表达"接口"？**

> 鸭子类型：不看类型，只看有没有对应方法。
> 表达接口有三种：`abc.ABC`（名义子类型，必须显式继承，实例化时检查）、
> `typing.Protocol`（**结构化子类型**，等价 TS 的 interface，静态检查）、
> 纯鸭子类型（不声明）。库的公开 API 我倾向 Protocol。

## F. 异常与上下文管理

**Q23. `except Exception` 能捕获所有异常吗？**

> 不能。`KeyboardInterrupt`、`SystemExit`、`GeneratorExit`（以及 3.8+ 的
> `asyncio.CancelledError`）继承自 `BaseException` 而非 `Exception`。
> 这是刻意设计——兜底的 `except` 不该让程序无法被 Ctrl+C 终止。

**Q24. `try/except/else/finally` 里 `else` 有什么用？**

> `else` 只在 try 块**没有异常**时执行，作用是**缩小 try 的保护范围**，
> 避免误捕获后续代码抛出的同类异常。
> 另外：**在 `finally` 里 `return` 会吞掉异常**，绝对不要写。

**Q25. 为什么一定要用 `with open()`？**

> 不是风格问题，是**正确性问题**。它保证异常路径下也释放资源。
> CPython 因为引用计数会让 `open(p).read()` "碰巧"及时关闭文件，
> 但**这是实现细节**——PyPy 用纯 GC，文件会延迟关闭直到 fd 耗尽。

**Q26. EAFP 和 LBYL 怎么选？**

> Python 偏好 **EAFP**（先做再处理异常），因为它消除 TOCTOU 竞态，
> 且 3.11 起是"零成本异常"——无异常时 try 的开销接近零。
> **但异常路径很贵**，所以预期失败率高时（比如缓存命中率只有 10%）仍应 LBYL。

## G. 字符串与编码

**Q27. Python 2 和 3 最大的区别？**

> 最重要的是**字符串模型**：Python 3 的 `str` 是 Unicode 文本、`bytes` 是字节，
> 二者**不能隐式转换**；Python 2 的 `str` 其实是字节串，隐式转换导致了大量
> `UnicodeDecodeError`。其余：print 变函数、`/` 变真除法、dict 的 keys/values 返回视图、
> 只有新式类、range 惰性、异常语法。**Python 2 已于 2020 年 EOL。**

**Q28. 为什么循环里 `s += x` 不好？**

> 字符串不可变，每次 `+=` 都创建新对象，理论上是 O(n²)。用 `"".join(parts)`。
> （CPython 对 refcount==1 时有原地扩容优化，所以你可能测不出来，但那是脆弱的实现细节。）

## H. 综合速答

| 问题 | 一句话答案 |
|---|---|
| Python 是解释型语言吗 | 先**编译成字节码**再由虚拟机解释执行，模型类似 Java |
| `.pyc` 能加速吗 | 只加速**导入**（省编译），不加速运行 |
| Python 支持函数重载吗 | 不支持，后定义覆盖前者；用默认参数/`*args`/`singledispatch`/`typing.overload` |
| Python 有尾递归优化吗 | 没有，Guido 明确拒绝；深递归改写成循环 |
| `__slots__` 有什么用 | 取消 `__dict__`，省内存 + 加快属性访问，代价是失去动态性 |
| 类型注解运行时有效吗 | 默认**无任何检查**，只存进 `__annotations__`；Pydantic 才让它有运行时效力 |
| 列表和元组的区别 | 可变性 + 可哈希性 + 语义（同质序列 vs 固定结构记录） |
| `sorted` 的算法 | **Timsort**，最坏 O(n log n)，**稳定**，对部分有序数据接近 O(n) |
| 单例怎么实现 | 模块级变量（最简）/ `__new__` / 元类 `__call__` / 装饰器 / `functools.cache` |

## 相关

- [[interview/question-bank-internals-concurrency]] —— 原理与并发题库
- [[interview/question-bank-web]] —— Web 与工程题库
- [[interview/traps]] —— 陷阱题
- [[interview/roadmap]] —— 复习路线
- [[sources/interview-python-cn]] —— 中文题库源导读
