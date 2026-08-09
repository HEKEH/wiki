---
title: "FastAPI + SQLAlchemy 2.0 异步数据库"
date: 2026-08-07
tags: [SQLAlchemy, 异步, asyncpg, session, N+1, 迁移, alembic]
sources: ["fastapi-doc/tutorial/sql-databases.md"]
---

# FastAPI + SQLAlchemy 2.0 异步数据库

数据库是后端面试的**深水区**。这一页给出 2026 年的现代写法（SQLAlchemy 2.0 风格 + asyncpg），
以及必考的 session 生命周期、N+1、事务问题。

## 1. 技术选型

| 层 | 选择 | 说明 |
|---|---|---|
| 驱动 | **asyncpg**（PostgreSQL）/ `asyncmy`（MySQL） | 纯异步，最快 |
| ORM | **SQLAlchemy 2.0**（async）/ SQLModel / Tortoise | SQLAlchemy 是事实标准 |
| 迁移 | **Alembic** | SQLAlchemy 官方配套 |
| 连接池 | SQLAlchemy 内置 / **pgbouncer**（生产） | |

> ⚠️ **同步驱动不能用在 `async def` 路由里**（`psycopg2`、`pymysql` 会阻塞事件循环）。
> 要么全异步，要么路由用 `def` 让 FastAPI 放线程池。见 [[web/fastapi-core]]。

## 2. 引擎与会话工厂

```python
# app/db.py
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

engine = create_async_engine(
    "postgresql+asyncpg://user:pw@host/db",
    echo=False,                 # True 会打印所有 SQL，调试用
    pool_size=20,               # 常驻连接数
    max_overflow=10,            # 峰值可额外创建
    pool_timeout=30,
    pool_recycle=1800,          # ★ 30 分钟回收，避免被数据库/中间件断掉的死连接
    pool_pre_ping=True,         # ★ 用前 ping 一下，彻底避免 "server closed the connection"
)

AsyncSessionLocal = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False,     # ★ 关键！见下文
    autoflush=False,
)

class Base(DeclarativeBase):
    pass
```

> **`expire_on_commit=False` 为什么关键**：默认 `True` 时，`commit()` 后所有对象属性被标记为过期，
> 下次访问会**重新发 SQL 查询**——在 async 场景里，如果此时 session 已关闭就会抛
> `MissingGreenlet` / `DetachedInstanceError`。这是 FastAPI + SQLAlchemy 最常见的报错之一。

## 3. 模型定义（2.0 风格）

```python
# app/models.py
from datetime import datetime
from sqlalchemy import String, ForeignKey, func, Index
from sqlalchemy.orm import Mapped, mapped_column, relationship

class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    name: Mapped[str] = mapped_column(String(50))
    is_active: Mapped[bool] = mapped_column(default=True)
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())

    posts: Mapped[list["Post"]] = relationship(back_populates="author", lazy="raise")

    __table_args__ = (Index("ix_users_name_active", "name", "is_active"),)

class Post(Base):
    __tablename__ = "posts"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str]
    author_id: Mapped[int] = mapped_column(ForeignKey("users.id", ondelete="CASCADE"))
    author: Mapped[User] = relationship(back_populates="posts")
```

**2.0 风格的核心变化**：用 `Mapped[T]` + `mapped_column()` 声明，**类型注解直接映射列类型**，
mypy 能完整推导。（旧的 `Column(Integer, ...)` 风格仍可用但不推荐。）

> **`lazy="raise"` 是防 N+1 的神器**：任何未显式预加载的关系访问都会直接抛异常，
> 强制你写出正确的查询。生产项目强烈建议加上。

## 4. Session 依赖（请求级生命周期）★

```python
# app/deps.py
from typing import Annotated
from fastapi import Depends

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionLocal() as session:
        try:
            yield session
            await session.commit()        # 请求成功 → 提交
        except Exception:
            await session.rollback()      # 出错 → 回滚
            raise
        # async with 退出时自动 close()

DB = Annotated[AsyncSession, Depends(get_db)]
```

