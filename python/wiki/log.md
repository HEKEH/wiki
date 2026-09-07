# Log

## [2026-08-07] ingest | 建库：Python 知识库初始化 + 7 组源材料

**触发**：用户为「前端工程师转 Python、冲刺高难度 Python 面试」建库；
会话中追加明确 **FastAPI 是关键技能** → 技术方向定为**后端 Web**，FastAPI 设为一等公民分类。

**抓取源材料**（`raw/`，41 个文件，约 1.4 MB，全部来自 GitHub raw）：

| 源 | 文件 | 说明 |
|---|---|---|
| CPython 官方文档 | `cpython-doc/` 4 个 rst（341 KB） | datamodel / descriptor-howto / functional-howto / faq-programming |
| PEP | 3 个 rst（186 KB） | PEP 8（风格）、PEP 492（async/await）、PEP 703（free-threading） |
| FastAPI 官方文档 | `fastapi-doc/` 24 个 md（约 250 KB） | async / python-types / virtual-environments / tutorial×14 / advanced×5 / deployment×2 |
| Pydantic 官方文档 | `pydantic-doc/` 4 个 md（124 KB） | models / validators / fields / performance |
| gto76/python-cheatsheet | `python-cheatsheet.md`（150 KB） | 语法与标准库速查 |
| satwikkansal/wtfpython | `wtfpython.md`（125 KB） | 陷阱与边界情况 |
| 中文面试题库 | `interview-python-cn.md` + `interview-bible-cn.md`（147 KB） | taizilongxu/interview_python、jackfrued/Python-Interview-Bible |
| faif/python-patterns | `python-patterns.md`（12 KB） | 设计模式索引 |

**产出**：`CLAUDE.md`（domain schema，定义 10 个分类 + 代码示例规范 + 面试落点标记约定）
及 **61 个 wiki 页面**（58 个内容页 + 3 个导航页）：

```text
language/     16 页   数据模型、可变性、作用域闭包、参数、装饰器、迭代器生成器、
                     上下文管理器、类与 MRO、描述符、元类、类型注解、异常、
                     数据类、推导式与函数式、字符串编码、模块导入
internals/     5 页   对象模型、垃圾回收、GIL、字节码执行、内存模型
concurrency/   5 页   并发选型、threading、multiprocessing、asyncio 原理、asyncio 实战
web/           9 页   WSGI/ASGI、FastAPI 核心/DI/架构/测试/生产、Pydantic、异步 DB、鉴权
stdlib/        3 页   内置数据结构与复杂度、collections/itertools/functools、标准库速查
engineering/   4 页   包管理(uv)、pytest、ruff/mypy 工具链、性能剖析
bridge/        3 页   JS→Python 语法心智、两种事件循环、npm↔Python 生态
interview/     6 页   路线图、3 个分主题题库、手撕代码模板、陷阱题
sources/       6 页   各源材料导读（含中文题库的过时内容标注表）
analysis/      1 页   前端转 Python 的知识差距分析
导航           3 页   home / index / log
```

**关键决策与偏离源材料之处**：

1. **面向 Python 3.12+ 写作**，中文题库里的 Python 2 内容（新式类/旧式类、`xrange`、
   `%` 格式化、`has_key`）在 [[sources/interview-python-cn]] 中列了**专门的过时对照表**，
   正文按现状重写。
2. **数据库层用 SQLAlchemy 2.0 async 而非官方教程的 SQLModel**——
   生产主流，且能讲清 session 作用域、N+1、事务这些面试深挖点。
3. **参数声明统一用 `Annotated`**（官方推荐的现代写法），不用旧的
   `q: str = Query(None)` 模式。
4. **每个概念页都加 `> 面试落点` 引用块**，可单独扫读做快速复习。
5. **贯穿"JS 里的 X ≈ Python 里的 Y，但差别在 Z"的对照式讲解**——
   这是本库针对读者背景的核心写作约定，写进了 `CLAUDE.md`。

**验证**：关键结论在 **CPython 3.13.3** 上实测，包括
类作用域推导式的 NameError、`getsizeof` 各类型数值、list/dict 扩容序列、
[[interview/traps]] 的 12 个陷阱（链式比较、dict 键合并、惰性生成器、`round` 银行家舍入、
`tuple += list` 的双重语义、`except` 变量删除、NaN 集合行为、`zip` 有损、
整数字符串转换限制、字符串隐式拼接等），以及 [[interview/coding-patterns]] 的
链表反转三重赋值与 `[[0]*n]*m` 陷阱。实测与初稿不符处已修正（如 `getsizeof("")` 为 41 而非 49）。

**已知缺口**（记在 [[home]] 的开放问题）：SQL 与数据库原理、分布式基础、
Django 方向、数据/AI 方向、真实项目、free-threading 实测。

## [2026-08-07] lint | 建库后自查：修正 6 处错误 + 补 1 页缺环

用户要求复查全库。用 CPython 3.13 重新实测了此前未验证的数值断言，并做了跨页一致性检查。

**修正的错误**：

1. **[[internals/memory-model]] 的内存对比表数值和排序都错了**——实测 100 万二维点：
   dict 175 MB（原写 240）、普通实例 130 MB（原写 180）、NamedTuple 53 MB（原写 70）、
   `__slots__` 46 MB（原写 90）。**且原表把 NamedTuple 排在 `__slots__` 之前，
   实测恰好相反**——`__slots__` 类比 NamedTuple 更省。已改并加注说明选 NamedTuple 的
   理由应该是不可变与可解包而非省内存。
2. **[[language/descriptors-properties]] 的 `__slots__` 内存数字错了**——
   实测 344 字节（48 + `__dict__` 296）vs 48 字节，原写 152 vs 56。
3. **[[interview/traps]] 第 6 题的示例选错了输入**——`[1,2,3,4]` 删偶数**碰巧结果正确**，
   演示不出 bug。改为 `[1,2,2,3]`（漏删第二个 2），并补上逐步索引追踪。
4. **两个线程池被混为一谈**——FastAPI 跑 `def` 路由用的是 Starlette/anyio 的池（默认 **40**），
   `asyncio.to_thread` 用的是事件循环默认 executor（**`min(32, cpu+4)`**，本机实测 16）。
   两处原文各自正确但读者易混，已在 [[web/fastapi-core]] 加对比表、
   [[concurrency/asyncio-patterns]] 加交叉说明。
5. 页数与源文件数统计前后不一致（58/61、39/40），已统一。
6. [[language/iterators-generators]] 的生成器内存数字 8.4 MB → 实测 8.1 MB。

**补的缺环**：新增 [[concurrency/io-multiplexing]]（select/poll/epoll、五种 IO 模型、
LT/ET、C10K、`selectors` 手写事件循环、asyncio 的完整调用链路）。

理由：① [[concurrency/asyncio-fundamentals]] 画出了 "Selector（epoll/kqueue/IOCP）"
却从未解释它，**讲 asyncio 讲不到它的物理底座是缺环**，而"asyncio 底层怎么实现的"
是深挖时的必然追问；② [[sources/interview-python-cn]] 的映射表原本声称覆盖了源材料的
网络篇，实际正文里没有 select/poll/epoll——**这是一张空头支票**，现已补上实质内容
并把未覆盖的部分（调度算法、分页分段、三次握手、MVCC 内部）显式标注为不覆盖。

**另外验证通过、未改动的**：14 项标准库版本可用性声明（`itertools.batched` 3.12+、
`asyncio.TaskGroup` 3.11+、`enum.StrEnum` 3.11+ 等）全部属实；
`@contextmanager` 返回对象可直接当装饰器用；`lru_cache` 装饰实例方法确实导致实例不被回收
（weakref 验证）；GIL 切换间隔、Timsort 稳定性等跨页表述一致。

## [2026-08-07] lint | 第二轮自查：8 处事实错误（含 3 处版本过时）

用户要求"只查错误"。把全库可测的 `#=>` 断言批量抽出，在 CPython 3.13 上逐条执行，
并与 3.8 / 3.12 / 3.14 交叉比对。共 200+ 条断言，**8 条为假**。

### A. 版本演进导致的过时（最严重的一类）

1. **`gc.get_threshold()` 不再是 `(700, 10, 10)`**——实测 3.8/3.12 是 `(700,10,10)`，
   **3.13 是 `(2000, 10, 10)`，3.14 是 `(2000, 10, 0)`（增量式 GC）**。
   本库原本在 5 处把 `(700,10,10)` 当作"必背数字"和"分水岭"。已全部改为标注版本区间，
   并把"主动补一句版本变化"转化为加分话术。
2. **经典竞态示例在现代 CPython 上跑不出丢失**——`counter += 1` 用 4 线程 ×500 万次，
   3.8 与 3.13 **都精确得到期望值，一次都没丢**。原因：**GIL 只在 `eval_breaker`
   检查点切换，紧凑循环的检查点在 `JUMP_BACKWARD`，`LOAD_GLOBAL→BINARY_OP→STORE_GLOBAL`
   之间没有检查点**。只要循环体里加一次函数调用（`CALL` 是检查点）立刻复现
   （实测丢失 40 万 / 21 万 / 21 万）。
   [[internals/gil]] 与 [[concurrency/threading]] 原文断言"counter 远小于 400000"**是错的**，
   已改写——并把这个反直觉结果转成更强的论据：
   **"观察不到竞态"≠"线程安全"，前者是实现细节，后者是语义保证。**
