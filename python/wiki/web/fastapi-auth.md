---
title: "FastAPI 认证与授权"
date: 2026-08-07
tags: [认证, 授权, JWT, OAuth2, 密码哈希, 安全]
sources: ["fastapi-doc/tutorial/security/oauth2-jwt.md"]
---

# FastAPI 认证与授权

前端工程师对 JWT、OAuth2、CORS 已经很熟——这一页补齐**服务端该怎么正确实现**，
以及面试常追问的安全细节。

## 1. 密码哈希

```python
# pip install "passlib[bcrypt]"  或更现代的 pwdlib / argon2-cffi
from passlib.context import CryptContext

pwd = CryptContext(schemes=["bcrypt"], deprecated="auto")

hashed = pwd.hash("plaintext")          # 每次结果不同（内含随机 salt）
pwd.verify("plaintext", hashed)         #=> True
pwd.needs_update(hashed)                # 算法升级时用于渐进式重哈希
```

**必须知道的**：

- **绝不明文存密码**，也不用 MD5/SHA-1/SHA-256 直接哈希（太快，易被暴力破解）。
- 用**慢哈希**：**bcrypt / argon2id / scrypt / PBKDF2**。argon2id 是当前推荐。
- 哈希函数**自带随机 salt**，无需自己加。
- 比较必须**恒定时间**（`pwd.verify` 内部已处理；自己比 token 时用 `hmac.compare_digest`）。

```python
import hmac
hmac.compare_digest(a, b)      # ✅ 防时序攻击
a == b                         # ❌ 短路比较会泄漏前缀信息
```

> ⚠️ **bcrypt 哈希很慢（故意的，约 100~300ms）**，在 `async def` 路由里直接调用
> 会阻塞事件循环！要么放在 `def` 路由里，要么 `await asyncio.to_thread(pwd.hash, pw)`。
> 这是面试里能体现 asyncio 理解深度的细节。

## 2. JWT 签发与校验

```python
# pip install pyjwt   （python-jose 已不再推荐）
import jwt
from datetime import datetime, timedelta, UTC

SECRET = settings.secret_key.get_secret_value()
ALGO = "HS256"

def create_access_token(sub: str, expires: timedelta = timedelta(minutes=15)) -> str:
    now = datetime.now(UTC)
    payload = {
        "sub": sub,                       # subject：用户标识
        "exp": now + expires,             # 过期时间（必须有！）
        "iat": now,                       # 签发时间
        "jti": str(uuid.uuid4()),         # token id（用于吊销）
        "type": "access",
        "scopes": ["read", "write"],
    }
    return jwt.encode(payload, SECRET, algorithm=ALGO)

def decode_token(token: str) -> dict:
    try:
        return jwt.decode(
            token, SECRET,
            algorithms=[ALGO],            # ★ 必须显式指定，防 alg=none 攻击
            options={"require": ["exp", "sub"]},
        )
    except jwt.ExpiredSignatureError:
        raise HTTPException(401, "token expired")
    except jwt.InvalidTokenError:
        raise HTTPException(401, "invalid token")
```

**JWT 安全清单（面试常考）**：

| 风险 | 防范 |
|---|---|
| `alg: none` 攻击 | **显式传 `algorithms=[...]`**，绝不信任 header 里的 alg |
| 对称密钥泄漏 | 密钥从环境变量读，足够长（≥32 字节随机） |
| **无法吊销** | access token 设短（15 分钟）+ refresh token + 黑名单（Redis 存 jti） |
| 敏感信息泄漏 | **JWT payload 只是 base64，不是加密**——不放密码、身份证 |
| 存储位置 | **httpOnly + Secure + SameSite Cookie** 优于 localStorage（防 XSS 窃取） |
| CSRF | 用 Cookie 存 token 时需配 SameSite=Lax/Strict 或 CSRF token |
| 重放 | 短过期 + jti + HTTPS |