> **面试落点**：「session 的作用域应该是什么？」
> 标准答案：**一个请求一个 session（request-scoped）**，通过依赖注入管理，
> 请求结束时提交或回滚并关闭。
> **绝不能用全局 session**（不是线程/协程安全的，且事务会互相污染）。
> 反模式还有：在依赖里 `yield` 之后不 commit 而在每个路由里手动 commit（容易漏）。

## 5. 查询（2.0 的 `select()` 风格）

```python
from sqlalchemy import select, update, delete, func
from sqlalchemy.orm import selectinload, joinedload

# 单条
stmt = select(User).where(User.email == email)
user = (await db.execute(stmt)).scalar_one_or_none()

# 主键直取（走 identity map 缓存）
user = await db.get(User, user_id)

# 列表 + 分页 + 排序
stmt = (
    select(User)
    .where(User.is_active.is_(True))
    .order_by(User.created_at.desc())
    .offset(skip).limit(limit)
)
users = (await db.execute(stmt)).scalars().all()

# 聚合
total = await db.scalar(select(func.count()).select_from(User))

# 只取部分列（更快，返回 Row 而非实体）
rows = (await db.execute(select(User.id, User.name))).all()

# 写
db.add(User(email=..., name=...))
await db.flush()               # 发 SQL 但不提交（拿自增 id）
await db.refresh(user)         # 从库里刷新对象

await db.execute(update(User).where(User.id == uid).values(name="x"))
await db.execute(delete(User).where(User.id == uid))

# 原生 SQL（复杂查询时）
from sqlalchemy import text
rows = (await db.execute(text("SELECT * FROM users WHERE id = :id"), {"id": 1})).all()
```

> ⚠️ **永远用参数绑定，不要拼接字符串**——`text(f"... WHERE id = {uid}")` 是 SQL 注入。

## 6. N+1 问题（必考）

```python
# ❌ N+1：1 次查用户 + N 次查每个用户的 posts
users = (await db.execute(select(User))).scalars().all()
for u in users:
    print(u.posts)          # 每次都发一条 SQL！（同步 SQLAlchemy 会；async 下会直接报错）

# ✅ 方案 1：selectinload —— 额外发 1 条 IN 查询（一对多首选）
stmt = select(User).options(selectinload(User.posts))
# SQL: SELECT * FROM users;  SELECT * FROM posts WHERE author_id IN (...)

# ✅ 方案 2：joinedload —— LEFT JOIN 一次拿完（多对一 / 一对一首选）
stmt = select(Post).options(joinedload(Post.author))

# ✅ 方案 3：只查需要的列
stmt = select(User.id, User.name)

# ✅ 方案 4：显式 join + 聚合
stmt = (
    select(User, func.count(Post.id).label("post_count"))
    .outerjoin(Post).group_by(User.id)
)
```

| 策略 | SQL 形态 | 适用 |
|---|---|---|
| `selectinload` | 2 条查询，第二条用 `IN` | **一对多**（避免行数膨胀） |
| `joinedload` | 1 条 LEFT JOIN | **多对一 / 一对一** |
| `subqueryload` | 2 条，第二条用子查询 | 老方案，一般用 selectinload |
| `lazy="raise"` | 直接报错 | **开发期强制发现 N+1** ✅ |

> **面试落点**：能说清 **`selectinload` vs `joinedload` 的选择依据**——
> 一对多用 `selectinload`（joinedload 会让主表行数被乘开，传输冗余数据）；
> 多对一用 `joinedload`（一次 JOIN 最省往返）。这是有实战经验的信号。

## 7. 事务

```python
# ① 依赖里的隐式事务（推荐，见上面的 get_db）

# ② 显式事务块
async with db.begin():          # 进入时开启，退出时自动 commit / 异常时 rollback
    db.add(a)
    await db.execute(...)

# ③ 嵌套（SAVEPOINT）
async with db.begin_nested():
    ...                          # 失败只回滚到保存点

# ④ 悲观锁
stmt = select(Account).where(Account.id == aid).with_for_update()

# ⑤ 乐观锁（版本号）
class Account(Base):
    version_id: Mapped[int] = mapped_column(nullable=False)
    __mapper_args__ = {"version_id_col": version_id}
```

