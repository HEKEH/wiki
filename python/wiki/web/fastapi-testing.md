---
title: "FastAPI 测试"
date: 2026-08-07
tags: [测试, pytest, TestClient, httpx, fixture, mock]
sources: ["fastapi-doc/tutorial/testing.md", "fastapi-doc/advanced/async-tests.md"]
---

# FastAPI 测试

FastAPI 的可测试性是它的隐藏优势——**依赖注入 + Starlette 的 TestClient** 让
"不启服务器就能测完整请求链路"变得非常自然。

## 1. 同步测试：`TestClient`

```python
# tests/test_api.py
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)          # 内部用 httpx + portal 驱动 ASGI，不真的开端口

def test_read_item():
    r = client.get("/items/1")
    assert r.status_code == 200
    assert r.json() == {"id": 1, "name": "foo"}

def test_create():
    r = client.post("/items/", json={"name": "x", "price": 1.0})
    assert r.status_code == 201

def test_auth():
    r = client.get("/me", headers={"Authorization": f"Bearer {token}"})
    assert r.status_code == 200

# lifespan 也想跑（默认 TestClient 只在 with 块里触发 lifespan）
with TestClient(app) as client:
    ...
```

**`TestClient` 能测 `async def` 路由**——它内部起了一个事件循环。
所以**大多数集成测试用同步 `TestClient` 就够了**，不必上 pytest-asyncio。

## 2. 异步测试：`httpx.AsyncClient` + `ASGITransport`

需要在测试里 `await` 异步 fixture（如异步 DB session）时才需要：

```python
# pip install pytest-asyncio httpx
import pytest
from httpx import AsyncClient, ASGITransport

@pytest.mark.asyncio
async def test_async():
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as ac:
        r = await ac.get("/items/1")
    assert r.status_code == 200
```

```toml
# pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"        # 不用每个测试加 @pytest.mark.asyncio
```

## 3. 依赖覆盖 —— 核心技巧 ★

```python
# tests/conftest.py
import pytest
from fastapi.testclient import TestClient
from app.main import app
from app.deps import get_db, get_current_user
from app.models import User

@pytest.fixture
def fake_user():
    return User(id=1, email="t@x.com", name="test", is_active=True)

@pytest.fixture
def client(db_session, fake_user):
    app.dependency_overrides[get_db] = lambda: db_session
    app.dependency_overrides[get_current_user] = lambda: fake_user
    with TestClient(app) as c:
        yield c
    app.dependency_overrides.clear()        # ★ 一定要清理，否则污染其它测试
```

> **面试落点**：「FastAPI 怎么 mock 数据库/鉴权？」
> 答：`app.dependency_overrides[真实依赖] = 假依赖`——
> **不需要 monkeypatch、不需要改生产代码**。这正是依赖注入设计的回报，
> 也是"为什么不在路由里直接 `SessionLocal()`"的最有力论据。

## 4. 数据库测试的三种策略

```python
# ── 策略 A：内存 SQLite（最快，但与生产 SQL 方言有差异）
engine = create_async_engine("sqlite+aiosqlite:///:memory:")

# ── 策略 B：真实 PostgreSQL + 每个测试一个事务并回滚（推荐 ★）
@pytest.fixture
async def db_session():
    conn = await engine.connect()
    trans = await conn.begin()
    session = AsyncSession(bind=conn, expire_on_commit=False)
    nested = await conn.begin_nested()          # SAVEPOINT

    yield session

    await session.close()
    await trans.rollback()                       # 整个测试的写入全部回滚
    await conn.close()

# ── 策略 C：testcontainers（最真实，CI 里跑）
# pip install testcontainers[postgres]
from testcontainers.postgres import PostgresContainer

@pytest.fixture(scope="session")
def pg():
    with PostgresContainer("postgres:16") as p:
        yield p.get_connection_url()
```

**推荐组合**：单元测试用 mock/内存库跑得飞快，集成测试用 testcontainers 起真实
Postgres + 事务回滚保证隔离。

## 5. conftest.py 全景示例

