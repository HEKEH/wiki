# Python Knowledge Base

面向 **「前端工程师转 Python + 冲刺高难度 Python 后端面试」** 的知识库。
**59 个内容页 + 3 个导航页**，全部带可运行代码示例，所有关键结论在 **CPython 3.13** 上实测过。

**技术方向已确定为后端 Web，FastAPI 是一等公民。**

导航见 [[index]]，活动记录见 [[log]]。

## 从哪开始

```text
第 0 步：搞清差距  → analysis/frontend-to-python-gap   （四类知识、时间怎么分）
第 1 步：修正直觉  → bridge/js-to-python               （10 个"看似相同实则不同"）
第 2 步：按周执行  → interview/roadmap                 （六周冲刺计划）
第 3 步：查漏补缺  → interview/traps + 三个 question-bank
```

## 核心洞见

### 1. 你的优势比你以为的大

**约 40% 的知识可以直接迁移**：HTTP/REST/鉴权、事件循环心智、TypeScript 类型思维、
包管理与锁文件、lint/CI、测试与 mock、Docker 与 12-factor。
Python 生态的 asyncio ≈ JS 事件循环，Pydantic ≈ zod，uv ≈ pnpm，ruff ≈ eslint+prettier，
mypy ≈ tsc——**这些是"换写法"而不是"重新学"**。

真正需要从零建立的只有三块：**CPython 运行时（GIL/GC/内存）、多线程多进程、数据库**。
见 [[analysis/frontend-to-python-gap]]。

### 2. 面试的分水岭不在语法，在 CPython 运行时

语法人人会背。**GIL、垃圾回收、内存模型这三题决定档位**，而且它们不是三个独立知识点，
而是**一套互相牵制的设计**：

> CPython 用**引用计数**做主回收 → 多线程改计数需要保护 → 引入 **GIL**（一把大锁，
> 换单线程性能和 C 扩展的简单性）→ 引用计数解决不了循环引用 → 再加**分代标记清除 GC**。
> 三者一体。

能把这条因果链讲出来，比背十条 API 有用。见 [[internals/gil]]、[[internals/garbage-collection]]。

而**再深一层的分水岭是知道 GC 的保证有多弱**。三条都是"背过 vs 真懂"的区分点：

> ① 循环 GC 的减法只区分"**在不在本次扫描的集合内**"，**完全不区分对象活着还是死了**——
> 所以一个已经是垃圾的老对象，照样能把年轻代里的整个垃圾环担保成"存活"（浮动垃圾）。
> ② 全量回收还有一道 **25% 硬门槛**，老对象越多做得越少，跨代环的等待时间**没有上界**。
> ③ 因此 `weakref` 的价值不是"让 GC 更快回收"，而是**打断环、让对象根本不进 GC**。

见 [[internals/garbage-collection]] §4 与 [[internals/weakref]]。

### 3. asyncio 与 JS 只有三处关键差异

你已经懂事件循环了，只需补：

1. **协程是惰性的**——调用 `async def` 什么都不执行，所以
   `await a(); await b()` 是**串行**，要并发必须 `gather`/`create_task`/`TaskGroup`。
2. **取消是真的抛异常**（`CancelledError`），所以资源清理必须写在 `finally` 里。
3. **生态还分裂成同步/异步两套**——一个 `requests` 调用就能卡死整个 worker，
   这是 Node 没有的问题，也是 FastAPI 生产事故的头号原因。

见 [[bridge/async-js-vs-python]]。

### 4. FastAPI 的一切都建立在"类型注解即契约"上

一个函数签名同时决定了参数解析、校验、序列化、OpenAPI 文档和依赖注入——
底层机制就是 `inspect.signature` + 类型注解 + Pydantic。

推论链：**装饰器必须用 `functools.wraps`**（否则签名丢失，FastAPI 解析不出参数）→
**依赖注入让 `dependency_overrides` 成为最干净的测试 mock 手段** →
**入参/出参模型分离是防止字段泄漏的安全边界**。

