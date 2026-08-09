---
title: "字节码与执行流程"
date: 2026-08-07
tags: [字节码, dis, 编译, pyc, 求值循环, 帧, JIT]
sources: ["python-cheatsheet.md"]
---

# 字节码与执行流程

「Python 是解释型语言吗？」——严格说 **Python 先编译成字节码，再由虚拟机解释执行**，
和 Java 的模型类似，只是编译这一步默认对用户透明。能答清这条流水线，面试印象分很高。

## 1. 完整流水线

```text
源码 .py
  │ ① 词法分析（tokenize）
  ▼ tokens
  │ ② 语法分析（PEG parser，3.9+ 替换了原来的 LL(1)）
  ▼ AST（抽象语法树）
  │ ③ 编译（symtable → 控制流图 → 优化）
  ▼ 字节码 bytecode（存在 code object 里）
  │ ④ 求值循环 CEVAL（ceval.c 的巨型 switch / computed goto）
  ▼ 执行
```

每一步都能在 Python 层观察：

```python
import ast, dis, symtable

src = "x = a + 1"
print(ast.dump(ast.parse(src), indent=2))     # ② AST
code = compile(src, "<s>", "exec")            # ③ 编译
dis.dis(code)                                 #    字节码
exec(code)                                    # ④ 执行
```

## 2. Code Object

编译产物是 **code object**，函数对象持有它：

```python
def f(a, b=1):
    x = a + b
    return x

c = f.__code__
c.co_name            #=> 'f'
c.co_varnames        #=> ('a', 'b', 'x')      局部变量名（含参数）
c.co_argcount        #=> 2
c.co_consts          #=> (None, ...)          常量池
c.co_names           #=> ()                   全局/属性名
c.co_freevars        #=> ()                   自由变量（闭包）
c.co_cellvars        #=> ()                   被内层捕获的本地变量
c.co_code            #=> b'...'               原始字节码
c.co_flags           # 标志位（是否生成器、是否协程等）
```

**函数对象 = code object + 默认值 + 闭包 cell + 全局命名空间引用**：

```python
f.__code__        # 代码
f.__defaults__    # 默认参数
f.__closure__     # 闭包 cell
f.__globals__     # 所属模块的 globals（这就是"函数记得自己在哪个模块"的原理）
```

## 3. 读懂 `dis` 输出

```python
import dis

def f(a, b):
    x = a + b
    return x * 2

dis.dis(f)
```

```text
  2           LOAD_FAST                0 (a)      ← 局部变量，数组索引
              LOAD_FAST                1 (b)
              BINARY_OP                0 (+)      ← 3.11 起合并了各种二元运算
              STORE_FAST               2 (x)

  3           LOAD_FAST                2 (x)
              LOAD_CONST               1 (2)      ← 常量池
              BINARY_OP                5 (*)
              RETURN_VALUE
```

**CPython 是栈式虚拟机**：指令从栈上取操作数、把结果压回栈。

必须认识的指令：

| 指令 | 含义 |
|---|---|
| `LOAD_FAST` / `STORE_FAST` | 局部变量（**数组索引，最快**） |
| `LOAD_GLOBAL` | 全局/内建（查 dict，较慢；3.11+ 有内联缓存） |
| `LOAD_DEREF` / `STORE_DEREF` | 闭包 cell 变量 |
| `LOAD_CONST` | 常量池 |
| `LOAD_ATTR` / `STORE_ATTR` | 属性访问 |
| `LOAD_METHOD` + `CALL` | 方法调用（避免创建绑定方法对象的优化） |
| `BINARY_OP` / `COMPARE_OP` | 运算 |
| `POP_JUMP_IF_FALSE` | 条件跳转 |
| `FOR_ITER` / `GET_ITER` | 循环 |
| `MAKE_FUNCTION` | 创建函数对象 |
| `RETURN_VALUE` / `YIELD_VALUE` | 返回 / 生成器产出 |
| `RESUME` | 3.11+ 的插桩点（JIT/profiling 挂钩） |

> ⚠️ **字节码在版本间不稳定**，3.11/3.12/3.13 变化很大（specializing adaptive interpreter）。
> 面试里说"我看过 dis 输出"就够了，不必背具体指令。

## 4. 用 `dis` 回答面试题

`dis` 是**验证语言行为**的终极工具，几个经典应用：