3. **pymalloc 是 16 字节对齐、32 个大小类**，不是原文的"8 字节对齐、64 个大小类"
   （那是 32 位时代的数字）。`PYTHONMALLOCSTATS=1` 实测：
   `Small block threshold = 512, in 32 size classes`，arena 确认为 1 MB。

### B. 语义理解错误

4. **`reversed()` 不在"老式迭代协议"回退范围内**——只实现 `__getitem__` 可以迭代、可以 `in`，
   但 `reversed()` 需要 `__len__`（实测 TypeError）。[[language/data-model]] 原文把
   `reversed` 一起列进了回退能力。
5. **列表推导式里的生成器表达式同样有延迟绑定问题**——
   `[(x for _ in range(2)) for x in range(3)]` 实测全部产出 `[2,2]`。
   [[language/iterators-generators]] 原文标注"这个是对的（推导式各自作用域）"**是错的**，
   已改为完整的机制解释与修法。
6. **tuple 的"去跟踪"优化是惰性的**——`gc.is_tracked((1,2))` 刚创建时是 `True`，
   **一次 `gc.collect()` 之后才变 `False`**。[[internals/garbage-collection]] 原文
   直接写 `#=> False`。

### C. 数值与示例瑕疵

7. `sys.getsizeof(生成器)` 是 **192** 不是 200。
8. [[language/strings-encoding]] 里 `f"{n:,}"` 的示例沿用了上文的 `n = 42`，
   却标注输出 `'1,234,567'`——已改为 `f"{1234567:,}"`。
   另修正 [[language/objects-mutability]] 中 `structuredClone` ≈ `deepcopy` 的类比：
   实测 **`copy.deepcopy(fn) is fn`（原样返回同一引用）**，而 `structuredClone` 遇函数**抛错**，
   原文"都不复制函数"含混。

**本轮未覆盖**：`web/` 与 `engineering/` 共 13 页依赖第三方库，当时无法实测（见下一条）。

**验证通过未改动的**（约 200 条）：MRO/C3 解析顺序、描述符四级优先级、
元类 `__prepare__→__new__→__init__→__call__` 顺序、`finally` 吞异常与覆盖返回值、
`functools.wraps` 的五项元信息与签名保留、装饰器"装饰自下而上/调用自上而下"、
`gather` 失败留下孤儿任务 vs `TaskGroup` 自动取消兄弟任务、协程惰性与"只能 await 一次"、
`Semaphore` 限流、`contextvars` 跨 Task 继承、`groupby` 不排序会分裂、
`hash(-1) == -2`、`{True:..., 1:...}` 同键、Timsort 稳定性、weakref 与
`WeakValueDictionary` 自动删键、字符串驻留与 `sys.intern`、正则 `match` vs `search`、
14 项标准库版本可用性。

## [2026-08-07] lint | 第三轮自查：装实测环境覆盖 web/ 层，1 处硬错误

前两轮把 `web/` 和 `engineering/` 共 13 页跳过了（依赖第三方库）。本轮在临时 venv 里装了
**FastAPI 0.141.1 / Pydantic 2.13.4 / SQLAlchemy 2.0.51 / Starlette 1.4.1**，
把这一层的行为断言逐条跑通。

### 唯一的硬错误：yield 依赖收尾 vs 后台任务的先后

原文（3 处）断言「**`BackgroundTasks` 在 yield 依赖收尾之后才跑，所以后台任务里
不能用请求级 DB session（那时它已关闭）**」，并称之为"经典线上 bug"。

**实测顺序是 `dep:enter → route → background → dep:exit`**——
后台任务跑在收尾**之前**，session 此刻**仍然打开**。原文说反了。

查 FastAPI 官方文档（`advanced/advanced-dependencies.md`，本轮补抓）确认
**这个顺序被反转过两次**：

| 版本 | yield 收尾时机 | 后台任务能否用请求 session |
|---|---|---|
| < 0.106.0 | 响应发送后、后台任务之后 | ✅ |
| 0.106.0 ~ 0.117 | 路由返回后、响应发送前 | ❌ ← **本库原文写的是这一版** |
| **≥ 0.118.0（当前）** | 响应发送后、后台任务之后 | ✅ |
| 0.121.0+ 显式 `Depends(scope="function")` | 路由返回后立即 | ❌ |

已在 [[web/fastapi-di]] 改为完整的版本对照表 + 实测输出，
并修正结论：**不复用请求 session 的真正理由是生命周期解耦（官方原话："后台任务应该是
独立的逻辑单元，有自己的资源"），而不是"技术上已关闭"**。
[[web/fastapi-architecture]] 与 [[interview/question-bank-web]] 的对应表述同步更正。

### 验证通过、未改动的 web/ 层断言

**FastAPI**：anyio 默认线程限流器确为 **40 tokens**；`def` 路由确实跑在
`AnyIO worker thread`、`async def` 跑在事件循环线程；同请求内依赖**只执行一次**且
`use_cache=False` 后变两次；`dependency_overrides` 替换与 `clear()` 恢复；
`response_model` 确实过滤掉未声明字段（password 不出现在响应里）；校验失败返回 **422**；
**裸 `TestClient(app)` 不触发 lifespan，必须 `with TestClient(app)`**。

**Pydantic v2**：7 个 `model_*` 新 API 齐全（v1 旧名仍以废弃别名存在）；
lax 模式 `"42"→42`、**有损 `1.9→int` 报错而 `1.0→int` 通过**、`strict=True` 拒绝转换；
**可变默认值 `list[str] = []` 确实每实例独立**（与 dataclass 相反）；
`alias_generator` + `populate_by_name` + `by_alias` 三向往返；`extra="forbid"`；
`field_validator` 必须配 `@classmethod`；`model_validator(mode="after")` 跨字段；
`discriminator` 判别联合正确分派；`from_attributes` 从 ORM 对象构造；
`model_construct` 跳过校验；`TypeAdapter`。

**SQLAlchemy 2.0**：`Mapped`/`mapped_column` 由注解推导列类型；
**N+1 实测 4 条 SQL、`selectinload` 2 条、`joinedload` 1 条**（与本库表述完全一致）；
`expire_on_commit=True` 时 commit 后访问属性**多发 1 条 SELECT**、`False` 时 0 条；
session 关闭后访问过期属性抛 `DetachedInstanceError`；`lazy="raise"` 抛 `InvalidRequestError`。

### 顺带补测的其它层

`typing`：PEP 695 的 `def f[T]` / `class C[T]` / `type X[T] = ...` 在 3.13 可用；
`@runtime_checkable` 才能 `isinstance`；注解无运行时检查。
`stdlib`：pathlib 路径运算、`fromisoformat` 往返、naive 与 aware 相减 `TypeError`、
`json` 默认转义非 ASCII / int 键转字符串 / `NaN` 输出非法 JSON、`Decimal` 与 `ROUND_HALF_UP`、
`secrets` / `hmac.compare_digest` / `textwrap.dedent`。
`modules-imports`：**循环导入的两条错误信息与本库引用的一字不差**——
`from a import ay` 抛 `ImportError: cannot import name ...`，改成 `import b2` 即可打破；
`python pkg/main.py` 抛 `attempted relative import with no known parent package`，
`python -m pkg.main` 正常。
`coding-patterns`：two_sum / 滑动窗口 / `lower_bound` 与 `bisect_left` 完全一致 /
回溯全排列 / LRU 淘汰顺序 / BFS，六个模板全部跑通。
`metaclasses`：`__init_subclass__` 注册表（含类关键字参数）、`__set_name__` 描述符命名、
装饰器与元类两种单例、**元类冲突报错与合并元类的解法**。
`multiprocessing`：pickle 边界实测——lambda / 生成器 / 文件对象 / 线程锁不可 pickle，
模块级函数与 `functools.partial` 可以。

**结论**：三轮累计约 250 条可测断言，共发现 **15 处错误**（第一轮 6、第二轮 8、第三轮 1）。
`web/` 层这一轮的通过率明显高于 `internals/` 层——**错误集中在"流传多年的教科书结论"
（GC 阈值、竞态示例、pymalloc 对齐）和"框架行为反转过的版本细节"（yield 依赖顺序）两类**，
纯语言语义几乎没错。这两类正是本库需要靠实测持续维护的部分。

## [2026-08-08] query | Python 超大整数用什么类型、会不会像 JS 失真

**问题**：非常大的整数用什么类型存？会像 JS 整数那样精度失真吗？

**依据**：[[bridge/js-to-python]] §⑧ 数字；[[internals/cpython-object-model]]（`int` 任意精度，`sys.getsizeof(10**100)` 随位数增大）。

**结论**：Python 3 只有一种整数类型 `int`（CPython 底层 `PyLongObject`），任意精度，整数运算不会像 JS `Number`（IEEE 754、安全整数 `2^53-1`）那样失真；精度问题出在 `float`，以及 `int` 转 `float` 时。

## [2026-08-09] ingest | functools.update_wrapper 详解