**跨服务的一致性**：单库事务解决不了"扣款 + 发消息"，
需要**事务性发件箱（transactional outbox）**或 Saga。这是系统设计面试的常见追问。

## 8. Alembic 迁移

```bash
pip install alembic
alembic init -t async migrations          # 异步模板
alembic revision --autogenerate -m "add users table"
alembic upgrade head
alembic downgrade -1
alembic history / current
```

```python
# migrations/env.py 里让 autogenerate 认识你的模型
from app.models import Base
target_metadata = Base.metadata
```

> **生产铁律**：`Base.metadata.create_all()` **只能用于测试/原型**，
> 生产环境一律用 Alembic 版本化迁移。迁移要**向后兼容**（先加列 → 双写 → 迁数据 → 删旧列），
> 否则滚动发布期间新旧代码会同时跑。

## 9. 完整 CRUD 分层示例

```python
# app/repositories/user.py —— 数据访问层
class UserRepo:
    def __init__(self, db: AsyncSession):
        self.db = db

    async def get(self, uid: int) -> User | None:
        return await self.db.get(User, uid)

    async def get_by_email(self, email: str) -> User | None:
        stmt = select(User).where(User.email == email)
        return (await self.db.execute(stmt)).scalar_one_or_none()

    async def list(self, skip=0, limit=100) -> list[User]:
        stmt = select(User).offset(skip).limit(limit).order_by(User.id)
        return list((await self.db.execute(stmt)).scalars())

    async def create(self, data: UserCreate) -> User:
        user = User(**data.model_dump())
        self.db.add(user)
        await self.db.flush()          # 拿到 id，但不提交（提交交给依赖统一做）
        return user

# app/services/user.py —— 业务逻辑层
class UserService:
    def __init__(self, repo: UserRepo):
        self.repo = repo

    async def register(self, data: UserCreate) -> User:
        if await self.repo.get_by_email(data.email):
            raise EmailTaken(data.email)
        data.password = hash_password(data.password)
        return await self.repo.create(data)

# app/api/users.py —— 路由层
@router.post("/", response_model=UserPublic, status_code=201)
async def register(data: UserCreate, db: DB):
    try:
        return await UserService(UserRepo(db)).register(data)
    except EmailTaken as e:
        raise HTTPException(409, str(e))
```

**分层的意义**：业务逻辑不依赖 FastAPI，可以在 CLI、Celery worker、测试里复用。

## 10. 常见错误速查

| 报错 | 原因 | 解法 |
|---|---|---|
| `MissingGreenlet` | 在 async 上下文外触发了懒加载 | `expire_on_commit=False` + 显式 `selectinload` |
| `DetachedInstanceError` | session 关闭后访问对象属性 | 在 session 内转成 Pydantic 模型再返回 |
| `object is already attached to session` | 同一对象被两个 session 管理 | 每请求一个 session，不跨请求传对象 |
| `QueuePool limit ... overflow` | 连接池耗尽 | 调大池 / 检查是否有未关闭的 session / 加 pgbouncer |
| `IntegrityError` | 唯一约束冲突 | 捕获并转成 409，或用 upsert |
| 事务一直不提交 | 忘了 commit / 依赖里没写 | 统一在依赖里 commit |

## 相关

- [[web/fastapi-di]] —— session 依赖的生命周期
- [[web/pydantic]] —— ORM 对象 → Pydantic 模型（`from_attributes`）
- [[web/fastapi-architecture]] —— 分层与仓储模式
- [[web/fastapi-production]] —— 连接池与容量规划
- [[concurrency/asyncio-patterns]] —— 异步上下文管理器
- [[interview/question-bank-web]] —— 数据库面试题
- [[sources/fastapi-docs]] —— 来源：FastAPI SQL 教程
