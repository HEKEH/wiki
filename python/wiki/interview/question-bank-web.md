---
title: "题库：Web 后端与工程"
date: 2026-08-07
tags: [面试题, FastAPI, 数据库, 架构, 部署, 参考答案]
sources: ["fastapi-doc/async.md", "interview-python-cn.md"]
---

# 题库：Web 后端与工程

## A. 框架与协议

**Q1. WSGI 和 ASGI 的区别？**

> WSGI（PEP 3333）是**同步**接口：`app(environ, start_response)` 返回可迭代的字节，
> 一个请求占一个线程从头到尾，**无法支持 WebSocket**。
> ASGI 是**异步**接口：`async app(scope, receive, send)`，用事件流通信，
> 支持 **HTTP + WebSocket + lifespan** 三种 scope 类型，单线程能处理海量并发连接。
> Flask/Django(旧) 是 WSGI，FastAPI/Starlette/Django(新) 是 ASGI。

**Q2. gunicorn 和 uvicorn 是什么关系？**

> **uvicorn 是 ASGI 服务器**（基于 uvloop + httptools）；
> **gunicorn 是进程管理器 + WSGI 服务器**。
> 传统组合是 `gunicorn -k uvicorn.workers.UvicornWorker`——gunicorn 管进程和优雅重启，
> uvicorn 处理 ASGI 协议。
> **但容器化环境下我倾向单容器单 uvicorn 进程**，把副本管理交给 K8s，
> 避免容器内再套一层进程管理带来的健康检查和优雅关闭的复杂性。

**Q3. FastAPI 为什么快？**

> 三个来源：① **ASGI + asyncio**，I/O 密集场景不再"一请求一线程"；
> ② **uvicorn 用 uvloop（libuv）+ httptools（C 解析器）**；
> ③ **Pydantic v2 的核心是 Rust 写的 pydantic-core**。
> FastAPI 本身只是胶水层——**快的是它站的这几个肩膀**。

**Q4. FastAPI vs Flask vs Django 怎么选？**

> **Django**：内容型/后台型项目，自带 ORM/Admin/Auth/表单，开发最快，生态最全。
> **Flask**：小而灵活的服务、需要完全自主组装时。
> **FastAPI**：API 优先、高并发 I/O、需要自动文档和类型安全。
> FastAPI 的短板是生态年轻、没有内置 Admin/Auth/ORM，这些要自己拼装——
> 团队小且需求标准时 Django 反而更快。

## B. FastAPI 核心

**Q5. `async def` 路由和 `def` 路由的区别？**（必考）

> `async def` 直接在**事件循环**里执行；`def` 会被 FastAPI **自动放进线程池**
> （Starlette 通过 anyio，默认 40 个线程）。
>
> **选择规则**：全是异步库 → `async def`；用了同步阻塞库（psycopg2/requests/pandas/PIL）
> → **写成 `def`**。最危险的是 `async def` 里调用同步阻塞代码，
> 会阻塞整个 worker 的所有并发请求。

**Q6. FastAPI 怎么知道一个参数是查询参数还是请求体？**

> 靠 **`inspect.signature` 读取函数签名 + 类型注解**：
> 名字出现在路径模板里 → 路径参数；标量类型 → 查询参数；
> Pydantic 模型 → 请求体；显式 `Header()`/`Cookie()`/`Form()` → 对应位置；
> `Depends()` → 依赖注入。
> 这也解释了**为什么装饰器必须用 `functools.wraps`**——签名丢了 FastAPI 就解析不出参数。

**Q7. 依赖注入解决了什么问题？同一个依赖会被调用几次？**

> 解决资源生命周期（DB session）、鉴权复用、参数校验复用、配置注入、**可测试性**。
> **同一个请求内，同一个依赖默认只执行一次，结果被缓存**（key 是依赖函数+参数），
> 所以 `get_db` 在依赖树里出现多次也只创建一个 session。用 `Depends(fn, use_cache=False)` 可关闭。

**Q8. `yield` 依赖的执行时机？**

> `yield` 之前在请求开始时执行，`yield` **之后在响应发送完毕后**执行。
> 推论：**yield 之后不能再抛 `HTTPException` 改变响应**（响应已经发出去了）。
>
> **顺序细节（容易答错，实测于 0.141）**：完整顺序是
> `依赖 yield 前 → 路由 → 响应发送 → 后台任务 → 依赖 yield 后`，
> 也就是**后台任务跑在依赖收尾之前，请求级 session 此刻还开着**。
>
> ⚠️ 这个顺序 FastAPI **反转过两次**：< 0.106 是"收尾在后台任务之后"；
> 0.106~0.117 改成"收尾在响应发送前"（那个时期后台任务确实用不了 session）；
> **≥ 0.118 又改回"收尾在后台任务之后"**；0.121+ 可用 `Depends(scope="function")` 选择提前收尾。
>
> 所以正确说法是：**后台任务应该自建资源、只接收 ID 这类纯数据——理由是生命周期解耦，
> 而不是"session 已关闭"**。能说清这个版本演进比背结论强得多。

