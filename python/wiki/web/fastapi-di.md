---
title: "FastAPI 依赖注入系统"
date: 2026-08-07
tags: [FastAPI, 依赖注入, Depends, yield, 缓存, 测试]
sources: ["fastapi-doc/tutorial/dependencies/index.md", "fastapi-doc/tutorial/dependencies/dependencies-with-yield.md"]
---

# FastAPI 依赖注入系统

**这是 FastAPI 最有辨识度的特性**，也是面试最容易深挖的点。
理解它 = 理解了 FastAPI 的"框架感"从哪来。

## 1. 本质：一个可调用对象的解析树

```python
from fastapi import Depends
from typing import Annotated

async def common_params(q: str | None = None, skip: int = 0, limit: int = 100):
    return {"q": q, "skip": skip, "limit": limit}

@app.get("/items/")
async def read_items(params: Annotated[dict, Depends(common_params)]):
    return params
```

FastAPI 做的事：

1. 用 `inspect.signature` 读取 `read_items` 的签名，发现 `params` 是 `Depends`。
2. **递归**分析 `common_params` 的签名——它的参数 `q`/`skip`/`limit` 是标量
   → 变成**查询参数**，出现在 OpenAPI 文档里。
3. 请求到来时：解析查询参数 → 调用 `common_params` → 把返回值注入 `params`。

**关键洞察**：依赖的参数会**继续被 FastAPI 解析**，形成一棵依赖树。
所以依赖既能声明自己需要什么请求数据，又能被复用。

> 前端类比：≈ React Hooks（可组合、可嵌套、有缓存）+ Angular DI（声明式注入）。
> 但它是**基于函数签名和类型注解**的，不需要装饰器或容器注册。

## 2. 依赖的几种形式

```python
# ① 函数依赖（最常见）
def get_db(): ...

# ② 类依赖（可调用对象）—— __init__ 的参数就是请求参数
class Pagination:
    def __init__(self, skip: int = 0, limit: int = 100):
        self.skip, self.limit = skip, limit

@app.get("/x")
async def x(p: Annotated[Pagination, Depends()]):     # Depends() 空参 = 用类型注解本身
    p.skip, p.limit

# ③ 带参数的依赖（依赖工厂）
def require_role(role: str):
    async def checker(user: Annotated[User, Depends(get_current_user)]):
        if role not in user.roles:
            raise HTTPException(403, "forbidden")
        return user
    return checker

@app.get("/admin", dependencies=[Depends(require_role("admin"))])
async def admin_only(): ...

# ④ 只为副作用的依赖（不需要返回值）—— 放在 dependencies= 列表里
@app.get("/x", dependencies=[Depends(verify_token), Depends(rate_limit)])
async def x(): ...

# ⑤ 全局依赖
app = FastAPI(dependencies=[Depends(verify_api_key)])
# 或整组
router = APIRouter(dependencies=[Depends(verify_token)])
```

## 3. 嵌套依赖与缓存（高频考点）

```python
async def get_db() -> AsyncSession: ...

async def get_current_user(
    token: Annotated[str, Depends(oauth2_scheme)],
    db: Annotated[AsyncSession, Depends(get_db)],       # 依赖的依赖
) -> User: ...

async def get_active_user(
    user: Annotated[User, Depends(get_current_user)],
) -> User:
    if not user.is_active:
        raise HTTPException(400, "inactive")
    return user

@app.get("/me")
async def me(
    user: Annotated[User, Depends(get_active_user)],
    db: Annotated[AsyncSession, Depends(get_db)],       # ← 同一个请求里 get_db 只执行一次
):
    ...
```

> **面试落点**：**同一个请求内，同一个依赖默认只执行一次，结果被缓存并复用**
> （缓存 key 是"依赖函数 + 参数"）。这就是为什么 `get_db` 在依赖树里出现多次
> 却只创建一个 session。想关掉缓存用 `Depends(fn, use_cache=False)`。

```python
Depends(get_random_token, use_cache=False)    # 每次都重新调用
```

## 4. `yield` 依赖：请求级资源管理 ★

```python
async def get_db():
    async with AsyncSessionLocal() as session:
        try:
            yield session              # ← yield 之前：请求开始时执行
            await session.commit()
        except Exception:
            await session.rollback()
            raise
        # ← yield 之后：响应发送后执行（收尾）
```

执行顺序（FastAPI **0.118.0+** 实测，`scope="request"` 即默认行为）：

```text
请求进入
  → 依赖 A 的 yield 前
    → 依赖 B 的 yield 前（B 依赖 A）
      → 路由函数执行
      → 响应发送给客户端
      → 后台任务（BackgroundTasks）执行
    → 依赖 B 的 yield 后（收尾）
  → 依赖 A 的 yield 后（收尾）
```

实测验证（FastAPI 0.141）：

```python
def dep_yield():
    order.append("dep:enter"); yield "res"; order.append("dep:exit")

@app.get("/z")
def z(bg: BackgroundTasks, r=Depends(dep_yield)):
    order.append("route"); bg.add_task(lambda: order.append("background"))

#=> ['dep:enter', 'route', 'background', 'dep:exit']
#              后台任务在收尾【之前】↑
```

**收尾代码在响应发送之后执行**——这意味着：

- 在 `yield` 之后**不能再抛 `HTTPException` 来改变响应**（响应已经发出去了）。
- **后台任务跑在 yield 依赖收尾之前，所以请求级的 DB session 此刻仍然是打开的。**

