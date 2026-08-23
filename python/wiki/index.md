# Index

Python 知识库内容目录。共 **62 个内容页**（另有 home/index/log 三个导航页），按分类组织。★ 标记的是面试高频/必读页。

## interview —— 面试题库与路线图

| 页面 | 一句话 |
|---|---|
| ★ [[interview/roadmap]] | 六周冲刺计划、12 道必背题、转行叙事、答题框架 |
| ★ [[interview/question-bank-language]] | 语言核心 28 题 + 现场可用的参考答案 |
| ★ [[interview/question-bank-internals-concurrency]] | GIL/GC/内存/并发/asyncio 25 题 + 答案 |
| ★ [[interview/question-bank-web]] | FastAPI/数据库/架构/部署/安全/工程 24 题 + 答案 |
| ★ [[interview/traps]] | 22 个经典陷阱题（全部在 3.13 实测）+ 四段式答题法 |
| [[interview/coding-patterns]] | 手撕代码模板与 Pythonic 惯用法 |

## review —— 复习题组（自测卷 + 批改）

按 [[interview/roadmap]] 的进度逐组产出，编号递增。每组 = 多考点串联的综合题 +「题目 / 参考答案 / 解析 / 面试落点」四段解答，输出全部实测。

| 页面 | 覆盖范围 | 一句话 |
|---|---|---|
| ★ [[review/review-set-01]] | Day 1–4 · 语言核心 | 对象模型/作用域/参数/装饰器/生成器五题综合体检，全部 3.13 实测 |
| ★ [[review/review-set-02]] | Day 5 · 类与 MRO | 类变量/name mangling/C3/协作式 super/ABC vs Protocol 两题，含批改记录 |
| ★ [[review/review-set-03]] | Day 6 · 数据模型 | 一个类考 11 个输出：str/repr、eq/hash、协议回退、`__iadd__`、槽位查找、`__getattr__` 兜底，3.11.9 实测，含批改记录 |

## language —— 语言核心机制

| 页面 | 一句话 |
|---|---|
| ★ [[language/data-model]] | dunder 全景；语法如何翻译成协议调用 |
| ★ [[language/objects-mutability]] | 名字绑定、可变性、拷贝、参数传递、`is` vs `==` |
| ★ [[language/scope-closure]] | LEGB、闭包 cell、延迟绑定、无块级作用域 |
| [[language/functions-arguments]] | 五类参数、解包、默认值陷阱、签名内省 |
| ★ [[language/decorators]] | 装饰器全谱、`wraps` / `update_wrapper`、手写 staticmethod/classmethod/property、顺序、异步装饰器 |
| ★ [[language/iterators-generators]] | 迭代器/生成器/`yield from`、惰性求值、异步生成器 |
| [[language/context-managers]] | `with` 协议、`@contextmanager`、ExitStack、标准库锁/池的 `__exit__` 语义差异 |
| ★ [[language/classes-mro]] | 类、C3 线性化（含 merge 推演）、`super()` 的真实含义、协作式 `**kwargs`、ABC vs Protocol |
| ★ [[language/descriptors-properties]] | 描述符协议（含最小可运行例）、属性查找顺序、property、`__slots__` |
| [[language/metaclasses]] | 类的创建流程、`__new__`、以及为什么不该用元类 |
| [[language/typing]] | 类型注解、泛型、Protocol、mypy（与 TS 对照） |
| [[language/exceptions]] | 异常层级、EAFP、异常链、ExceptionGroup |
| [[language/dataclasses-models]] | dict/TypedDict/NamedTuple/dataclass/Pydantic 选型 |
| [[language/comprehensions-functional]] | 推导式、walrus、排序、`match`、JS 数组方法对照 |
| [[language/strings-encoding]] | str vs bytes、编码、f-string、切片、正则 |
| [[language/modules-imports]] | 导入系统、搜索路径、循环导入、src layout |

## internals —— CPython 实现原理

| 页面 | 一句话 |
|---|---|
| [[internals/cpython-object-model]] | PyObject、类型 slot、引用计数、对象缓存 |
| ★ [[internals/garbage-collection]] | 引用计数 + 标记清除 + 分代阈值（含 3.13/3.14 变化）、泄漏排查 |
| ★ [[internals/gil]] | GIL 为什么存在、何时释放、PEP 703 free-threading |
| [[internals/memory-model]] | pymalloc/arena/pool、为什么 RSS 不降、省内存手段 |
| [[internals/bytecode-execution]] | 编译流水线、`dis`、帧、`.pyc`、3.11+ 特化解释器 |

## concurrency —— 并发与异步