**Q9. 怎么 mock 掉数据库做单元测试？**

> `app.dependency_overrides[get_db] = fake_get_db`，测完 `clear()`。
> **不需要 monkeypatch 也不需要改生产代码**——这正是依赖注入设计的回报。

**Q10. 怎么防止密码等字段泄漏到响应？**

> **入参和出参用不同的 Pydantic 模型**（`UserCreate` 含密码，`UserPublic` 不含），
> 用 `response_model=UserPublic` 强制过滤——即使路由返回了完整对象，
> 序列化时也只输出声明的字段。

## C. Pydantic

**Q11. Pydantic 和 dataclass 的区别？**

> **dataclass 只生成样板代码，没有任何运行时校验**；
> Pydantic 会**校验并强制转换类型**，还能序列化和生成 JSON Schema。
> 架构上：**边界（API/配置/消息）用 Pydantic，内部数据结构用 dataclass**——
> 因为 Pydantic 的校验在高 QPS 下是真实成本。

**Q12. Pydantic v1 到 v2 有哪些变化？**

> API：`.dict()`→`.model_dump()`、`.json()`→`.model_dump_json()`、
> `parse_obj()`→`.model_validate()`、`class Config`→`model_config = ConfigDict(...)`、
> `@validator`→`@field_validator`（且必须加 `@classmethod`）、
> `@root_validator`→`@model_validator`、`orm_mode`→`from_attributes`。
> 内核：**用 Rust 重写（pydantic-core），快 5~50 倍**；类型转换更严格（`1.9` → int 会报错，v1 会截断）。

**Q13. Pydantic 会自动转换类型吗？**

> 默认 **lax 模式会转换**（`"42"` → `42`），这对 HTTP 场景是合理的（传来的都是字符串）。
> 需要严格时用 `ConfigDict(strict=True)` 或 `StrictInt`。
> **API 入口保持 lax，内部关键逻辑用 strict** 是常见做法。

## D. 数据库

**Q14. session 的作用域应该是什么？**

> **一个请求一个 session**，通过 `yield` 依赖管理，请求成功 commit、失败 rollback、
> 最后 close。**绝不能用全局 session**——不是协程安全的，且事务会互相污染。

**Q15. 什么是 N+1？怎么发现和解决？**（必考）

> 查 N 条主记录后，访问每条的关联对象各发一次 SQL，共 N+1 次查询。
> **发现**：开 SQL 日志/APM 看查询次数；或给关系设 `lazy="raise"` 让懒加载直接报错。
> **解决**：`selectinload`（一对多，额外一条 IN 查询）、`joinedload`（多对一，LEFT JOIN）、
> 或只 select 需要的列。
> **一对多不要用 joinedload**——JOIN 会让主表行数被乘开，传输大量冗余数据。

**Q16. `expire_on_commit=False` 是干什么的？**

> 默认 `True` 时 commit 后所有对象属性被标记为过期，下次访问要重新查询；
> 在 async 场景里如果此时 session 已关闭，就会抛 `MissingGreenlet`/`DetachedInstanceError`。
> 设为 `False` 可以避免这个最常见的报错。

**Q17. 数据库迁移怎么做？上线时要注意什么？**

> 用 **Alembic** 版本化管理，`revision --autogenerate` + `upgrade head`。
> **生产绝不用 `create_all()`**。
> 关键是**迁移要向后兼容**：滚动发布期间新旧代码同时在跑，
> 所以要"先加列（可空）→ 双写 → 回填数据 → 切读 → 删旧列"分多次发布，
> 不能一次性改列名或删列。

**Q18. 连接池怎么配？**

> `pool_size` + `max_overflow` 控制连接数，`pool_pre_ping=True` 避免死连接，
> `pool_recycle=1800` 定期回收。
> **关键约束：worker 数 × pool_size ≤ 数据库的 max_connections**。
> 4 个 worker × pool_size 20 = 80 个连接，很容易打爆 PostgreSQL 默认的 100。
> 大规模时前面加 **pgbouncer** 做连接复用。

## E. 架构与工程

**Q19. 你会怎么组织一个 FastAPI 项目？**

> 分层：**路由层只做参数解析和异常映射；业务层放规则和事务边界且不 import fastapi；
> 仓储层封装 ORM 查询**。请求/响应用独立的 Pydantic 模型与 DB 模型解耦。
> 依赖注入管 session/用户/配置，lifespan 管应用级资源（连接池、HTTP 客户端），
> 中间件只做横切关注点。业务异常定义成异常类，全局 handler 统一映射成 HTTP 响应。
> 项目变大后从"按技术分层"切到"按功能垂直切分"。
>
> **"业务层不依赖框架"是关键**——这样它才能在 CLI、Celery worker、测试里复用。

**Q20. 中间件和依赖怎么选？**