> **面试落点**：「JWT 有什么缺点？」——
> **① 签发后无法主动失效**（这是最大缺点，只能靠短过期 + 黑名单，而黑名单又让它失去了"无状态"的优势）；
> ② payload 只是编码不是加密；③ token 比 session id 大得多，每个请求都带；
> ④ 密钥轮换复杂。
> 补一句：「所以很多团队在单体/内网服务里仍然用 **服务端 session + Redis**，
> JWT 更适合跨服务、跨域的场景。」这个权衡意识比背 JWT 结构值钱得多。

## 3. OAuth2 密码流 + 依赖链

```python
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/api/v1/auth/token")
# ↑ 它做两件事：① 从 Authorization: Bearer xxx 头提取 token
#              ② 在 Swagger UI 里生成 "Authorize" 按钮

@router.post("/token")
async def login(form: Annotated[OAuth2PasswordRequestForm, Depends()], db: DB):
    user = await authenticate(db, form.username, form.password)
    if not user:
        raise HTTPException(
            401, "Incorrect username or password",
            headers={"WWW-Authenticate": "Bearer"},
        )
    return {
        "access_token": create_access_token(str(user.id)),
        "refresh_token": create_refresh_token(str(user.id)),
        "token_type": "bearer",
    }

# 依赖链：token → payload → 用户 → 活跃用户 → 有权限的用户
async def get_current_user(
    token: Annotated[str, Depends(oauth2_scheme)],
    db: DB,
) -> User:
    payload = decode_token(token)
    user = await db.get(User, int(payload["sub"]))
    if user is None:
        raise HTTPException(401, "user not found")
    return user

async def get_active_user(
    user: Annotated[User, Depends(get_current_user)],
) -> User:
    if not user.is_active:
        raise HTTPException(400, "inactive user")
    return user

CurrentUser = Annotated[User, Depends(get_active_user)]

@router.get("/me", response_model=UserPublic)
async def me(user: CurrentUser):
    return user
```

## 4. 授权（RBAC / 权限）

```python
# ① 角色检查依赖工厂
def require_roles(*roles: str):
    async def checker(user: CurrentUser) -> User:
        if not set(roles) & set(user.roles):
            raise HTTPException(403, "insufficient permissions")
        return user
    return checker

@router.delete("/users/{uid}", dependencies=[Depends(require_roles("admin"))])
async def delete_user(uid: int): ...

# ② OAuth2 scopes（FastAPI 原生支持，会出现在 OpenAPI 里）
from fastapi.security import SecurityScopes
from fastapi import Security

oauth2_scheme = OAuth2PasswordBearer(
    tokenUrl="token",
    scopes={"items:read": "读取", "items:write": "写入", "admin": "管理"},
)

async def get_user_with_scopes(
    security_scopes: SecurityScopes,
    token: Annotated[str, Depends(oauth2_scheme)],
) -> User:
    payload = decode_token(token)
    token_scopes = payload.get("scopes", [])
    for scope in security_scopes.scopes:
        if scope not in token_scopes:
            raise HTTPException(
                403, f"Not enough permissions: {scope}",
                headers={"WWW-Authenticate": f'Bearer scope="{security_scopes.scope_str}"'},
            )
    return await load_user(payload["sub"])

@router.post("/items/")
async def create_item(
    user: Annotated[User, Security(get_user_with_scopes, scopes=["items:write"])],
): ...

# ③ 对象级权限（ABAC）—— 不能只靠依赖，要在业务层判断
async def get_post(post_id: int, user: CurrentUser, db: DB) -> Post:
    post = await db.get(Post, post_id)
    if post is None:
        raise HTTPException(404)
    if post.author_id != user.id and "admin" not in user.roles:
        raise HTTPException(403)     # ⚠️ 也可以返回 404 避免泄漏资源存在性
    return post
```