而 FastAPI 本身只是胶水层——快的是 **ASGI + uvloop + Rust 写的 pydantic-core**。
见 [[web/fastapi-core]]、[[web/fastapi-di]]、[[web/wsgi-asgi]]。

### 5. 高分回答的共同结构

对比"背过"和"真懂"的四个信号：

| 信号 | 例子 |
|---|---|
| 区分**语言规范**与 **CPython 实现细节** | "小整数缓存是 CPython 的实现细节，PyPy 上不同" |
| 知道**版本演进** | "3.7 起 dict 有序写进了规范"；"3.13 有 free-threading 构建" |
| 知道**何时不该用** | "元类 99% 的场景该用 `__init_subclass__` 替代" |
| 说得出**权衡**而非只有优点 | "JWT 无状态但难吊销，内网单体我更倾向 session" |

见 [[interview/roadmap]] 的答题框架。

### 6. Python 的很多"设计模式"是不必要的

函数是一等对象、鸭子类型、模块天然单例、装饰器和迭代器是语言内建——
Strategy 直接传函数、Singleton 用模块级变量、Visitor 用 `singledispatch`。
被问设计模式时，讲这个比背 23 个模式有价值。见 [[sources/python-patterns]]。

## 知识地图

```text
                    ┌─────────────────────────┐
                    │  interview/  面试题库    │  ← 最终产出层
                    │  roadmap / 3 个题库/陷阱 │
                    └───────────┬─────────────┘
            ┌───────────────────┼───────────────────┐
            ▼                   ▼                   ▼
     ┌─────────────┐   ┌───────────────┐   ┌──────────────┐
     │  language/  │   │  internals/   │   │    web/      │
     │  语言核心    │   │  CPython 原理  │   │  FastAPI 栈  │
     │  16 页       │   │  6 页          │   │  9 页        │
     └──────┬──────┘   └───────┬───────┘   └──────┬───────┘
            │                  │                   │
            ▼                  ▼                   ▼
     ┌─────────────┐   ┌───────────────┐   ┌──────────────┐
     │  stdlib/    │   │ concurrency/  │   │ engineering/ │
     │  数据结构    │   │  并发与 asyncio│   │  工程实践    │
     └─────────────┘   └───────────────┘   └──────────────┘
            └───────────────────┬───────────────────┘
                                ▼
                    ┌─────────────────────────┐
                    │  bridge/  前端视角对照   │  ← 加速层（贯穿全库）
                    └─────────────────────────┘
```

## 开放问题 / 待补充

- [ ] **SQL 与数据库原理**——本库只覆盖了 ORM 侧（[[web/fastapi-async-db]]），
      索引、执行计划、事务隔离级别、MVCC 需要单独的知识库或额外资料。
      **这是当前最大的缺口**，也是前端背景最大的短板。
- [ ] **分布式基础**——缓存策略、消息队列、一致性、限流熔断，只在
      [[web/fastapi-production]] 里零散提到。
- [ ] **Django 方向**——若目标公司用 Django，需要补 ORM、Admin、DRF、
      异步视图与 ASGI 迁移。
- [ ] **数据/AI 方向**——numpy/pandas/polars 与 LLM 应用后端，目前只在
      [[bridge/npm-vs-pip]] 里列了库名。
- [ ] **真实项目**——[[interview/roadmap]] 第 6 周要求的项目还没做；
      项目里的真实踩坑经历是面试里最有说服力的素材。
- [ ] **free-threading 实测**——3.13t 的实际性能与生态兼容性还没验证过。

## 使用约定

- 每页末尾有 **相关** 区块，跟着走能形成学习路径。
- 页内的 `> **面试落点**` 引用块是**考官真正想听的话**，可以只扫这些块做快速复习。
- 代码示例用 `#=>` 标注预期输出，`# ❌` / `# ✅` 标注反面/正面写法。
- 目标版本 **Python 3.12+**，涉及版本差异处会显式标注。