```python
# ① 看清 i += 1 是多条字节码（GIL 那题；注意"多条"≠"实测一定能观察到竞态"，见 gil 页）
dis.dis("i += 1")
#   LOAD_NAME i / LOAD_CONST 1 / BINARY_OP += / STORE_NAME i   ← 4 步，可被打断

# ② 证明 a is b 的常量折叠
dis.dis("x = 1000; y = 1000")     # 两个 LOAD_CONST 指向同一个常量池条目

# ③ 证明 "".join 比 += 好
dis.dis("s = a + b + c")          # 每步都创建新 str

# ④ 证明局部变量比全局快
def g():
    return len([])
dis.dis(g)                        #   LOAD_GLOBAL len   ← 字典查找

# ⑤ 看清 f-string 的编译结果
dis.dis('f"{x}"')                 #   FORMAT_VALUE，比 % 和 .format 少一次函数调用
```

## 5. `.pyc` 缓存

```text
myapp/
├── mod.py
└── __pycache__/
    └── mod.cpython-312.pyc
```

- 首次 import 时编译并缓存字节码，后续直接加载 → **只省编译时间，不提升运行速度**。
- 缓存有效性靠 **源文件 mtime + size**（默认）或**内容 hash**（PEP 552，可选，
  用于可复现构建）。
- **直接运行的脚本（`python main.py`）不生成 `.pyc`**，只有被 import 的模块才有。
- `PYTHONDONTWRITEBYTECODE=1` 或 `python -B` 禁用（Docker 镜像里常设）。
- `compileall` 可以提前编译（Docker 构建时预编译能加快容器启动）。

```bash
python -m compileall -q .
PYTHONDONTWRITEBYTECODE=1 python main.py
```

> **面试落点**：「`.pyc` 能加速程序吗？」——只加速**导入阶段**（省去编译），
> **不加速运行**（执行的字节码完全一样）。这是常见误解。

## 6. 帧（Frame）与调用栈

每次函数调用创建一个**帧对象（frame）**，保存局部变量、值栈、指令指针。

```python
import sys, inspect

def f():
    fr = sys._getframe()
    fr.f_code.co_name        #=> 'f'
    fr.f_locals              #=> 局部变量字典
    fr.f_back                #=> 调用者的帧
    fr.f_lineno
    inspect.stack()          #=> 完整调用栈
```

**递归深度限制**由帧的开销决定：

```python
sys.getrecursionlimit()      #=> 1000
sys.setrecursionlimit(10000) # ⚠️ 设太大会撞 C 栈溢出直接段错误（3.12+ 有改善）
```

> **面试落点**：「Python 有尾递归优化吗？」——**没有**，Guido 明确拒绝
> （理由：会破坏 traceback、且递归不是 Python 的核心风格）。
> 深递归应改写为循环或显式栈。

**生成器/协程的帧被保留**，这正是它们能挂起-恢复的原因：

```python
def gen():
    x = 1
    yield
    print(x)

g = gen(); next(g)
g.gi_frame.f_locals      #=> {'x': 1}   帧还活着，局部变量都在
```

见 [[language/iterators-generators]]。

## 7. 3.11+ 的性能革命

| 版本 | 关键优化 |
|---|---|
| **3.11**（Faster CPython 计划） | **自适应特化解释器（PEP 659）**：热点指令根据实际类型特化，如 `BINARY_OP` 变 `BINARY_OP_ADD_INT`；**零成本异常**（try 无异常时零开销）；**内联 Python 函数调用**（不再创建 C 栈帧）；惰性创建帧对象。整体比 3.10 快 10~60% |
| **3.12** | 更多特化、`comprehension inlining`（PEP 709，推导式不再创建函数帧）、per-interpreter GIL 的 C API |
| **3.13** | 实验性 **JIT（copy-and-patch）**、free-threading 构建、更好的 REPL |
| **3.14** | JIT 与 free-threading 成熟化、延迟注解求值（PEP 649） |

```python
# 观察特化（3.11+）
dis.dis(f, adaptive=True)      # 运行若干次后再看，会出现 _INT / _STR 后缀的特化指令
```

> **面试落点**：能说出「3.11 起的 Faster CPython 计划把解释器改成了自适应特化解释器，
> 3.13 加了 copy-and-patch JIT」——说明你在跟进语言演进，这是很稀缺的信号。

## 8. 相关工具

```bash
python -X importtime app.py        # 各模块导入耗时（定位启动慢）
python -m dis file.py              # 反汇编整个文件
python -m ast file.py              # 打印 AST（3.9+）
python -m cProfile -s cumtime app.py
py-spy top --pid 1234              # 无侵入采样剖析（生产可用）
```

## 相关

- [[internals/cpython-object-model]] —— LOAD_FAST 为什么比 LOAD_GLOBAL 快
- [[internals/gil]] —— 字节码级别的线程切换
- [[language/iterators-generators]] —— 生成器帧的挂起
- [[language/scope-closure]] —— cellvars / freevars
- [[engineering/performance]] —— 剖析与优化实战
- [[interview/question-bank-internals-concurrency]] —— 相关面试题
