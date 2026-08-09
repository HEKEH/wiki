---
title: "FastAPI 项目架构与中间件"
date: 2026-08-07
tags: [架构, 分层, 中间件, lifespan, 后台任务, 项目结构]
sources: ["fastapi-doc/tutorial/bigger-applications.md", "fastapi-doc/advanced/events.md", "fastapi-doc/tutorial/middleware.md", "fastapi-doc/tutorial/background-tasks.md", "fastapi-doc/tutorial/handling-errors.md"]
---

# FastAPI 项目架构与中间件

面试到中高级岗位，**「你会怎么组织一个 FastAPI 项目」** 几乎必问。
这一页给出一套可直接答的分层方案。

## 1. 推荐目录结构

```text
myapp/
├── pyproject.toml
├── alembic.ini
├── .env.example
├── Dockerfile / docker-compose.yml
├── migrations/                     # Alembic
├── src/
│   └── app/
│       ├── __init__.py
│       ├── main.py                 # 创建 FastAPI 实例、注册路由和中间件
│       ├── config.py               # Settings（pydantic-settings）
│       ├── db.py                   # engine / session factory
│       ├── deps.py                 # 公共依赖（get_db / get_current_user）
│       ├── exceptions.py           # 业务异常与 handler
│       ├── middleware.py
│       ├── api/
│       │   ├── __init__.py
│       │   └── v1/
│       │       ├── router.py       # 汇总所有子路由
│       │       ├── users.py
│       │       └── orders.py
│       ├── schemas/                # Pydantic 模型（API 契约）
│       │   ├── user.py
│       │   └── order.py
│       ├── models/                 # SQLAlchemy 模型（DB 表）
│       ├── repositories/           # 数据访问层
│       ├── services/               # 业务逻辑层 ★ 不依赖 FastAPI
│       ├── workers/                # 任务队列的任务
│       └── core/                   # 安全、日志、工具
└── tests/
    ├── conftest.py
    ├── unit/
    └── integration/
```

**另一种流派：按功能垂直切分**（大团队、微服务倾向时更好）：

```text
app/
├── users/
│   ├── router.py  schemas.py  models.py  service.py  repository.py
├── orders/
│   └── ...
└── core/
```

> **面试落点**：能说清**两种切法的取舍**——
> 按技术分层（`schemas/` `services/`）适合中小项目、职责清晰；
> 按功能垂直切（`users/` `orders/`）适合大项目，改一个功能只动一个目录，
> 也更容易拆成微服务。**并明确"业务逻辑层不能 import fastapi"**——
> 这样它才能被 CLI、定时任务、消息消费者复用，也更好测试。

## 2. 分层职责

```text
路由层 (api/)         ← 只做：解析参数、调用 service、映射异常到 HTTP 状态码
   │  依赖注入
业务层 (services/)     ← 业务规则、事务边界、跨仓储编排。【不 import fastapi】
   │
数据层 (repositories/) ← 只做 SQL/ORM 操作，不含业务判断
   │
模型层 (models/)       ← SQLAlchemy 表定义
```

**契约层 (schemas/)** 横切：`XxxCreate` / `XxxUpdate` / `XxxPublic` / `XxxInDB`。

```python
# schemas/user.py
class UserBase(BaseModel):
    email: EmailStr
    name: str

class UserCreate(UserBase):
    password: str                    # 入参有密码

class UserUpdate(BaseModel):
    name: str | None = None          # 全部可选（PATCH 语义）

class UserPublic(UserBase):           # 出参无密码 ★
    model_config = ConfigDict(from_attributes=True)
    id: int
    created_at: datetime

class UserInDB(UserPublic):
    hashed_password: str              # 内部用
```

## 3. main.py 与应用工厂