> ⚠️ **这个顺序在 FastAPI 历史上反复变过，是最容易记错的点之一**：
>
> | 版本 | yield 收尾的时机 | 后台任务能否用请求 session |
> |---|---|---|
> | < 0.106.0 | 响应发送后、**后台任务之后** | ✅ 能 |
> | 0.106.0 ~ 0.117 | 路由返回后、**响应发送前** | ❌ 不能（session 已关） |
> | **≥ 0.118.0（当前）** | 响应发送后、**后台任务之后** | ✅ 能 |
> | 0.121.0+ 且显式 `Depends(scope="function")` | 路由返回后立即 | ❌ 不能 |
>
> **但官方仍然建议后台任务自建资源**——理由不是"session 已关闭"，而是
> **后台任务应该是独立的逻辑单元，有自己的生命周期和连接**；
> 复用请求 session 会让请求的收尾被后台任务拖长，也容易出现跨任务的事务纠缠。

```python
# ⚠️ 当前版本能跑通，但不推荐：后台任务会拖住请求的 session 不释放
@app.post("/x")
async def x(bg: BackgroundTasks, db: Annotated[AsyncSession, Depends(get_db)]):
    bg.add_task(send_email, db)

# ✅ 推荐：只传 ID，后台任务自己开 session
@app.post("/x")
async def x(bg: BackgroundTasks, db: Annotated[AsyncSession, Depends(get_db)]):
    order = await create_order(db)
    bg.add_task(send_email, order.id)

async def send_email(order_id: int):
    async with AsyncSessionLocal() as db:
        order = await db.get(Order, order_id)
        ...
```

`yield` 依赖里捕获异常必须重抛：

```python
async def get_db():
    async with SessionLocal() as s:
        try:
            yield s
        except SomeError:
            log.error(...)
            raise            # ✅ 不重抛的话，FastAPI 无法知道请求失败了
```

## 5. 依赖覆盖：测试的杀手锏 ★

```python
# 生产依赖
async def get_db(): ...

# 测试里替换掉
from app.main import app

async def override_get_db():
    async with TestSessionLocal() as s:
        yield s

app.dependency_overrides[get_db] = override_get_db
# 测完清理
app.dependency_overrides.clear()
```

> **面试落点**：「FastAPI 怎么做单元测试的依赖 mock？」
> 答：`app.dependency_overrides[原依赖] = 假依赖`。
> 不需要 monkeypatch，也不需要 DI 容器——**这正是 DI 设计带来的可测试性**，
> 是"为什么要用依赖注入而不是在函数里直接 `SessionLocal()`"的最佳论据。

见 [[web/fastapi-testing]]。

## 6. 依赖注入解决的实际问题

| 问题 | 依赖注入的解法 |
|---|---|
| 数据库会话生命周期 | `yield` 依赖，请求结束自动关闭/提交/回滚 |
| 鉴权与权限 | `get_current_user` → `require_role("admin")` 链式复用 |
| 参数校验复用 | 分页参数、过滤参数抽成依赖 |
| 配置注入 | `Settings` 依赖 + `lru_cache` |
| 多租户 / 请求上下文 | 从 header 解析租户 ID 的依赖 |
| 限流 | `dependencies=[Depends(rate_limiter)]` |
| 单元测试 | `dependency_overrides` |

## 7. 配置注入的标准写法

```python
from functools import lru_cache
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")
    app_name: str = "MyAPI"
    database_url: str
    secret_key: str
    debug: bool = False

@lru_cache                              # ← 只创建一次（读 .env 是 I/O）
def get_settings() -> Settings:
    return Settings()

SettingsDep = Annotated[Settings, Depends(get_settings)]

@app.get("/info")
async def info(settings: SettingsDep):
    return {"app": settings.app_name}

# 测试里覆盖
app.dependency_overrides[get_settings] = lambda: Settings(database_url="sqlite://", secret_key="t")
```

> 注意：`@lru_cache` 配合依赖是官方文档推荐的模式——
> 依赖本身很轻（只是返回缓存对象），但语义上仍是可覆盖的注入点。

## 8. 与其它 DI 方案的对比

| | FastAPI `Depends` | Spring / NestJS 容器 | 手工传参 |
|---|---|---|---|
| 注册 | **无需注册**，用函数本身 | 需要装饰器/模块声明 | — |
| 作用域 | 请求级（默认缓存一次） | singleton/request/transient 可配 | — |
| 生命周期 | `yield` 前后 | `@PostConstruct` 等 | 手写 |
| 测试替换 | `dependency_overrides` | 容器替换 | 传参 |
| 类型安全 | 基于注解 | 基于装饰器元数据 | 天然 |

**局限**（面试里主动说出来会加分）：

- FastAPI 的依赖是**请求作用域**的，**没有真正的 singleton 作用域**——
  应用级单例要靠模块级变量、`lru_cache` 或 `app.state` / lifespan。
- 依赖只在**路由处理链路**里生效；后台任务、Celery worker、CLI 脚本里用不上，
  需要自己写 `async with get_session()`。
- 复杂依赖树会让 OpenAPI 文档里出现意外的查询参数（依赖的标量参数会冒泡上来）。

```python
# 应用级单例的正确位置：lifespan + app.state
@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.redis = await create_redis_pool()
    yield
    await app.state.redis.close()

def get_redis(request: Request):
    return request.app.state.redis        # 通过依赖暴露给路由
```

## 相关

- [[web/fastapi-core]] —— 路由与参数
- [[web/fastapi-testing]] —— `dependency_overrides` 实战
- [[web/fastapi-async-db]] —— session 依赖的完整实现
- [[web/fastapi-auth]] —— 鉴权依赖链
- [[web/fastapi-architecture]] —— lifespan 与应用级资源
- [[language/functions-arguments]] —— `inspect.signature` 是 DI 的基础
- [[language/context-managers]] —— yield 依赖与上下文管理器的关系
- [[interview/question-bank-web]] —— 依赖注入面试题