**触发**：连续追问 `@functools.wraps(fn)` 与类装饰器里的 `functools.update_wrapper(self, fn)`，
原页面对后者只有一句注释「类版的 wraps」，跳步。

**改动**：[[language/decorators]] §1 新增子节「`functools.update_wrapper`：`wraps` 底层真正干活的函数」——
`wraps` 是 `update_wrapper` 预填 `wrapped=fn` 的 `partial`、签名与两个常量的作用、
额外设置的 `__wrapped__`、参数顺序与返回值两个易错点、类装饰器为何只能用它
（没有 `def` 可贴装饰器语法）+ `Counter` 示例。同步更新 §3 的注释指向与 `index.md` 摘要。
初稿约 100 行，按「过长」的反馈压到约 30 行，只留主干。

**实测环境**：CPython 3.12.10，本节所有 `#=>` 输出均已跑通。

**同批修正**：本页 §3 `CountCalls.__get__` 补上 `obj is None` 分支——原版从类上访问
（`Svc.m`）会返回 `partial(self.__call__, None)`，把 `None` 当成 self，`Svc.m(inst, x)`
报 `takes 2 positional arguments but 3 were given`。同时补注不加 `__get__` 的报错、
与 `property`/`staticmethod` 同构、以及「一个方法只对应一个装饰器实例 → `count` 全实例共享」。

**遗留**：[[language/functions-arguments]] §1 的
`api(url="/a", timeout=1)` 报错信息与实测不符（函数带 `**opts` 时 `url=` 被吞进 kwargs，
实际报 `missing 1 required positional argument`）。两处待确认后修正。

## [2026-08-09] ingest | 描述符协议的最小可运行例 + 类装饰器共享状态的展开

**触发**：追问 `CountCalls.__get__` 里「描述符协议」是什么、以及「为什么 `count` 是所有实例共享的」。

**改动**：

- [[language/descriptors-properties]] §1 原本只有方法体全是 `...` 的骨架，**没有可运行示例**
  （违反本库硬性要求）。补入 `Positive` 描述符最小完整例（`__set_name__` 自动获知字段名、
  `obj is None` 分支、值存 `obj._radius`），并点明三个关键点：必须挂类上、
  描述符实例本身不存值（否则实例间共享）、`__set_name__` 免去手写字段名。
  末尾接住原有的数据/非数据描述符表格——`Positive` 有 `__set__` 故为数据描述符，
  实测 `c.__dict__["radius"] = -1` 也绕不过校验。
  未与 §6 `Typed` 重复：§1 是最小例，§6 是可复用字段校验的放大版。
- [[language/decorators]] §3 把原先压缩成四行的说明展开：`__get__` 补的是「函数自动绑定 self」
  这一能力（含不加时的 TypeError 实测）、`obj is None` 分支的含义与内置描述符的一致性、
  新增「⚠️ `count` 是所有实例共享的」小节——装饰发生在类体执行时故装饰器实例是**类变量**，
  附归属示意图与 `a.m.func.__self__ is b.m.func.__self__ #=> True` 的证据，
  并指出每实例状态必须落到 `obj.__dict__`（`cached_property` 的做法），
  与 §4 `lru_cache` 装饰实例方法泄漏同根。

**实测环境**：CPython 3.12.10，两处新增代码块的 `#=>` 全部跑通（含 `count #=> 4` 是
顺序执行上一个代码块后的累计值，已在注释中写明）。

**遗留**：[[language/functions-arguments]] §1 的 `api(url="/a", timeout=1)` 报错信息与实测不符
（函数带 `**opts` 时 `url=` 被吞进 kwargs，实际报 `missing 1 required positional argument`）。

## [2026-08-09] ingest | 手写 staticmethod / classmethod / property

**触发**：读到 §4 的 `@staticmethod` 时追问「怎么自己实现一个」。

**改动**：[[language/decorators]] §4 新增子节「自己实现 `staticmethod`/`classmethod`/`property`（高频手撕题）」。
核心论点：**三者都只是描述符，差别全在 `__get__` 返回什么**——
`staticmethod` 原样返回 `fn`（不绑定）、`classmethod` 返回 `MethodType(fn, objtype)`（绑定到类）、
`property` 直接 `fget(obj)`。附三段实现 + 一个把三者用上的 `C`/`Sub` 演示，
以及四行对照表（普通函数 / staticmethod / classmethod / property 各自的 `__get__` 返回值与描述符类型）。

**实测要点**（CPython 3.12.10 全部跑通）：`type(C.util)` 就是 `function`；
`Sub.create(1)` 的 `cls` 是 `Sub` 而非 `C`（classmethod 的真正价值）；
`my_property` 因定义了 `__set__` 是数据描述符，`c.__dict__["w"] = 999` 遮不住它；
`setter` 返回**新对象**，故两个函数必须同名。另注 3.10+ 的真 `staticmethod` 可直接调用，
实现里补了 `__call__` 对齐。

与 [[language/descriptors-properties]] §3「方法为什么能自动绑定」互补：那边讲原理，
这边是「手写复刻」的落地版，两处交叉引用。

**补注**（同日）：§4 该子节的代码里 `objtype` 与 `MethodType` 原为直接使用未加解释，
追问后补两条要点——`objtype` 是「**从哪个类访问到的**」而非「定义在哪个类上」
（实测 `Sub.p` 传入 `Sub`，哪怕描述符定义在 `C` 上，这是 classmethod 配合继承的根因）；
`MethodType(fn, obj)` ≈ JS 的 `fn.bind(obj)`，`type(d.m)` 即 `types.MethodType`，
比 `partial` 多 `__self__`/`__func__`，代价是只能绑第一个位置参数。

**补注 2**（同日）：§4 `lru_cache` 坑列表里「每实例的 `lru_cache`」原先只给了名字没给写法，
现补上完整例子——`__init__` 里 `self.get = lru_cache(...)(self._get)`，包的是**已绑定方法**
故 key 不含 `self`，两实例缓存独立（实测 `cache_info` 各自计数、`g1.get is g2.get` 为 `False`）。
并如实标注代价：这制造了循环引用（实例 → `self.get` → 绑定方法 → 实例），
`del` 后引用计数回收不掉、**`gc.collect()` 之后才回收**（weakref 实测），
与类级 `lru_cache`「`del` + `gc.collect()` 后依然存活」有本质区别。
末尾加三行选型表（无参 → `cached_property`；有参实例少 → 每实例 `lru_cache`；
实例极多 → `cachetools.cachedmethod` 或 `WeakKeyDictionary`）。

**补注 3**（同日）：[[language/decorators]] 页面因多轮追问已增长到约 500 行，
在开头「前端类比」之后加了「记忆点速查（复习先扫这里）」——12 行表格，
每行一个一句话结论 + 指向的小节号，覆盖全页 7 节（脱糖等式、闭包+转发、`wraps` 的作用与后果、
`update_wrapper` 参数顺序、层数规律、类装饰器补 `__get__`、装饰器实例是类变量、
三个内置都是描述符、`lru_cache` 三坑、装饰/调用方向、异步装饰、三者作用范围）。
目的是让长页可先扫结论再按需下钻，与既有的 `> 面试落点` 块形成两级复习入口。

## [2026-08-17] ingest | 复习题组 01：语言核心五题（会话产出落页）

**触发**：用户按 [[interview/roadmap]] 第 1 周 Day 1–4 的四个主题要 5 道复习题，
逐题作答后逐条批改；批改内容有普适价值，落成 [[interview/review-set-01]]。

**新页** `interview/review-set-01.md`：五题各含「题目 / 参考答案 / 解析 / 面试落点」，
覆盖对象模型与拷贝、作用域与闭包、参数全谱与签名内省、装饰器、迭代器生成器与上下文管理器。
**所有输出在 CPython 3.13.3 实测**（版本差异处标注，另在 3.9.6 上验证过 `@contextmanager` 复用的报错类型）。

**本轮实测确认、值得单列的事实**（部分修正了原先页面里说得不够准的地方）：

1. `copy.copy(tuple)` 是 **no-op**——`copy.py` 的 `_copy_immutable` 表含 tuple，
   `copy.copy(t) is t`、`tuple(t) is t`、`t[:] is t` 全部为 `True`；
   `deepcopy` 仅在「每个元素的拷贝都 `is` 原元素」时才复用原 tuple。
   原 [[language/objects-mutability]] 用「浅拷贝共享内层元素」解释这一现象，对 list/dict 成立，对 tuple 偏软。
2. **类作用域推导式的边界条件**：坑的准确表述不是「推导式不能用类变量」，而是
   「**只有最外层 iterable 能用**」——因为它在类作用域求值后作为参数 `.0` 传入隐式函数。
   若同名变量在模块级存在，则全部成立（实测 A/B 两个类对比）。这是 [[language/scope-closure]] 可补的一处。
3. `@contextmanager` **完全忽略生成器的返回值**——`return`／`return False`／`return True`
   行为一致，判据是「生成器有没有 `except` 住」（内部走 `gen.throw()`）。
   与类实现 CM 看 `__exit__` 返回值形成对照，[[language/context-managers]] 未强调此差异。