| 页面 | 一句话 |
|---|---|
| ★ [[concurrency/concurrency-models]] | 三模型对比与选型决策树、`concurrent.futures` |
| [[concurrency/threading]] | 线程、锁、死锁四条件、Queue、原子性误区 |
| [[concurrency/multiprocessing]] | 进程、fork vs spawn、pickle 限制、IPC 手段、Pool 的 terminate 语义 |
| ★ [[concurrency/asyncio-fundamentals]] | 事件循环、协程惰性、`await` 机制、阻塞事故 |
| ★ [[concurrency/asyncio-patterns]] | TaskGroup、超时、取消、限流、contextvars、生态 |
| [[concurrency/io-multiplexing]] | select/poll/epoll、C10K、asyncio 的物理底座 |

## web —— FastAPI 后端栈

| 页面 | 一句话 |
|---|---|
| [[web/wsgi-asgi]] | WSGI/ASGI 协议、gunicorn vs uvicorn、中间件模型 |
| ★ [[web/fastapi-core]] | 路由、参数来源、`Annotated`、响应模型、`async def` vs `def` |
| ★ [[web/fastapi-di]] | 依赖注入、缓存、`yield` 依赖、`dependency_overrides` |
| ★ [[web/pydantic]] | Pydantic v2、v1→v2 对照、校验器、settings、性能 |
| ★ [[web/fastapi-async-db]] | SQLAlchemy 2.0 async、session 作用域、**N+1**、事务、Alembic |
| [[web/fastapi-auth]] | 密码哈希、JWT、OAuth2、RBAC、安全清单 |
| ★ [[web/fastapi-architecture]] | 分层与目录结构、lifespan、中间件、异常映射、后台任务 |
| [[web/fastapi-testing]] | TestClient、依赖覆盖、DB 测试策略、conftest 全景 |
| ★ [[web/fastapi-production]] | worker 模型、Docker、优雅关闭、可观测性、调优、上线清单 |

## stdlib —— 数据结构与标准库

| 页面 | 一句话 |
|---|---|
| ★ [[stdlib/builtin-data-structures]] | **复杂度表**、list/dict/set 底层、哈希契约 |
| [[stdlib/collections-itertools]] | Counter/defaultdict/deque/heapq/bisect/itertools/functools |
| [[stdlib/stdlib-essentials]] | pathlib/datetime/json/logging/subprocess + Node 对照 |

## engineering —— 工程实践

| 页面 | 一句话 |
|---|---|
| [[engineering/packaging-envs]] | uv、pyproject.toml、依赖锁、wheel/sdist |
| [[engineering/testing]] | pytest、fixture、参数化、mock、覆盖率、hypothesis |
| [[engineering/tooling-quality]] | ruff、mypy、pre-commit、CI、PEP 8 |
| [[engineering/performance]] | 剖析工具、优化优先级、缓存三问题、换库、编译加速 |

## bridge —— 前端视角对照

| 页面 | 一句话 |
|---|---|
| ★ [[bridge/js-to-python]] | 语法对照表 + **10 个"看似相同实则不同"的危险区** |
| ★ [[bridge/async-js-vs-python]] | 两种事件循环的 5 个核心差异 + API 对照 |
| [[bridge/npm-vs-pip]] | 工具链、Web 框架、常用库的完整生态对照 |

## analysis —— 综合分析

| 页面 | 一句话 |
|---|---|
| ★ [[analysis/frontend-to-python-gap]] | 四类知识差距分析 + 时间分配建议 |

## sources —— 源材料导读

| 页面 | 一句话 |
|---|---|
| [[sources/cpython-docs]] | CPython 官方 datamodel/descriptor/functional/FAQ + PEP 8/492/703 |
| [[sources/fastapi-docs]] | FastAPI 全套教程 + Pydantic v2 文档，及本库的有意差异 |
| [[sources/interview-python-cn]] | 中文经典面试题库 + **过时内容标注表** |
| [[sources/wtfpython]] | 陷阱集导读 |
| [[sources/python-cheatsheet]] | 语法速查表导读 |
| [[sources/python-patterns]] | 设计模式集导读 + "Python 让哪些模式不必要" |

## 快速入口

- **今天就要面试** → [[interview/roadmap]] 的"12 道必背题" + [[interview/traps]]
- **不知道从哪开始** → [[analysis/frontend-to-python-gap]] → [[bridge/js-to-python]]
- **要写 FastAPI 项目** → [[web/fastapi-architecture]] → [[web/fastapi-core]] → [[web/fastapi-async-db]]
- **被 asyncio 卡住** → [[bridge/async-js-vs-python]] → [[concurrency/asyncio-fundamentals]]