> **面试落点**：**认证（authentication，你是谁）** 和 **授权（authorization，你能做什么）**
> 要分清。FastAPI 里两者都用依赖实现，但授权分两级：
> **路由级**（角色/scope，用 `dependencies=`）和**对象级**（这条数据是不是你的，必须在业务层判断）。
> 只做路由级检查是 **IDOR（不安全的直接对象引用）** 漏洞的根源——OWASP Top 1。

## 5. Refresh Token 流程

```python
@router.post("/refresh")
async def refresh(refresh_token: str, db: DB, redis: Redis):
    payload = decode_token(refresh_token)
    if payload["type"] != "refresh":
        raise HTTPException(401)
    if await redis.get(f"revoked:{payload['jti']}"):      # 黑名单检查
        raise HTTPException(401, "revoked")

    # 轮转：旧的 refresh token 立即作废（检测 token 重放）
    await redis.setex(f"revoked:{payload['jti']}", 7*86400, "1")
    return {
        "access_token": create_access_token(payload["sub"]),
        "refresh_token": create_refresh_token(payload["sub"]),
    }

@router.post("/logout")
async def logout(user: CurrentUser, token: Annotated[str, Depends(oauth2_scheme)], redis: Redis):
    p = decode_token(token)
    ttl = int(p["exp"] - time.time())
    await redis.setex(f"revoked:{p['jti']}", max(ttl, 1), "1")
```

配套的 access token 校验里要加黑名单检查（代价是失去了完全无状态性——**这就是 JWT 的核心权衡**）。

## 6. 其它安全要点

```python
# ① API Key（服务间调用）
from fastapi.security import APIKeyHeader
api_key_header = APIKeyHeader(name="X-API-Key")
async def verify_api_key(key: Annotated[str, Depends(api_key_header)]):
    if not hmac.compare_digest(key, settings.api_key):
        raise HTTPException(401)

# ② 限流（防暴力破解/DoS）
# pip install slowapi
from slowapi import Limiter
limiter = Limiter(key_func=get_remote_address)
@router.post("/token")
@limiter.limit("5/minute")            # 登录接口必须限流
async def login(request: Request, ...): ...

# ③ 安全响应头
app.add_middleware(TrustedHostMiddleware, allowed_hosts=["api.example.com"])
app.add_middleware(HTTPSRedirectMiddleware)

# ④ 不要泄漏用户是否存在
# ❌ "用户不存在" / "密码错误"  → 可枚举用户
# ✅ 统一返回 "用户名或密码错误"

# ⑤ 敏感字段不进日志 / 不进响应
class UserInDB(BaseModel):
    hashed_password: str = Field(exclude=True)     # 序列化时排除
password: SecretStr                                 # repr 显示为 **********
```

## 7. 常见安全面试题速答

| 问题 | 要点 |
|---|---|
| Session vs JWT | Session 有状态、易吊销、需共享存储；JWT 无状态、跨服务友好、难吊销 |
| JWT 存哪 | httpOnly Cookie（防 XSS）> localStorage；用 Cookie 要防 CSRF |
| 怎么防 SQL 注入 | 参数绑定（ORM 默认就是），不拼字符串 |
| 怎么防 XSS | 输出转义（前端职责）+ CSP + httpOnly Cookie |
| 怎么防 CSRF | SameSite Cookie + CSRF token + 校验 Origin/Referer |
| 密码存储 | bcrypt/argon2id 慢哈希 + 自带 salt |
| 越权（IDOR） | 对象级权限校验，不能只查角色 |
| Python 特有风险 | `pickle.loads` 不可信数据 = RCE；`eval/exec`；`yaml.load` 用 `safe_load`；`subprocess(shell=True)` |

## 相关

- [[web/fastapi-di]] —— 依赖链的组织
- [[web/fastapi-core]] —— CORS 与异常处理
- [[web/fastapi-production]] —— HTTPS、密钥管理
- [[stdlib/stdlib-essentials]] —— `secrets` / `hmac` / `hashlib`
- [[interview/question-bank-web]] —— 安全面试题
- [[sources/fastapi-docs]] —— 来源：FastAPI 安全教程