4. `@contextmanager` 产物复用报 **`AttributeError: '_GeneratorContextManager' object has no attribute 'args'`**
   （3.9 与 3.13 一致），根因是 `__enter__` 末尾 `del self.args, self.kwds, self.func`。
   非 `RuntimeError`——这个报错在线上排查时极易误导。
5. **可重用（reusable）≠ 可重入（reentrant）**：文件对象类实现但不可重用；
   `threading.Lock` 可重用不可重入（实测嵌套 `acquire(timeout=0.2)` 返回 `False`）；`RLock` 两者皆可。
6. 同步 wrapper 装 `async def` **不报错**：coroutine 对象被透传，`asyncio.run` 照常工作，
   计时只覆盖「协程创建」（实测 300ms 协程测出 0.7µs）。`@wraps` 修不了
   `inspect.iscoroutinefunction`（查的是 `CO_COROUTINE` 标志位）；3.12+ 可用 `inspect.markcoroutinefunction`。
7. `def api(a, /, ..., **kw)` 与去掉 `**kw` 的版本，对 `api(a=1)` 报**两句不同的错**：
   前者 `missing 1 required positional argument`，后者
   `got some positional-only arguments passed as keyword arguments`——加 `**kwargs` 会牺牲报错质量。

**交叉引用**：新页加入 [[interview/roadmap]] 第 1 周表格下方作为 Day 1–4 自测入口；
[[wiki/index]] interview 段新增一行，内容页计数 59 → 60。

**与既有页面的分工**：[[interview/traps]] 是单点陷阱速查，本页是**多考点串联的综合题**，
更接近真实面试「一段代码问五个输出」的形态。末尾另立一节「不报错的错」，
把 1-(4)、4-(5)、5-(5) 三处归纳为同一模式：Python 选择静默地做点别的，而不是拒绝执行。

## [2026-08-17] refactor | 复习题组独立成 review 分类

**动机**：复习题组会随 roadmap 进度持续新增（review-set-01、02、03…），
留在 `interview/` 里会把路线图、题库、陷阱页淹没在编号文件中。

**变更**：
- 新建分类目录 `wiki/review/`，`interview/review-set-01.md` → `review/review-set-01.md`（`git mv`，内容未改）。
- `python/CLAUDE.md` Categories 段新增 **review** 分类，明确约定：文件名 `review-set-NN.md` 递增不复用；
  单题四段结构（题目/参考答案/解析/面试落点）；输出实测并标注 CPython 版本；页首表格 + 回链 roadmap 天数。
- `wiki/index.md`：interview 段删去该行，新增独立的「review —— 复习题组」段（带「覆盖范围」列，便于按 roadmap 天数定位）。
- `wiki/interview/roadmap.md` 第 1 周表格下方的自测入口改指 [[review/review-set-01]]。

内容页总数不变（60）。本日志此前条目中的 `interview/review-set-01` 为历史记录，按 append-only 约定保留原样。

## [2026-08-17] ingest | with + 标准库锁/池：__exit__ 语义差异（勘误）

**触发**：读 [[language/context-managers]] §5 的「③ 锁」三行示例时发现其中一行是**错的**，
顺手实测后把三种同步原语的 CM 语义拆开写清楚。

**勘误**：原示例 `with asyncio.Lock(): ...`（注释写着 `# async with`）**跑不通**。
`asyncio.Lock` 只实现 `__aenter__`/`__aexit__`，3.7 起弃用了那个抛 `RuntimeError` 提示改用
`async with` 的 `__enter__`，3.9 起彻底移除，于是现在报的是
`TypeError: 'Lock' object does not support the context manager protocol`——
友好提示退化成了协议缺失错误。同段的 `with threading.Lock():` 是当场 new 一个锁再加解锁，
作为「锁必须共享」的反例更有价值，已改写为先绑定再 `with`。

**新增**（[[language/context-managers]] §5.1，CPython 3.11.9 实测，3.9–3.13 一致）：

1. `threading.Lock.__enter__` 返回 **`True`** 而非锁对象——`with lk as l:` 拿到布尔值，
   后续 `l.release()` 报 `AttributeError`。所以锁一律写 `with lk:` 不带 `as`。
2. `multiprocessing.Pool.__exit__` 源码只有一行 `self.terminate()`，**不是** `close()` + `join()`。
   实测 `with Pool(2) as p: r = p.map_async(slow, range(4))` 后在块外 `r.get(timeout=5)`
   直接 `TimeoutError`——任务随 terminate 被静默杀掉。解法：块内用阻塞式 API（`map`/`starmap`），
   或把 `get()` 写进块内。
3. **反向对照**：`concurrent.futures.Executor.__exit__` 是 `shutdown(wait=True)`，
   离开 `with` 会等所有已提交任务跑完（实测 4 任务 / 2 worker 阻塞 ~2.1s，结果完整）。
   同样形状的 `with` 代码，`Pool` 丢任务、`ProcessPoolExecutor` 不丢。

**归纳（已落成面试落点）**：`with` 只保证「离开时调用 `__exit__`」，
不保证 `__exit__` 做的是「优雅收尾」。用不熟的 CM 之前先看一眼它的 `__exit__` 源码。

**交叉引用**：[[concurrency/multiprocessing]] §5 补 Pool 的 terminate 语义与双向链接；
其陷阱清单 ⑨「忘记 close()/join() 或不用 with → 僵尸进程」原话有误导性（暗示 with 是安全解），
补一行说明 with 同样会杀任务。[[wiki/index]] 两行摘要相应更新。内容页总数不变（60）。

## [2026-08-20] query | 类、MRO 与 ABC 的复习题组（review-set-02）

**请求**：基于 [[language/classes-mro]] 出两道复习题（先不给答案），逐题作答后逐条批改。

**新页** `review/review-set-02.md`：两道多考点串联题 + 批改记录，每行输出均 CPython 3.11.9 实测。

1. **类变量 / 实例变量 / name mangling / 三种方法**（5 小问）
2. **MRO / C3 / 协作式 `**kwargs` / ABC vs Protocol**（A 6 问 + B + C 3 问）

**本次实测新确认的事实**（已写进题组解析）：

- 可变类变量的**子类不隔离**：`Sub` 往 `Base.registry` 里写，
  `'registry' in Sub.__dict__` 是 **False**。要每子类一张表得用 `__init_subclass__`。
- 协作式 `**kwargs` 的末端：无人认领的 kwarg 漂到 MRO 终点，抛
  `TypeError: object.__init__() takes exactly one argument`。这是**特性**（拼写错误能被拓住），
  删掉末端的 `super().__init__(**kw)` 是错的修法。
- `@abstractmethod` 包 `@property`（顺序写反）在 3.11.9 上是**类定义时直接报错**：
  `AttributeError: attribute '__isabstractmethod__' of 'property' objects is not writable`（只读计算属性）。
- `@runtime_checkable` 的 `isinstance` **只查方法名不查签名**：`class Weird: get = 123` 也返回 True。
- 显式继承 Protocol 的子类（`class Empty(Storage): pass`）**能实例化**，
  未实现方法也放过——Protocol 不提供 ABC 那种运行时抽象性保护。
- 时机对比（已归纳为考点）：**C3 失败在类定义时报错，而 ABC 的抽象方法检查在实例化时**。

**出题自省**（已写进页内）：A6 原本让对比 `C().who()`，但 `C.who` 的 `super()`
在 C 实例和 D 实例上都指向 `A`，演示不了「同一行指向不同类」。
能演示的是 `B`：`B().who()` → `'B->A'` vs `D().who()` → `'D->B->C->A'`。已改为 B 版。

**作答错点归类**（两大类，已记入页内批改表）：
① 不逐行读代码而按「应该长什么样」补细节（凭空补了不存在的 `D.init`、把实参当默认值）；
② 知道机制但说错结论（mangling 的目的、可变类变量的危害）。

**交叉引用**：[[wiki/index]] review 节新增一行；[[interview/roadmap]] Day 5 下方加自测入口；
[[language/classes-mro]] 「相关」回链本题组。内容页总数 60 → 61。

## [2026-08-20] ingest | 把 review-set-02 的知识点拆进主题页

**动机**：上一条条目只建了题组页，新实测确认的事实还躺在 [[review/review-set-02]] 里，
按本库 ingest 约定（「把知识点拆进对应的分类页」）补这一步。

**[[language/classes-mro]] 新增 8 处**（行数 292 → 448）：

1. §1 类变量陷阱——补**子类不隔离**：`'registry' in Sub.__dict__` 是 False，
   子类往父类的表里写；给出 `__init_subclass__` 的正确写法；归纳三层代价（串味/共写/泄漏）。
2. §2 绑定方法——补 `c.m is c.m` 是 **False**（每次现造），并加三种方法的 `__get__` 结果对比：
   `Sub.sm is P.sm` True / `Sub.cm.__self__ is Sub`（**不是** `P`，这就是多态构造的根源）/ `p.im.__func__ is P.im`。
3. §4 `super()`——补零参 `super()` == `super(B, self)` 的 `__class__` cell 机制，以及
   **同一行代码不同目标**的铁证：`B().go()` → `B A` vs `D().go()` → `D B C A`。
   （注明用 `C` 演示不了：`C` 的 MRO 是 `(C, A, object)`，两种实例上 `C.go` 的 super 都指向 `A`。）
   反面对照硬编码 `A.go(self)` → `D B A`，危害是后插的 mixin **静默失效**。