```python
# app/main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI):
    # ── 启动 ──
    app.state.redis = await create_redis_pool(settings.redis_url)
    app.state.http = httpx.AsyncClient(timeout=10)
    setup_logging()
    yield
    # ── 关闭 ──（收到 SIGTERM 后执行，务必让它快速完成）
    await app.state.http.aclose()
    await app.state.redis.aclose()
    await engine.dispose()

def create_app() -> FastAPI:          # ★ 应用工厂：测试里可以造多个隔离实例
    app = FastAPI(
        title=settings.app_name,
        lifespan=lifespan,
        docs_url="/docs" if settings.debug else None,
        default_response_class=ORJSONResponse,
    )
    app.add_middleware(CORSMiddleware, allow_origins=settings.cors_origins, ...)
    app.add_middleware(RequestIDMiddleware)
    register_exception_handlers(app)
    app.include_router(api_v1_router, prefix="/api/v1")

    @app.get("/health", include_in_schema=False)
    async def health():
        return {"status": "ok"}

    return app

app = create_app()
```

> **`lifespan` 已取代 `@app.on_event("startup"/"shutdown")`**（后者自 0.93 起弃用）。
> 它是一个 `@asynccontextmanager`，yield 前是启动、yield 后是关闭——
> 见 [[language/context-managers]]。

## 4. 中间件

```python
# app/middleware.py
import time, uuid
from starlette.middleware.base import BaseHTTPMiddleware

class RequestIDMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        rid = request.headers.get("X-Request-ID", str(uuid.uuid4()))
        request_id_ctx.set(rid)              # contextvars，见 asyncio-patterns
        request.state.request_id = rid
        start = time.perf_counter()
        try:
            response = await call_next(request)
        except Exception:
            log.exception("unhandled", extra={"rid": rid})
            raise
        elapsed = time.perf_counter() - start
        response.headers["X-Request-ID"] = rid
        response.headers["X-Process-Time"] = f"{elapsed:.4f}"
        log.info("%s %s %d %.3fs", request.method, request.url.path,
                 response.status_code, elapsed, extra={"rid": rid})
        return response
```

**执行顺序**（洋葱模型，和 Koa 一样）：

```text
后添加的中间件在【外层】
app.add_middleware(A)      ← 最先执行请求、最后执行响应
app.add_middleware(B)
app.add_middleware(C)      ← 最贴近路由

请求：A → B → C → 依赖 → 路由函数
响应：路由函数 → 依赖收尾 → C → B → A
```

> ⚠️ `BaseHTTPMiddleware` 有已知的性能开销和**流式响应/后台任务的边界问题**。
> 性能敏感或需要处理 WebSocket 时，写**纯 ASGI 中间件**（见 [[web/wsgi-asgi]]）。

**中间件 vs 依赖的选择**：

| | 中间件 | 依赖 |
|---|---|---|
| 作用范围 | 所有请求（含 404、静态文件） | 声明了它的路由 |
| 能否访问路由信息 | 有限（`request.scope["route"]`） | 完整 |
| 能否返回值给路由 | 只能塞 `request.state` | **直接注入参数** ✅ |
| 能否被测试覆盖 | 需要整体替换 | `dependency_overrides` ✅ |
| 适合 | 日志、追踪、CORS、压缩、限流 | 鉴权、DB session、参数校验 |

**经验法则**：**能用依赖就用依赖**，只有真正需要"所有请求都过"时才用中间件。

## 5. 异常处理

```python
# app/exceptions.py
class AppError(Exception):
    status_code = 500
    code = "internal_error"
    def __init__(self, msg: str = "", **extra):
        self.msg, self.extra = msg, extra
        super().__init__(msg)

class NotFound(AppError):      status_code, code = 404, "not_found"
class Conflict(AppError):      status_code, code = 409, "conflict"
class PermissionDenied(AppError): status_code, code = 403, "forbidden"

def register_exception_handlers(app: FastAPI):
    @app.exception_handler(AppError)
    async def app_error_handler(request: Request, exc: AppError):
        return JSONResponse(
            status_code=exc.status_code,
            content={"code": exc.code, "message": exc.msg, **exc.extra},
        )

    @app.exception_handler(RequestValidationError)
    async def validation_handler(request: Request, exc: RequestValidationError):
        return JSONResponse(status_code=422, content={
            "code": "validation_error",
            "errors": [{"field": ".".join(map(str, e["loc"][1:])), "msg": e["msg"]}
                       for e in exc.errors()],
        })

    @app.exception_handler(Exception)                 # 兜底，防止泄漏堆栈
    async def unhandled(request: Request, exc: Exception):
        log.exception("unhandled error")
        return JSONResponse(500, content={"code": "internal_error", "message": "服务异常"})
```