```python
# tests/conftest.py
import asyncio, pytest, pytest_asyncio
from httpx import AsyncClient, ASGITransport
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession

from app.main import create_app
from app.db import Base
from app.deps import get_db

TEST_DB = "postgresql+asyncpg://test:test@localhost/test_db"

@pytest.fixture(scope="session")
def event_loop():                       # pytest-asyncio 旧版需要；新版可省
    loop = asyncio.new_event_loop()
    yield loop
    loop.close()

@pytest_asyncio.fixture(scope="session")
async def engine():
    eng = create_async_engine(TEST_DB, poolclass=NullPool)
    async with eng.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield eng
    async with eng.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
    await eng.dispose()

@pytest_asyncio.fixture
async def db(engine):
    conn = await engine.connect()
    trans = await conn.begin()
    Session = async_sessionmaker(bind=conn, expire_on_commit=False)
    async with Session() as s:
        yield s
    await trans.rollback()
    await conn.close()

@pytest_asyncio.fixture
async def client(db):
    app = create_app()
    app.dependency_overrides[get_db] = lambda: db
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://t") as c:
        yield c

# 工厂 fixture：造测试数据
@pytest_asyncio.fixture
async def user_factory(db):
    async def _make(**kw):
        u = User(email=kw.get("email", "a@b.c"), name=kw.get("name", "n"))
        db.add(u); await db.flush()
        return u
    return _make
```

## 6. Mock 外部依赖

```python
# ① HTTP 调用：respx（专为 httpx 设计）
import respx, httpx

@respx.mock
async def test_external():
    respx.get("https://api.stripe.com/charge").mock(
        return_value=httpx.Response(200, json={"id": "ch_1"})
    )
    r = await client.post("/pay", json={...})
    assert r.status_code == 200

# ② 通用 mock
from unittest.mock import AsyncMock, patch

@patch("app.services.email.send", new_callable=AsyncMock)
async def test_register(mock_send, client):
    await client.post("/register", json={...})
    mock_send.assert_awaited_once()

# ③ 时间
from freezegun import freeze_time
@freeze_time("2026-01-01")
def test_expiry(): ...

# ④ Redis
import fakeredis.aioredis
app.dependency_overrides[get_redis] = lambda: fakeredis.aioredis.FakeRedis()
```

## 7. 测试分层

| 层次 | 测什么 | 工具 | 速度 |
|---|---|---|---|
| **单元** | service/工具函数的纯逻辑 | pytest + mock | 毫秒 |
| **集成** | 路由 → service → 真实 DB | TestClient + testcontainers | 百毫秒 |
| **契约** | OpenAPI schema 没被意外破坏 | schemathesis | 秒 |
| **端到端** | 真实启动 + 真实依赖 | docker-compose + pytest | 分钟 |

**契约测试（对前端团队很有价值）**：

```bash
pip install schemathesis
schemathesis run http://localhost:8000/openapi.json --checks all
# 自动根据 OpenAPI 生成用例，检查响应是否符合声明的 schema
```

**快照 OpenAPI 防止意外破坏 API**：

```python
def test_openapi_snapshot():
    schema = client.get("/openapi.json").json()
    expected = json.loads(Path("tests/openapi.snapshot.json").read_text())
    assert schema == expected      # 变更时要显式更新快照 → 强制 review API 变更
```

## 8. 覆盖率与 CI

```bash
pytest --cov=app --cov-report=term-missing --cov-report=xml --cov-fail-under=80
pytest -x -q                      # 首次失败即停，安静模式
pytest -k "user and not slow"     # 按名字筛
pytest -m integration             # 按 marker 筛
pytest -n auto                    # pytest-xdist 并行（注意 DB 隔离）
pytest --lf                       # 只跑上次失败的
```

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
asyncio_mode = "auto"
addopts = "-q --strict-markers"
markers = ["slow: 慢测试", "integration: 需要外部依赖"]

[tool.coverage.run]
source = ["src/app"]
omit = ["*/migrations/*", "*/tests/*"]
```

## 9. 常见问题

| 问题 | 解法 |
|---|---|
| `dependency_overrides` 泄漏到其它测试 | fixture 里 `yield` 后 `clear()` |
| 事件循环冲突（`Event loop is closed`） | 统一 `asyncio_mode="auto"`；engine 用 `NullPool` |
| 测试之间数据污染 | 事务回滚 fixture，或每测试重建 schema |
| lifespan 没被触发 | 用 `with TestClient(app)` 而不是裸 `TestClient(app)` |
| 测试慢 | 单元测试 mock 掉 IO；集成测试用 session 级容器 |
| 时间相关不稳定 | freezegun / 注入 clock 依赖 |

## 相关

- [[web/fastapi-di]] —— `dependency_overrides` 的原理
- [[web/fastapi-architecture]] —— 分层带来的可测试性
- [[engineering/testing]] —— pytest 全面用法
- [[web/fastapi-async-db]] —— 测试用的 session 配置
- [[interview/question-bank-web]] —— 测试相关面试题