4. §5 C3 失败——补**逐步 merge 推演**（三步到双头被拒死锁），并指明报错在 `class` 定义时刻。
5. §5 协作式继承——补**链条末端撞 `object.__init__`**：无人认领的 kwarg 抛
   `TypeError: object.__init__() takes exactly one argument`。强调这是**特性**，
   并标出「删末端 super」的两个代价（静默丢参数 + 换继承结构时断链）。
6. §6 ABC——补**检查时机**：`ABCMeta` 定义期只收集 `__abstractmethods__`，拦截点在
   `object.__new__`；归纳为「从未被实例化的坏子类能合并进主干」，并与 C3 的定义期报错**对比**。
7. §6 装饰器顺序——补原理（`__isabstractmethod__` 标记由 `property` 向上传播），
   以及写反的真实后果：**不是静默丢失抽象性，而是类定义那一刻 `AttributeError`**。
8. §6 ABC vs Protocol 表——加「依赖方向」一行，并补 Protocol 的两个运行时坑。

**[[language/typing]] §4 新增**：`runtime_checkable` 的 `isinstance` 只查方法名（`__len__ = 123` 也过）；
显式继承 Protocol 的空子类能实例化；不加 `runtime_checkable` 用 isinstance 是硬错误而非返回 False。
回链 [[language/classes-mro]] §6。

**验证**：新增的每一段代码写成 20 条断言的脚本跑过，CPython 3.11.9 全部 OK。
（`S3()` 的报错原文是复数 `with abstract methods get, put`——文案随版本/个数变化，页内已标注。）

**内容页总数不变（61）**。

## [2026-08-23] ingest | 复习题组 03（数据模型 dunder 全景）

按 [[language/data-model]] 出了一道综合复习题，落成 [[review/review-set-03]]（对应
[[interview/roadmap]] Day 6）。一个 `Tally` 类 + 11 个输出 + 6 个追问，覆盖六大考点：

1. **str/repr 分工** —— `print(t)` → `1+2`，`print([t])` → `Tally(1, 2)`，
   且 `print(u, t)` 是 str 不是 repr（多参数 print 常被答错）。
2. **`__eq__` / `__hash__` 成对** —— `Tally.__hash__` 被自动置为 `None`；
   追问补了「补 `__hash__` 之后可变对象在 set 里变幽灵」的实测。
3. **`NotImplemented` vs `False`** —— `t == (1, 2)` 走反射链退回身份比较得 `False`；
   反面用 `BadEq`/`Odd` 实测出对称性被破坏（换顺序结论就变）。
4. **容器协议回退边界** —— 本题的 `Tally` 同时有 `__len__` + `__getitem__`，
   所以 `reversed()` **成功**（`[2, 1]`）；与主题页里只有 `__getitem__` 的 `OnlyGet` 报错形成正反对照。
   这个边界原页只写了「失败」一侧，现在两侧都有例子。
5. **`__iadd__` 忘记 `return self`** —— `t += [9]` 静默把 `t` 变成 `None`（对象改成功了，名字丢了），
   并接到 tuple 里放 list 的 `t[0] += [x]`「既改成功又报错」两步字节码解释。
6. **槽位查找 + `__getattr__` 兜底** —— `v.__len__ = lambda: 99` 后 `len(v)` 仍是 1 而 `v.__len__()` 是 99；
   `hasattr` 在无条件 `__getattr__` 下恒 `True`（连不存在的 `__iter__` 都 True）。

**新发现（主题页未覆盖）**：`copy.deepcopy` 在**实例**上探测 `__deepcopy__`，
会被无条件的 `__getattr__` 拦到并把字符串当函数调用 → `TypeError: 'str' object is not callable`；
`copy.copy` 不受影响（copier 在类上取）。修法是 `__getattr__` 里对 dunder 显式
`raise AttributeError`。附带核实：`pickle` 在 **3.11+ 不再中招**（3.11 给 `object` 加了
`__getstate__`，类上就能找到），3.10 及更早才会踩同一个坑——已在页内标注版本边界。

**验证**：题目代码与全部追问代码逐条跑过，CPython **3.11.9** 输出与页内一致
（含 `__new__` 打印插入的行序）。

**同步更新**：[[interview/roadmap]] 加「Day 6 的自测」指针；[[review/review-set-03]] 页尾
回链 data-model / objects-mutability / iterators-generators / descriptors-properties /
cpython-object-model / traps。

**内容页 61 → 62**（index.md 的计数原为 60，与实际不符，本次一并校正为 62）。

## [2026-08-23] lint | review-set-03 批改 + data-model 属性访问层补全

**批改 [[review/review-set-03]]**：实际作答 **7.5 / 11**，页内新增「批改记录」一节。
对的：(1)(2)(3)(4)(7)(10)(11)——`reversed` 回退边界与「dunder 只在类型上查找」都答对了，
说明协议层/槽位层机制是通的。错的三处及错法归类：

- (5) `{t}` 被当成 dict（实为 set 字面量），且不知道 `__eq__` 会把 `__hash__` 清成 `None`。
- (6) `2 in t` 答成 `False`——误以为「没 `__contains__` 就不支持 `in`」（那会是 `TypeError`），
  实际退化成迭代比对元素值。
- (8)(9) 与漏掉的五行 `new (...)` 是**同一个答题习惯**：答「对象里装着什么」而非
  「这一行打印什么」。(9) 因此没看到 `__iadd__` 缺 `return self` 会把 `t` 绑成 `None`。

结论：缺口集中在「`__eq__`/`__hash__` 成对」与「`+=` 是调用 + 重新绑定两步」两个必考点。

**[[language/data-model]] §6 属性访问两轮补全**（起因是读页时追问「`copy.copy` 为什么能成功」）：

1. **`copy` / `deepcopy` 的不对称有了确切根因**——读 `copy.py` 源码确认是同模块内相邻两个函数
   探测位置不一致：`copy()` 是 `getattr(cls, "__copy__")`（**类**上取），
   `deepcopy()` 是 `getattr(x, "__deepcopy__")`（**实例**上取）。`copy()` 还躲过第二关：
   随后的 `getattr(x, "__reduce_ex__")` 虽在实例上取，但 `object.__reduce_ex__` 真实存在，
   常规查找即成功，`__getattr__` 只在查找**失败后**才调，压根没触发。
   反证：给类加 metaclass 级 `__getattr__`，`copy.copy` 立刻同样 `TypeError`。
2. **新增「`__getattr__` 的作用域」小节**——它管的是「定义它的那个类的**实例**」，不是「只管实例」；
   类本身是元类的实例，拦类上的查找要把钩子放进元类。三条实测推论：
   ① 基类的 `__getattr__` 被子类**实例**继承，但 `Sub.missing` 仍 `AttributeError`
   （`Sub` 是 `type` 的实例）；② `__getattr__` 自己是 dunder，挂进实例 `__dict__` 无效；
   ③ **隐式 dunder 调用绕开它**——`G().__len__()` 命中兜底得 42，而 `len(G())` 报
   `TypeError: object of type 'G' has no len()`。第 ③ 条正好解释了 deepcopy 事故的前提：
   `copy.py` 用的是**手写的显式 `getattr`**，走普通属性协议才会被拦。

**验证**：六种作用域情形 + `copy.py` 两处探测 + metaclass 反证逐条跑过，CPython **3.11.9**。

**内容页总数不变（62）**。

## [2026-08-23] ingest | exceptions §7「实战要点」从注释清单扩成分层实战篇

**动机**：原 §7 只有一段 7 条注释的代码块（约 35 行），信息密度低、无法回答
「你们线上怎么做错误处理」这类分层追问。改成 7.1–7.10 十个小节（约 330 行）。

**新增内容**：

1. **7.1 日志**：`log.exception` 只能在 except 块内调用（否则记 `NoneType: None`）；
   `str(e)` 常为空（`str(ValueError()) == ''`）、`str(KeyError("k")) == "'k'"` 是 repr；
   `OSError.errno/strerror/filename` 结构化字段；`traceback.format_exc/format_exception(_only)`。
2. **7.2 兜底屏障**：入口兜底 + `sys.excepthook` + `threading.excepthook`(3.8+) +
   `loop.set_exception_handler` 三层；重点写「Task exception was never retrieved」——
   实测异常直到 Task 被 GC 才打印，时间点错位且进程不失败；给出 TaskGroup 与
   「保引用 + done callback」两种正解；`__del__` 里的异常只打 `Exception ignored in:` 后丢弃。
3. **7.3 重试**：RETRYABLE/FATAL 分类、指数退避 + full jitter（惊群）、
   `CancelledError` 不可重试、`raise ... from e` 保 `__cause__`、tenacity。
