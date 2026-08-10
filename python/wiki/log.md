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