> 中间件对**所有请求**生效（含 404、静态文件），适合日志、追踪、CORS、压缩；
> 依赖只对**声明了它的路由**生效，能**直接把值注入参数**，且能被 `dependency_overrides` 替换。
> **能用依赖就用依赖。**

**Q21. 后台任务用 BackgroundTasks 还是 Celery？**

> `BackgroundTasks` 跑在**同一个进程**里，**没有持久化和重试，进程崩溃就丢**，
> 还会占用 worker 的处理能力。只适合"丢了也无所谓"的轻量收尾。
> **任何有业务意义的异步任务都该进消息队列**（Celery / arq / Dramatiq），
> 它们有持久化、重试、延时、定时和独立扩容。

**Q22. 怎么保证服务优雅关闭？**

> 收到 SIGTERM 后：停止接受新连接 → 等待进行中的请求完成 → 触发 lifespan 关闭段
> （关连接池、刷缓冲）→ 退出。
> K8s 上还要配 `terminationGracePeriodSeconds` 大于 graceful timeout，
> 并加 **preStop 的 sleep**——因为"从 Service endpoints 摘除"和"发 SIGTERM"是并行的，
> 不等一下会有流量打到正在关闭的 Pod 上。

**Q23. 服务变慢了怎么排查？**（必考）

> ① 看监控确认是延迟还是错误率、全局还是单个端点；
> ② 看链路追踪定位是 DB / 外部调用 / 应用自身；
> ③ 应用自身慢就用 **`py-spy record` 生成火焰图**（无侵入，生产可用），
>    重点看是不是阻塞了事件循环；
> ④ DB 慢就看慢查询日志 + `EXPLAIN ANALYZE` + 查 N+1 和缺索引；
> ⑤ 确认连接池/线程池有没有排队。

**Q24. worker 开几个？**

> 纯异步 I/O 密集时 worker ≈ CPU 核数就够（并发靠协程不靠进程）；
> 路由里有同步阻塞或 CPU 计算时考虑 `2×核数+1`，并**压测确认**。
> 每个 worker 是完整的 Python 进程，内存开销要算进容量规划。

## F. 安全

| 问题 | 要点 |
|---|---|
| **JWT 的缺点** | **签发后无法主动失效**（只能短过期+黑名单，但黑名单又让它失去无状态优势）；payload 只是编码不是加密；体积比 session id 大 |
| Session vs JWT | Session 有状态、易吊销、需共享存储；JWT 无状态、跨服务友好、难吊销。**内网单体我更倾向 session** |
| JWT 存哪 | httpOnly + Secure + SameSite Cookie（防 XSS）优于 localStorage；用 Cookie 要防 CSRF |
| 密码怎么存 | bcrypt/argon2id 等**慢哈希** + 自带 salt；绝不用 MD5/SHA 直接哈希 |
| bcrypt 在 async 里的坑 | 哈希要 100~300ms，在 `async def` 里直接调用会阻塞事件循环，要 `to_thread` |
| 防 SQL 注入 | 参数绑定，不拼字符串；ORM 默认安全但 `text()` 要注意 |
| 越权（IDOR） | **路由级角色检查不够**，必须在业务层校验"这条数据是不是你的" |
| Python 特有风险 | `pickle.loads` 不可信数据 = **任意代码执行**；`eval/exec`；`yaml.load` 要用 `safe_load`；`subprocess(shell=True)` 拼接 |
| 限流 | 登录/写接口必须限流（slowapi / 网关层） |

## G. 工程化

| 问题 | 答案 |
|---|---|
| 依赖怎么管 | **uv** + `pyproject.toml` + **`uv.lock` 提交进仓库**；CI 用 `uv sync --frozen` |
| 虚拟环境为什么必须 | Python 没有 node_modules，默认装到全局 site-packages，会冲突 |
| 代码规范怎么保证 | **ruff check + ruff format** 在 pre-commit 和 CI 里强制，不靠人记 PEP 8 |
| 类型检查 | mypy strict（老项目按模块渐进开启）；ruff 的 `ASYNC` 规则能抓 async 里的阻塞调用 |
| 测试怎么写 | pytest + fixture + 参数化；DB 用**事务回滚 fixture** 或 testcontainers；`dependency_overrides` 替换依赖 |
| 覆盖率多少合适 | 覆盖率是"没测到什么"的指标不是质量指标；关注**核心路径 + 边界 + 错误分支**，用**分支覆盖**而非行覆盖 |
| Docker 镜像怎么优化 | 多阶段构建、依赖层与代码层分开 COPY、slim 基础镜像、非 root 用户、`PYTHONUNBUFFERED=1` |

## 相关

- [[web/wsgi-asgi]] / [[web/fastapi-core]] / [[web/fastapi-di]] / [[web/pydantic]]
- [[web/fastapi-async-db]] / [[web/fastapi-auth]] / [[web/fastapi-architecture]]
- [[web/fastapi-testing]] / [[web/fastapi-production]]
- [[engineering/packaging-envs]] / [[engineering/tooling-quality]] / [[engineering/testing]]
- [[interview/roadmap]] —— 复习路线