4. **7.4 assert**：`-O` 移除整句；补 `assert (cond, "msg")` 元组恒真（3.12 SyntaxWarning）。
5. **7.5 suppress**：语义价值而非性能；裸 `except: pass` 的危害。
6. **7.6 性能与内存**（本次最有价值的新料，均为实测）：
   - 基准表（3.11.9，ns/次）：EAFP 命中 **17** < LBYL 命中 24 ≈ `dict.get` 16；
     EAFP 未命中 76 vs LBYL 未命中 **14**。→ 快乐路径 EAFP 确实更快，异常路径约贵 4~5 倍。
   - `e.__traceback__` → frame → 局部变量的强引用：存异常对象会钉住整个调用栈
     （实测 `saved = None` 后被引用的大对象才析构）。
   - 由此解释 `except X as e:` 块结束时**语言显式 `del e`**，块外访问 `NameError`。
7. **7.7 pickle**：`super().__init__(f"{a}/{b}")` 使 `args` 只有一个元素，
   实测 unpickle 抛 `TypeError: Bad.__init__() missing 1 required positional argument`；
   给出 `__reduce__` 写法；traceback 不可 pickle。
8. **7.8 add_note**：`__notes__` 首次调用才创建（用 `getattr(e, "__notes__", [])`）；
   实测 traceback 里逐行紧跟异常行打印；不改变异常契约，优于包一层 Wrapper。
9. **7.9 异常组拆分**：`eg.split(T)` → (match, rest)、`eg.subgroup(T)`；
   实测 `except* ValueError` 捕获裸 `ValueError("solo")` 时 `as` 变量仍是 `ExceptionGroup`；
   未匹配部分重新打包外抛。
10. **7.10 反模式清单**：10 行表格（反模式 / 后果 / 正确写法）+ 分层回答的面试落点。

**勘误**：§3 原写「异常路径比 if 慢一个数量级」，实测为 76ns vs 14ns 约 4~5 倍，
已改为「约 4~5 倍（见 §7.6）」并回链基准表。

**验证**：所有 `#=>` 输出均在 CPython **3.11.9** 实测（含 pickle 往返、add_note traceback、
split/subgroup、`except*` 包组、`__del__` 吞异常、Task 异常延迟打印、timeit 基准）。

**内容页总数不变（62）**。

## [2026-08-25] ingest | 复习题组 04（异常体系）+ assert SyntaxWarning 勘误

**动机**：Day 6 的另一半（[[language/exceptions]]）此前只有主题页，没有自测卷；
review-set-03 只覆盖了数据模型。本次按同一体例产出 [[review/review-set-04]]。

**新增页**：`wiki/review/review-set-04.md` —— 一道题 **17 个输出 + 7 个追问 + 1 道加分题**，
分七段覆盖：
① 控制流（`finally: return` 吞异常 / try 保护范围过大误捕同类异常 / `else` 的正解）；
② `except X as e` 块末隐式 `del e` —— 在函数里报的是 **`UnboundLocalError`** 而非 `NameError`，
且连 try 之前赋的同名值一起删；
③ 异常链三件套 `__cause__` / `__context__` / `__suppress_context__` 的六种组合取值；
④ `str(ValueError()) == ''`、`str(KeyError("k")) == "'k'"`、以及自定义异常的 pickle 契约
（`super().__init__(拼好的字符串)` → `args` 只剩一个元素 → 重建时 `TypeError`）；
⑤ `add_note` / `__notes__` 首次调用才创建、note 在 `format_exception_only` 里独占一行；
⑥ `except*` 的部分接管语义（`as` 永远是组、未匹配部分重新打包外抛）；
⑦ `except Exception` 罩不住 `BaseException` 一支，但 `finally` 对它一样生效。

**本次新查证的事实**（均 CPython **3.11.9** 实测，主题页此前未记）：

1. `except* ValueError` 与普通 `except` **不能写在同一个 try 上**：
   `SyntaxError: cannot have both 'except' and 'except*' on the same 'try'`。
2. `except*` 块里**不能有 `break` / `continue` / `return`**：
   `SyntaxError: 'break', 'continue' and 'return' cannot appear in an except* block`
   —— 根因是同一个 try 可能执行多个 `except*` 分支，`return` 该听谁的没有定义。
3. `ExceptionGroup("m", [KeyboardInterrupt()])` → `TypeError: Cannot nest BaseExceptions
   in an ExceptionGroup`；而 `BaseExceptionGroup("m", [ValueError()])` **自动降级**返回
   `ExceptionGroup`。设计目的是保证装着 `KeyboardInterrupt` 的组不会被 `except* Exception` 接住。
4. `eg.split(T)` / `eg.subgroup(T)` 在**无匹配时返回 `None`**，不是空组
   （`split` → `(None, 原组)`）。
5. 函数内 `except ... as e` 之后访问 `e` 是 **`UnboundLocalError`**（主题页 §7.6 只写了模块级的
   `NameError`）；且 try **之前**赋给同名变量的值也会被那次隐式 `del` 一并清掉。

**勘误**（[[language/exceptions]] §7.4）：原写「`assert (cond, "msg")` … 3.12 起会给
`SyntaxWarning`」——实测 **3.11.9 已有**该警告（原文
`assertion is always true, perhaps remove parentheses?`），该警告自 3.7 起就存在。已改。

**同步更新**：`wiki/index.md` 加 review-set-04 行；`wiki/interview/roadmap.md` 的 Day 6
自测段补上异常体系那一半的入口；`wiki/language/exceptions.md` 相关链接回链本组题。

**待补**：本组题**尚无批改记录**，等作答后按 review-set-02/03 的体例补 `## 批改记录`。

**内容页总数 62 → 63**。

## [2026-08-25] lint | review-set-04 批改（12/17）+ exceptions 异常链与 as 变量补全

**批改结果**：[[review/review-set-04]] 实际作答 **12 / 17**（全对 10、半对 4、全错 3），
已补 `## 批改记录` 段。

**两块知识性缺口**（已回补进 [[language/exceptions]]）：

1. **异常链三件套整体反转**（(5)(6) 两问全错）——作答认为「不写 `from` → `__cause__` 有值、
   `__context__` 为 None」，实际正好相反，且 `__suppress_context__` 在两问里都答反。
   §4 新增**三种写法 × 三个字段的完整取值表**（3.11.9 实测）+ 三条结论：
   ① `__context__` 解释器自动记录，三种写法下**都有**；
   ② **`from None` 不擦除 `__context__`**，只置 `__suppress_context__=True` 让 traceback 不打印，
   仍可手动取回；③ `__suppress_context__` 回答的是「写没写过 `from`」，所以 `from None` 也是 `True`。
   配一条面试落点：「context 自动、总在；cause 手动、要写 from；suppress 只标记写过 from 没有」。
2. **`except ... as e` 的隐式 `del` 是盲区**（(4) 全错）——§7.6 原文只给了模块级的
   `NameError`。新增编译器插入的等价代码（`finally: e = None; del e`），并实测补充：
   **函数内报 `UnboundLocalError`、模块级才报 `NameError`**；且**连 try 之前赋给同名变量的值
   也会被一并删掉**——由此得出「别拿外层已有的变量名做 `as` 目标」这条实操规则。

**答题习惯问题（非知识缺口）**：(1)(7)(8)(11) 四条半对同属「答语义、不答字面输出」——
机制都对，丢分在没把 stdout 逐字符写出来（(1) 只说「没有抛出」不写 `1`；(7) 漏 `KeyError`
的内层引号；(11) 只写两条消息大意，漏 list[str] 与 `
`）。这与 [[review/review-set-03]]
批改记录里记下的是**同一个习惯的复发**，已在本次批改中显式点出并交叉引用。
另记：`e.args` 与 `eg.exceptions` **都是 tuple**（作答四处写成 list，本次未扣分）。

**掌握扎实**：(9) 跨进程 pickle 契约、(14) `except*` 的部分接管、(15)(16)(17) `BaseException`
一支的边界——本题最难的三问全对。

**内容页总数不变（63）**。

## [2026-08-25] refactor | review 题组的题型限定：只出可机械判分的题

**决定**：从今天起，`wiki/review/` 的「题目」段只出三类题——**写输出 / 判断题 / 选择题**，
不再出开放式简答与追问题（「为什么这么设计」「说说三者关系」「给出两种修法」这类）。
理由：作答耗时太长，同样时间多做几道输出题收益更高。

**关键约束：只限制题型，不削减内容。** 原本放在「追问」里的机制解释、设计动机、修法、
事故案例，一律移进「解析」段由我直接写清楚，读者读而不答；确实需要考查机制理解时，
把它**改写成输出题或判断题**（例：不问「`from None` 时三个字段各是什么」，
改为给一段打印 `__cause__` / `__context__` / `__suppress_context__` 的代码让人写输出）。

**落点**：已写进 `python/CLAUDE.md` 的 review 分类规范。
[[review/review-set-04]] 及更早的题组保留原有的「追问」段，不回改。

**内容页总数不变（63）**。

## [2026-08-26] ingest | PyObject / PyVarObject / PyTypeObject 三者关系

[[internals/cpython-object-model]] 原先只贴了 `PyObject`、`PyVarObject` 的 C 结构体定义，
没讲清三者关系，读者容易误以为它们是同一维度上的三层继承。新增第 2 节讲**两条正交的关系**：

