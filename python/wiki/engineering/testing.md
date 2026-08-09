---
title: "pytest 测试实践"
date: 2026-08-07
tags: [pytest, fixture, mock, 参数化, 覆盖率, TDD]
sources: []
---

# pytest 测试实践

Python 生态里 **pytest 是事实标准**（标准库的 `unittest` 只在遗留项目里见）。
面试问"你怎么写测试"时，答 pytest + fixture + 参数化 + mock 是标配。

> 前端类比：pytest ≈ vitest/jest。`fixture` ≈ `beforeEach` + 依赖注入的结合体，
> 但比它强大得多（有作用域、可组合、可参数化）。

## 1. 基础

```python
# tests/test_calc.py —— 文件名 test_*.py，函数名 test_*
def test_add():
    assert add(1, 2) == 3          # ✅ 直接用 assert，pytest 会重写它给出详细差异

def test_raises():
    with pytest.raises(ValueError, match="invalid"):
        parse("x")

def test_approx():
    assert 0.1 + 0.2 == pytest.approx(0.3)     # 浮点比较

def test_warns():
    with pytest.warns(DeprecationWarning):
        old_api()
```

pytest 的 **assert 重写**是它相对 unittest 的最大优势：

```text
E       assert {'a': 1, 'b': 3} == {'a': 1, 'b': 2}
E         Omitting 1 identical items, use -vv to show
E         Differing items:
E         {'b': 3} != {'b': 2}
```

```bash
pytest                        # 跑全部
pytest tests/test_calc.py::test_add    # 跑单个
pytest -k "user and not slow" # 按名字筛
pytest -m integration         # 按 marker 筛
pytest -x                     # 首次失败即停
pytest --lf                   # 只跑上次失败的
pytest -vv -s                 # 详细输出 + 不吞 print
pytest --pdb                  # 失败时进调试器
pytest -n auto                # 并行（pytest-xdist）
pytest --durations=10         # 最慢的 10 个测试
```

## 2. Fixture

```python
import pytest

@pytest.fixture
def db():                         # 名字就是注入的参数名
    conn = create_connection()
    yield conn                    # yield 前 = setup，后 = teardown
    conn.close()

def test_query(db):               # 声明即注入
    assert db.query("SELECT 1")
```

### 作用域

```python
@pytest.fixture(scope="function")  # 默认：每个测试函数一次
@pytest.fixture(scope="class")
@pytest.fixture(scope="module")    # 每个测试文件一次
@pytest.fixture(scope="package")
@pytest.fixture(scope="session")   # 整个测试会话一次（起 Docker 容器用这个）
```

> **权衡**：`session` 快但有状态污染风险；`function` 干净但慢。
> 常见做法：**昂贵资源用 session（DB 容器、引擎），状态隔离用 function（事务回滚）**。

### 组合与自动使用

```python
@pytest.fixture
def user(db):                     # fixture 可以依赖 fixture
    return db.create_user()

@pytest.fixture(autouse=True)     # 不用声明也自动生效（谨慎用）
def reset_cache():
    cache.clear()
    yield
    cache.clear()

# conftest.py 里的 fixture 对同目录及子目录的所有测试可见（无需 import）
```

### 工厂 fixture（最实用的模式）

```python
@pytest.fixture
def make_user(db):
    created = []
    def _make(**kwargs):
        u = User(name=kwargs.get("name", "test"), **kwargs)
        db.add(u); db.flush()
        created.append(u)
        return u
    yield _make
    for u in created:             # 统一清理
        db.delete(u)

def test_x(make_user):
    a = make_user(name="alice")
    b = make_user(name="bob", role="admin")
```

### 内置 fixture

```python
def test_files(tmp_path):              # pathlib.Path，自动清理的临时目录
    (tmp_path / "x.txt").write_text("hi")

def test_env(monkeypatch):
    monkeypatch.setenv("API_KEY", "test")
    monkeypatch.setattr("app.service.client", FakeClient())
    monkeypatch.delitem(cfg, "debug")
    monkeypatch.chdir(tmp_path)        # 自动在测试结束后还原 ★

def test_output(capsys):
    print("hello")
    assert capsys.readouterr().out == "hello\n"

def test_logs(caplog):
    with caplog.at_level(logging.WARNING):
        do_thing()
    assert "failed" in caplog.text

def test_meta(request):
    request.node.name / request.config
```

## 3. 参数化

```python
@pytest.mark.parametrize("a,b,expected", [
    (1, 2, 3),
    (0, 0, 0),
    (-1, 1, 0),
    pytest.param(1, "x", None, marks=pytest.mark.xfail(raises=TypeError)),
])
def test_add(a, b, expected):
    assert add(a, b) == expected

# 多个 parametrize 会做笛卡尔积
@pytest.mark.parametrize("db", ["sqlite", "postgres"])
@pytest.mark.parametrize("mode", ["fast", "safe"])
def test_matrix(db, mode): ...        # 4 个用例

# 参数化 fixture
@pytest.fixture(params=["sqlite", "postgres"])
def engine(request):
    return create_engine(request.param)   # 所有用到 engine 的测试自动跑两遍

# 给用例起可读的名字
@pytest.mark.parametrize("case", CASES, ids=lambda c: c.name)
```

> **面试落点**：参数化是"用一份代码覆盖多种输入"的标准手段，
> 比在测试里写 for 循环好——**每个参数是独立用例，失败时能精确定位是哪组数据**。

## 4. Mock

