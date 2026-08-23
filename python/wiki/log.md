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