1. **内存布局的嵌套**（C 模拟单继承）：每层把上一层原封不动放在内存最开头，故可逐级向上强转。
   `PyVarObject` 只是**变长对象的可选分支**，不是必经之路（`float`/`dict`/`set` 直接用 `PyObject`）。
   关键结论：`PyTypeObject` 自己也以 `PyObject_VAR_HEAD` 开头，因而它本身就是 `PyObject`——
   这是「类也是对象」在 C 层的落地。
2. **实例 → 类型的指向**：`ob_type` 指向 `PyTypeObject`，链条终点 `PyType_Type` 自指，
   对应 `type(type) is type`。

另记两个易错点：`ob_size` 语义随类型而变（`int` 是 30-bit 数字位个数、用负号表示负数）；
`PyTypeObject` 的 `ob_size` 静态类型恒为 0、堆类型是内部用途，不要读。
原第 2~7 节顺延为第 3~8 节。

**内容页总数不变（63）**。

## [2026-08-26] lint | 修正「共享键字典 = 3.11 / PEP 412」的版本错误

[[internals/cpython-object-model]] 第 6 节原文把**两代不同的优化压成了一句**，并给 3.3 的 PEP 号
配了 3.11 的版本号。已拆开改正：

- **3.3**：共享键字典（split table，**PEP 412**），键表挂在 heap type 的 `ht_cached_keys` 上。
- **3.11**：**内联值数组 + `__dict__` 惰性创建**（faster-cpython，不是 PEP 412）；
  3.12 起值数组真正内联进对象分配。

同时补上原文缺的三件事：
1. **量化「优势被削弱」**：104 字节 vs `__slots__` 64 字节（3 属性，3.11.9/64 位，
   `tracemalloc` 20 万实例人均），从早年好几倍缩到 1.6 倍。
2. **「更快」要加限定**：3.11 的 `LOAD_ATTR_INSTANCE_VALUE` 让普通实例读属性也接近固定偏移，
   `__slots__` 的速度优势同样缩水——原文只说「依然更省更快」，会给人错误的量级印象。
3. **这是机会性优化，两个失效场景**（真实代码里经常拿不到）：碰一次 `o.__dict__` 触发物化
   104→168 字节；实例键序分歧导致共享解除 88→342 字节。触发物化的常见操作：`vars()`、
   默认 `pickle`/`copy`、ORM 与序列化库、调试器。

[[internals/memory-model]] 实测表下的同一处版本错误（「3.11+ 共享键字典」）一并改为
「共享键字典 3.3 + 内联值 3.11 两轮优化」。

**内容页总数不变（63）**。


## [2026-08-27] lint | 修正「3.14 = 增量式 GC」——3.14.5 已回滚

[[internals/garbage-collection]] 原先在阈值表和答题模板里写「**3.14 换成增量式 GC**，
阈值 `(2000, 10, 0)`，第三个阈值不再使用」。**这条已经过期。**

为核实，从 GitHub 补抓三份一手材料进 `raw/cpython-doc/`（见 [[sources/cpython-docs]]）：
`InternalDocs/garbage_collector.md` 的 3.14 分支版与 `v3.14.0` tag 版、以及 `Doc/whatsnew/3.14.rst`。
官方原话：

> **From Python 3.14.5 onwards:** Python 3.14.0-3.14.4 shipped with a new incremental GC.
> However, due to a number of reports of **significant memory pressure in production
> environments**, it has been **reverted back to the generational GC from 3.13**.