```python
from unittest.mock import Mock, MagicMock, AsyncMock, patch, call

# ① 直接构造
m = Mock(return_value=42)
m(1, 2)                       #=> 42
m.assert_called_once_with(1, 2)
m.call_count / m.call_args / m.call_args_list

m = Mock(side_effect=[1, 2, 3])          # 依次返回
m = Mock(side_effect=ConnectionError)    # 抛异常
m = Mock(side_effect=lambda x: x * 2)    # 动态

# ② patch —— 替换目标位置的对象
@patch("app.services.email.smtp_send")   # ★ patch 的是【使用处】，不是定义处！
def test_register(mock_send):
    register(...)
    mock_send.assert_called_once()

with patch.object(Service, "fetch", return_value={"a": 1}):
    ...
with patch.dict(os.environ, {"KEY": "v"}):
    ...

# ③ 异步
mock = AsyncMock(return_value=1)
await mock()
mock.assert_awaited_once()

# ④ 严格 mock（防止拼错方法名）
from unittest.mock import create_autospec
mock_db = create_autospec(Database, spec_set=True)
mock_db.qeury()               # ❌ AttributeError，拼写错误立刻暴露
```

> **`patch` 的头号陷阱**：`patch("模块.名字")` 打的是**名字被查找的地方**。
> 如果 `app/service.py` 里写了 `from app.email import send`，
> 那要 patch 的是 `app.service.send` 而不是 `app.email.send`。
> 这是"Where to patch"，官方文档专门有一节讲它，面试也常问。

**什么该 mock、什么不该**：

| 该 mock | 不该 mock |
|---|---|
| 外部 HTTP 调用 | 你自己的业务逻辑 |
| 支付/短信等第三方 | 数据结构、纯函数 |
| 当前时间、随机数 | 数据库（用真实测试库更可靠） |
| 慢/不稳定的依赖 | 被测对象本身 |

**过度 mock 的代价**：测试通过但生产失败（mock 的行为和真实的不一致）。
优先用**真实依赖 + 隔离环境**（testcontainers），mock 留给真正的外部边界。

## 5. 组织与标记

```python
# pyproject.toml
[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-q --strict-markers --strict-config"
markers = [
    "slow: 运行较慢的测试",
    "integration: 需要外部依赖",
]
filterwarnings = ["error", "ignore::DeprecationWarning:third_party.*"]
```

```python
@pytest.mark.slow
@pytest.mark.integration
@pytest.mark.skip(reason="待实现")
@pytest.mark.skipif(sys.platform == "win32", reason="仅 Unix")
@pytest.mark.xfail(reason="已知 bug #123", strict=True)
def test_x(): ...
```

```bash
pytest -m "not slow"                  # 本地开发跳过慢测试
pytest -m "integration"               # CI 单独跑
```

## 6. 覆盖率

```bash
pytest --cov=src/app --cov-report=term-missing --cov-report=html --cov-fail-under=80
```

```toml
[tool.coverage.run]
source = ["src/app"]
branch = true                          # ★ 分支覆盖率比行覆盖率有意义得多
omit = ["*/migrations/*", "*/__init__.py"]

[tool.coverage.report]
exclude_lines = [
    "pragma: no cover",
    "if TYPE_CHECKING:",
    "raise NotImplementedError",
    "if __name__ == .__main__.:",
]
```

> **面试落点**：**覆盖率是"没测到什么"的指标，不是"测得好不好"的指标**。
> 100% 覆盖率也可能一个断言都没有。更该关注的是**核心业务路径 + 边界条件 + 错误分支**
> 是否被覆盖。能说出这个区分是成熟度的体现。

## 7. TDD 与测试设计

```text
红 → 绿 → 重构
写一个失败的测试 → 写最小实现让它通过 → 在测试保护下重构
```

**好测试的特征（FIRST）**：

- **F**ast —— 单元测试应该毫秒级
- **I**ndependent —— 顺序无关，可并行
- **R**epeatable —— 不依赖时间、网络、随机
- **S**elf-validating —— 只有通过/失败，不用人看输出
- **T**imely —— 和代码一起写

**AAA 结构**：

```python
def test_transfer():
    # Arrange
    acc_a = make_account(balance=100)
    acc_b = make_account(balance=0)
    # Act
    transfer(acc_a, acc_b, 30)
    # Assert
    assert acc_a.balance == 70
    assert acc_b.balance == 30
```

## 8. 其它有用的库

| 库 | 用途 |
|---|---|
| `pytest-asyncio` / `anyio` | 异步测试 |
| `pytest-cov` | 覆盖率 |
| `pytest-xdist` | 并行执行 |
| `pytest-mock` | `mocker` fixture（比 `patch` 装饰器更灵活） |
| `freezegun` | 冻结时间 |
| `factory_boy` / `polyfactory` | 测试数据工厂 |
| `hypothesis` | **基于属性的测试**（自动生成边界用例）★ |
| `respx` / `responses` | mock HTTP |
| `testcontainers` | 真实的 DB/Redis 容器 |
| `schemathesis` | 基于 OpenAPI 的契约测试 |

**hypothesis 值得单独提**——它能自动找出你想不到的边界：

```python
from hypothesis import given, strategies as st

@given(st.lists(st.integers()))
def test_sort_idempotent(xs):
    assert sorted(sorted(xs)) == sorted(xs)      # 自动生成上百组输入，包括空列表、极值
```

> 面试里提到 hypothesis（property-based testing）通常会让面试官眼前一亮。

## 相关

- [[web/fastapi-testing]] —— FastAPI 专用的测试模式
- [[engineering/tooling-quality]] —— CI 中的测试与检查
- [[engineering/packaging-envs]] —— 测试依赖管理
- [[interview/question-bank-web]] —— 测试相关面试题