**价值**：业务层只 `raise NotFound("user")`，不需要知道 HTTP——**这是分层解耦的关键一步**。

## 6. 后台任务

```python
from fastapi import BackgroundTasks

@router.post("/orders")
async def create_order(data: OrderIn, bg: BackgroundTasks, db: DB):
    order = await service.create(data)
    bg.add_task(send_confirmation_email, order.id)    # 响应返回【之后】执行
    return order
```

**`BackgroundTasks` 的适用边界（重要）**：

| | BackgroundTasks | 任务队列（Celery/arq/Dramatiq） |
|---|---|---|
| 执行位置 | **同一个进程** | 独立 worker 进程/机器 |
| 进程崩溃 | **任务丢失** ❌ | 有持久化，可重试 |
| 重试 | 无 | 有 |
| 定时任务 | 无 | 有（beat / cron） |
| 适合 | 发个日志、清个缓存、几十毫秒的收尾 | 发邮件、生成报表、调外部 API、任何可能失败的事 |

> **面试落点**：「后台任务用 BackgroundTasks 还是 Celery？」
> 答：**BackgroundTasks 只适合"丢了也无所谓"的轻量收尾**，因为它跑在同一个进程里、
> 没有持久化和重试，且会占用 worker 的处理能力（进而影响并发）。
> **任何有业务意义的异步任务都应该进消息队列**。
> 再补一句资源边界：**后台任务应该自建资源（自己开 session），只从请求里接 ID 之类的纯数据**。
> ⚠️ 注意别说成"session 已经被关闭了"——**FastAPI ≥ 0.118 里后台任务跑在 yield 依赖收尾之前，
> session 其实还开着**（这个顺序在 0.106 和 0.118 两次反转过）。
> 不复用的真正理由是**生命周期解耦**，不是技术上不可行。见 [[web/fastapi-di]]。

```python
# arq（asyncio 原生任务队列，与 FastAPI 最搭）
from arq import create_pool
from arq.connections import RedisSettings

async def send_email(ctx, order_id: int): ...

class WorkerSettings:
    functions = [send_email]
    redis_settings = RedisSettings.from_dsn(settings.redis_url)

# 路由里入队
await app.state.arq.enqueue_job("send_email", order.id)
```

## 7. API 版本化

```python
# 路径版本（最常见、最简单）
app.include_router(v1_router, prefix="/api/v1")
app.include_router(v2_router, prefix="/api/v2")

# 子应用（完全独立的 OpenAPI 文档）
v2 = FastAPI(title="API v2")
app.mount("/api/v2", v2)
```

**兼容性原则**：加字段是兼容的，**删字段/改类型/改语义是破坏性的**。
用 `deprecated=True` 标记待废弃端点，OpenAPI 会体现出来，前端能提前看到。

## 8. 一段能背的架构陈述

> 我会按分层组织：**路由层只做参数解析和异常映射，业务层放规则和事务边界且不依赖 FastAPI，
> 仓储层封装 ORM 查询**。请求/响应用独立的 Pydantic 模型（`Create`/`Update`/`Public`），
> 与 DB 模型解耦，避免内部字段泄漏。
>
> **依赖注入**管理 session、当前用户和配置；`lifespan` 管理连接池等应用级资源；
> **中间件**只做横切关注点（request ID、日志、CORS）。
> 业务异常定义成异常类，用全局 handler 统一映射成 HTTP 响应。
>
> **有业务意义的异步工作走任务队列**（arq/Celery），不用 BackgroundTasks。
> 数据库迁移用 Alembic 版本化管理，且保证向后兼容以支持滚动发布。
>
> 项目变大后会从"按技术分层"切到"按功能垂直切分"，为将来拆服务留余地。

## 相关

- [[web/fastapi-core]] —— 路由与响应
- [[web/fastapi-di]] —— 依赖注入
- [[web/fastapi-async-db]] —— 仓储层与事务
- [[web/fastapi-production]] —— 部署与可观测性
- [[web/fastapi-testing]] —— 分层带来的可测试性
- [[language/modules-imports]] —— src layout 与循环导入
- [[language/exceptions]] —— 异常体系设计
- [[interview/question-bank-web]] —— 架构面试题