（回归 issue [gh-142516](https://github.com/python/cpython/issues/142516)）

**用 uv 装 3.14.6 实测确认**：`get_threshold()` → `(2000, 10, 10)`、`get_stats()` 三代。
和 3.13 完全一致，增量式的痕迹已经没有了。

已改成完整的版本演进表（≤3.12 / 3.13 / **3.14.0–3.14.4** / **3.14.5+** 四行），
并把增量式的设计单独留档在新的 §3.1（两代 + `pending`/`visited` 双链表 + increment 的传递闭包 +
`threshold1` 语义反转 + `gc.collect(1)` 变成"做一个 increment"），标注**已回滚**。
答题模板同步改成「3.14 试过增量式，因生产环境内存压力在 3.14.5 回滚了」。

> **教训**：涉及 patch 版本的行为变更，只记 minor 版本号是不够的。本库以后写
> 「3.x 起改为 Y」时，如果该变更在 patch 版本里被动过，必须写到 patch 级。

## [2026-08-27] ingest | GC 深挖：减法算法、浮动垃圾、全局环 + 新页 internals/weakref

起因是三轮追问，每一问都暴露出原页面把结论压得太狠、缺推导和反例。全部结论在
**3.11.9 + 3.14.6 实测**，机制描述以补抓的 `InternalDocs/garbage_collector.md` 为准。

**[[internals/garbage-collection]] 的六处补全**：

1. **§2.1 / §2.2 减法到底在算什么**（原先只有 4 行伪代码）。补恒等式
   `ob_refcnt = 集合内引用 + 集合外引用`，第 2 步是把第一项精确减掉，所以剩下的 `gc_refs`
   **恰好等于外部引用数**。再用两个例子说明**为什么必须有第 3 步**：纯垃圾环减法就够了；
   但 `keep → x ⇄ y` 里 x/y 的 `gc_refs` 会被误减到 0，只能靠从存活根做可达性传播来赦免。
   一句话：**减法只能证明"被外面直接引用"，间接引用要靠第 3 步**。
2. **§3.1 版本演进表 + 增量式设计留档**（见上一条 lint）。
3. **§3.2 什么对象在哪一代**：原子对象与不朽对象**不在任何代**；长寿容器赖在 gen2 被反复空扫；
   `gc.freeze()` 后进**永久代**（第 4 个链表）。
4. **§3.3 跨代环**：老对象引用新对象时，收年轻代把老代引用算作外部引用 → 年轻成员被误判存活
   **并被晋升**，只能等扫到最老那代。方向容易搞反——**没有降级，是年轻的那个被晋升上去等**。
   实测 `collect(0)/(1)` 各 0 个、`collect(2)` 才回收。这是 tracing GC 的 old→young 指针问题，
   别的语言用 write barrier + remembered set，CPython 没有。
5. **§3.4 25% 硬门槛**（原页面完全没提）：`long_lived_pending / long_lived_total > 25%` 才做全量回收。
   结论是**老对象越多，全量回收越贵，所以做得越少** → §3.3 的等待时间**没有上界**。
6. **§3.5 `gc.freeze()` 与永久代**：实测 freeze 把三代搬空（`[182,4821,0]` → `[2,0,0]`，
   `freeze_count=5001`），**unfreeze 是落回 gen2 而不是 gen0**。用途是 fork 前调用保住 CoW。

**新增 §4「环一定能被回收吗？三个条件」** —— 这是本轮最重要的产出。
原页面暗示"环在被扫的那一代内就能收"，**这是错的**。三个条件缺一不可，配三个反例：

- **§4.1 浮动垃圾**：环**整体在 gen0**，但被一个**自己也是垃圾**的 gen2 对象引用 →
  `collect(0)`/`collect(1)` 都是 0，只有 `collect(2)` 回收 3 个。
  根因：**减法只区分"集合内/外"，完全不区分"活/死"**。
- **§4.2 C 扩展 GC 协议不完整**：缺 `Py_TPFLAGS_HAVE_GC` → 环对 GC 不可见（永久泄漏）；
  `tp_traverse` 漏字段 → 误判存活根；报告多余引用 → **过度减计数 → 段错误**。
- **§4.3 全局对象之间的环**：运行期**永远收不掉**（`gc.collect()` → 0），因为
  `module.__dict__` ← module ← `sys.modules` 是集合外部引用。但关键认知是
  **GC 眼里没有"全局对象"这个类别**，只有"被 `module.__dict__` 引用"——`del` 一执行立刻可收。

**§5 `__del__` 补两层语义**：

- **销毁不可达对象的五步**（`InternalDocs` 原文）：清弱引用 → legacy finalizer 进 `gc.garbage` →
  调 `tp_finalize` 并**打上 finalized 标记** → 重跑环检测处理复活对象 → `tp_clear`。
- **复活只生效一轮**：实测 `__del__` 里 `saved.append(self)` 让 `collect()` 返回 0，
  清空 `saved` 后再 `collect()` 回收 2 个、**`__del__` 不再被调用**（对应 finalized 标记）。
- **退出时**：实测全局环会被收，且 `__del__` 里模块全局**仍然可访问**（3.4 起不再置 `None`，
  老 Python 那个"看到一堆 None"的坑已作废）；但文档明确不保证，daemon 线程被强杀、
  `os._exit()`、C 静态持有仍会失效。

**新页 [[internals/weakref]]**（原 GC 页只有半节用法，展开成独立页）：

- **`tp_weaklistoffset`**：类型必须预留槽位，`C.__weakrefoffset__` 可见（普通类 16，`list` 是 0）。
  实测**`list`/`dict`/`tuple`/`int`/`str`/`object` 都不能被弱引用，`set` 可以** —— 这条最常记错。
  `__slots__` 要显式加 `'__weakref__'`。
- **生命周期四步**：存裸指针不 `INCREF`（实测 refcnt 前后都是 2）→ refcnt 归零 → `tp_dealloc`
  首先调 `PyObject_ClearWeakRefs` 把 `wr_object` 改写成 `Py_None` → 调 callback，
  **参数是弱引用自己，原对象已经是 `None`**。
- **三个坑**：无 callback 的弱引用**被缓存复用**（`weakref.ref(c) is weakref.ref(c)` → True）；
  不保存返回值 → 弱引用自己先死 → **callback 永不触发**（正解 `weakref.finalize`）；
  **循环 GC 里 callback 只在"弱引用自身可达"时才调用**（`InternalDocs` 原文），
  弱引用和对象一起进垃圾环则直接丢弃。
- 定位：弱引用不是"让 GC 更快"，而是**让对象根本不进 GC**。

**顺带补的三处**：

- [[internals/cpython-object-model]] §5 新增「**不朽对象**」小节。实测 3.11.9 的
  `getrefcount(0)=1000000067`（基数 `999999999`，源头 `pycore_object.h` 的
  `_PyObject_IMMORTAL_INIT`），**3.11 里 `None` 还是普通计数**；3.12 PEP 683 才把
  `None`/`True`/`False` 也变不朽；3.14.6 魔数是 `0xC0000000`，新增 `sys._is_immortal()`。
  **结论：别背魔数，用 `sys._is_immortal()`。**
- [[internals/memory-model]] 新增 §8「**fork 与 copy-on-write**」（原先完全没有，
  而 GC 页要引用它）。两个把 CoW 页写脏的元凶：引用计数改对象头、**循环 GC 就地修改 `gc_refs`**。
  后者可用 fork 前 `gc.freeze()` 彻底消掉（Instagram 那套）；前者由 3.12+ 不朽对象部分缓解。
  加了「只对 `fork` 有效，Windows / `spawn` 不适用」的限定。
- [[interview/question-bank-internals-concurrency]] Q6 的版本注意同步改正（原文也写着「3.14 换成增量式」）；
  **Q7「循环引用一定会泄漏吗」整段重写**——原答案「循环引用会被循环 GC 回收，不会永久泄漏」
  太强，改成三个条件 + 浮动垃圾/全局环/C 扩展三个反例，并补上「`__del__` 一辈子只调一次、
  复活只能推迟一轮」。

**内容页 63 → 64**（新增 [[internals/weakref]]）。`raw/` 新增 3 个文件。

## [2026-08-30] lint | 重写 memory-model §1「三层分配器」

**问题**：原 §1 的 ASCII 图把 Layer 0–3 画成四行却标题叫「三层」，
又把 `arena → pool → block` 塞进 Layer 2 那一行，导致**两个互相垂直的「三」被混成一个**，
读者分不清哪个是调用链、哪个是 pymalloc 的内部结构。

**改动**：

- §1 拆成 **1.1 纵向：一次分配的调用链**（新流程图 + 层次表 + `x = 3.5` 的四步走位，
  并点明 > 512 字节直接跳过 pymalloc 走 `malloc`——大小对象两条路）
  与 **1.2 横向：pymalloc 内部 arena → pool → block**（公寓楼类比，
  「一个 pool 只服务一种大小类」这条规矩同时解释了 O(1) 的快和内部碎片的代价）。
  开头先明确声明两个「三」的区别，并注明源码出处 `Objects/obmalloc.c`。
- §2 补上**对称的反向释放路径**（freelist → pool → arena → `munmap`），
  「三个原因」改写成「三个截留点，对应上面三层」，与 §1 的分层一一对齐；
  补充「绕过 pymalloc（`PYTHONMALLOC=malloc`）也一样不降」。
- §2 工程对策加了一句定性：没有一条是「手动释放」，只有「换进程」或「不制造峰值」两条路。

**未改**：§3 及之后的编号全部保持不变（`garbage-collection.md:612` 引用了本页 §8）。

## [2026-09-06] query | 出题：collections/itertools/functools/heapq/bisect 自测卷

用户读完 [[stdlib/collections-itertools]] 后要求「基本覆盖文档内容」的题目，并要能自己填写、由 LLM 批改。

**产出**：[[review/review-set-05]] —— 50 题（选择 20 / 判断 15 / 填空 15），每题下留 `**答：**` 空位，
答案与解析暂不写入，等作答后追加到页末的「参考答案与批改」一节。

**命题覆盖**（对照主题页六节）：

- **collections**：`__missing__` 的写入副作用、`Counter` 减法丢负数 vs `subtract` 保留负数、
  `most_common(n)` 走 `nlargest`、`maxlen` 挤出方向、`extendleft` 逆序、`rotate` 方向、
  deque 与 list 的复杂度镜像、namedtuple 的 tuple 相等语义、ChainMap 写入只落第一层且不拷贝、
  OrderedDict 三处独有能力。
- **itertools**：`islice` 三参切片与不支持负索引、`chain.from_iterable` 只展平一层、
  `tee` 之后不能再动源迭代器、`dropwhile` 与 `filter` 的区别、`accumulate(max)`、
  `pairwise`(3.10+) / `batched`(3.12+)、`groupby` 不排序 + group 惰性失效、`product` 规模 n^r。
- **functools**：`@cache` = `lru_cache(maxsize=None)`、装饰实例方法的内存泄漏、
  底层 dict + 双向循环链表、`cached_property` 与 `__slots__` 不兼容、`partial` 可 pickle、
  `wraps` / `__wrapped__`、`singledispatchmethod` 按第二个参数分派。
- **heapq / bisect**：堆序 ≠ 有序、`heapreplace` vs `heappushpop` 的空堆行为、
  取负实现最大堆、`itertools.count` 打破平局、`bisect_right`、出现次数 = right − left、
  `insort` 整体 O(n)。

**未覆盖**（刻意留白，属于「读一遍知道就行」的条目）：`total_ordering` 的性能代价、
`cmp_to_key` 的适用边界、`heapq.merge` 的外部排序场景、`bisect` 的 3.10+ `key` 参数。
批改时若前 50 题正确率高，再补一组追问。

## [2026-09-07] query | 批改：复习题组 05 填空题

**成绩**：选择 17/20 · 判断 15/15 · 填空 8.25/15（合计 40.25/50）。批改记录已追加到
[[review/review-set-05]] 的「参考答案与批改」一节。

**失分画像**：判断题满分、选择题里 deque/groupby/tee/islice/partial/singledispatchmethod/
heapreplace 全对 —— **机制性理解扎实**；失分集中在「具体名字、版本号、复杂度」这类纯记忆点，
填空题有 4 道直接空着（第 1、9、12、14）。

**回炉五处**（已写进批改页）：

1. `__missing__` —— defaultdict 的唯一机制，也是「读一下就插入」副作用的原因；它是 dict 的钩子而非 defaultdict 私有。
2. `chain.from_iterable` —— 只写 `from_iterable` 不成立，它是挂在 `chain` 上的类方法。
3. **版本号表**（新增到批改页）：`accumulate(initial=)` 3.8+、`cache` 3.9+、`Counter.total()` / `pairwise` / `bisect(key=)` 3.10+、`batched` 3.12+。
4. `lru_cache` = dict + 双向循环链表（**不是 OrderedDict**）—— 用户答 OrderedDict，错得有价值：
   OrderedDict 本身就是 dict+双向链表，但 `lru_cache` 是 C 实现自维护链表。与「手写 LRU 用 OrderedDict」
   和「装饰实例方法泄漏」串成一条链。
5. heapq 的**两个不同问题**被混为一谈：取负实现最大堆 `(-priority, item)` vs `itertools.count()`
   序号打破平局（顺带 FIFO）；完整写法 `(-priority, next(counter), task)`。

**选择题 3 错**：#2（Counter 减法丢弃 ≤0，判断题第 3 题已判对却没迁移到输出题）、
#15（`cache` = `maxsize=None` 而非 128）、#20（`bisect_right` 返回 4，答成了列表长度 5）。

**下一步**：主题页 [[stdlib/collections-itertools]] 本身无需修改（题目全部有出处）。
若要再练，可就 log 上一条留白的四个条目（`total_ordering` 性能代价、`cmp_to_key` 适用边界、
`heapq.merge` 外部排序、`bisect` 的 3.10+ `key`）出一组追问。

## [2026-09-07] lint | 补齐复习题组 05 第 9 题的详细解答

**问题**：批改页的「需要重点回炉」只展开了 5 处，第 9 题（`defaultdict(list)` vs `groupby`）
虽在逐题表里判了 ❌，却没有解析——而它恰恰是四道空题里**唯一考选型判断而非记忆**的一题。

**改动**：新增 ④「第 9 题 · `defaultdict(list)` vs `groupby`」（原 ④⑤ 顺延为 ⑤⑥，
标题改为「六处」），内容为：不排序时 groupby 把两个 `eng` 拆成两组的反例、
必须先 `sorted` 的 O(n log n) 代价、`defaultdict(list)` 的 O(n) 线性写法、
四行选型表（不关心顺序 / 数据天然有序 / 游程编码 / 需要分组有序）、以及 RLE 这个
groupby 的主场例子。全部输出在 CPython 3.11.9 实测。

**顺带**：「回主题页重读」第 5 条从「§6 速查表第二行」改指 **§2 的 groupby 一节**
（那里才有「什么时候 groupby 才是对的选择」的完整论述），速查表降为一句话版。
