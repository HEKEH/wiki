---
title: "CPython 对象模型与引用计数"
date: 2026-08-07
tags: [CPython, PyObject, 引用计数, 类型对象, 缓存, 不朽对象, 内部实现]
sources: ["cpython-doc/datamodel.rst", "interview-python-cn.md"]
---

# CPython 对象模型与引用计数

## 1. 一切皆 PyObject

CPython 里**每个 Python 对象**都是一个以 `PyObject` 头开始的 C 结构体：

```c
typedef struct _object {
    Py_ssize_t ob_refcnt;        // 引用计数
    PyTypeObject *ob_type;       // 指向类型对象
} PyObject;

typedef struct {
    PyObject ob_base;
    Py_ssize_t ob_size;          // 变长对象的元素个数（list/tuple/str/int）
} PyVarObject;
```

推论（面试常考的"为什么 Python 慢"）：

- **连一个小整数都是堆上的对象**，有 refcount + type 指针的开销（`sys.getsizeof(1)` = 28 字节）。
- `list[int]` 存的是**指针数组**，不是连续的整数——所以 CPU 缓存不友好，
  这正是 NumPy 用连续 C 数组能快 10~100 倍的原因。
- 每次算术运算都要**装箱/拆箱 + 类型分派**。

```python
import sys
sys.getsizeof(1)            #=> 28      （int 对象本体）
sys.getsizeof(10**100)      #=> 72      任意精度，位数越多越大
sys.getsizeof([])           #=> 56
sys.getsizeof([1,2,3])      #=> 88      = 56 + 3*8（指针）
sys.getsizeof("")           #=> 41
sys.getsizeof("a")          #=> 42      紧凑 ASCII 表示，每字符 1 字节
sys.getsizeof("中")          #=> 60      非 ASCII → 切换到更宽的表示（PEP 393）
# 以上数值为 CPython 3.13 / 64 位实测，随版本会变——记住量级和相对关系即可
```

`sys.getsizeof` **只算对象本体，不递归**——`[big_obj]` 的大小仍是 64 左右。

## 2. PyObject / PyVarObject / PyTypeObject 三者的关系

三者不在同一个维度上，它们之间有**两条不同的关系**。

**关系一：内存布局的嵌套**（C 里模拟单继承）。每一层把上一层原封不动放在自己内存的最开头，
所以地址可以逐级向上强转：

```
PyLongObject  ┌──────────┬──────────┬──────────┬────────────┐
（一个 int）    │ ob_refcnt│ ob_type  │ ob_size  │ digits[]   │
              └──────────┴──────────┴──────────┴────────────┘
              └── 当作 PyObject* ───┘
              └───────  当作 PyVarObject*  ───────┘
```

由此推出一条关键结论：**`PyTypeObject` 自己也以 `PyObject_VAR_HEAD` 开头**，因而它本身就是一个
`PyVarObject`/`PyObject` —— 这就是「类也是对象」在 C 层的落地：`int` 这个类自己有引用计数、
自己有 `ob_type`，可以放进 list、当 dict 的 key。

`PyVarObject` 不是必经之路，它只是给**变长**对象用的分支：

| 类型 | 头 | 长度存哪 |
|---|---|---|
| `int` / `list` / `tuple` / `bytes` | `PyVarObject` | `ob_size` |
| `float` / `dict` / `set` / 普通实例 | `PyObject` | 自己的字段里，或压根没有 |

`ob_size` 的语义随类型而变：`list`/`tuple` 是元素个数，`bytes` 是字节数，`int` 是 30-bit 数字位的
个数（用负号表示负数）。`PyTypeObject` 虽然带了 `ob_size`，静态类型里恒为 0，堆类型（`type()`
动态造的类）里是内部用途——别去读它。

**关系二：实例 → 类型的指向**。`ob_type` 一定指向某个 `PyTypeObject`，即 `type(x)` 的结果；
链条终点 `PyType_Type`（也就是 `type`）的 `ob_type` 指向自己：

```
  x = [1, 2]

  PyListObject (x)          PyTypeObject (list)       PyTypeObject (type)
  ┌──────────┐              ┌──────────┐              ┌──────────┐
  │ ob_refcnt│              │ ob_refcnt│              │ ob_refcnt│
  │ ob_type ─┼─────────────>│ ob_type ─┼─────────────>│ ob_type ─┼──┐
  │ ob_size=2│              │ ob_size  │              │ ob_size  │  │
  │ ob_item ─┼─> [ptr, ptr] │ "list"   │              │ "type"   │<─┘
  └──────────┘              │ tp_dealloc│             └──────────┘
                            └──────────┘              PyType_Type 自指
```

```python
type([1, 2])            #=> <class 'list'>    读 PyListObject.ob_type
type(list)              #=> <class 'type'>    读 PyTypeObject(list).ob_type
type(type) is type      #=> True              自指，递归终止
len([1, 2])             #=> 2                 多数情况下就是直接读 ob_size
```

配套宏也体现这个分层：`Py_REFCNT(op)`、`Py_TYPE(op)` 对任何对象都能用（走 `PyObject` 层），
`Py_SIZE(op)` 只对变长对象有意义（走 `PyVarObject` 层）。

> **面试落点**：`PyObject` 是所有对象的公共头（refcnt + type）；`PyVarObject` 是它加一个
> `ob_size`，供变长对象用，是可选的中间层；`PyTypeObject` 是类型对象的完整布局。关键是
> `PyTypeObject` 同时站在两条关系上——它既是 `ob_type` 指向的目标（决定别人的行为），
> 又本身是个 `PyObject`（所以类自己也是对象，`type(type) is type`）。

## 3. 类型对象与 slot

`ob_type` 指向 `PyTypeObject`，里面是一大堆**函数指针槽位（slot）**：

```c
struct _typeobject {
    const char *tp_name;
    Py_ssize_t tp_basicsize;
    destructor tp_dealloc;
    reprfunc tp_repr;
    PyNumberMethods *tp_as_number;      // __add__ __sub__ ...
    PySequenceMethods *tp_as_sequence;  // __len__ __getitem__ ...
    PyMappingMethods *tp_as_mapping;
    hashfunc tp_hash;                   // __hash__
    ternaryfunc tp_call;                // __call__
    getattrofunc tp_getattro;           // __getattribute__
    ...
};
```

**这解释了 [[language/data-model]] 里的核心结论**：`len(x)` 走 `tp_as_sequence->sq_length`，
是一次 C 函数指针调用，**不经过属性查找**，所以：

1. 实例上挂 `__len__` 无效（slot 在类型上）。
2. dunder 调用比等价的普通方法调用快。
3. 定义类时，CPython 会扫描类命名空间，把 `__len__` 之类**填进对应 slot**（slot wrapper 机制）。

## 4. 引用计数

CPython 的主内存管理机制是**引用计数（reference counting）**：每个对象记录有多少个引用指向它，
**归零立刻释放**。

```python
import sys
a = []
sys.getrefcount(a)      #=> 2   （a 本身 + getrefcount 的参数临时引用）

b = a
sys.getrefcount(a)      #=> 3
del b
sys.getrefcount(a)      #=> 2
```

> ⚠️ `sys.getrefcount(x)` 的结果总是**比你以为的多 1**，因为传参本身创建了一次引用。

引用计数**增加**的时机：赋值给变量、加入容器、作为参数传递、成为属性值。
**减少**的时机：`del`、重新赋值、离开作用域、容器被销毁。

### 优点与缺点

| 优点 | 缺点 |
|---|---|
| **即时回收**（不等 GC 周期），内存曲线平滑 | **无法回收循环引用** → 需要额外的 GC（见 [[internals/garbage-collection]]） |
| 实现简单，析构时机确定（`with` 之外也常"碰巧"正确） | 每次赋值都要改计数 → **性能开销大** |
| 无 stop-the-world 长暂停 | 计数字段占内存 |
| | **多线程下计数必须原子操作 → 这是 GIL 存在的核心原因**（见 [[internals/gil]]） |

> **面试落点**：把「引用计数 + GIL + 循环 GC」三者串起来讲：
> CPython 用引用计数做主回收，为了让计数增减在多线程下安全且不用给每个对象加锁，
> 引入了全局解释器锁 GIL；引用计数解决不了循环引用，所以又加了分代标记-清除 GC。
> **三者是一套互相牵制的设计，不是三个独立特性。**

### 引用计数的可观测行为

```python
class R:
    def __del__(self): print("freed")

r = R()
r = None        #=> freed   立刻打印（Java/JS 里不会这样）

# 因此这在 CPython 上"碰巧"能工作：
data = open("f.txt").read()     # 文件对象 refcnt 归零 → 立即关闭
# 但在 PyPy 上文件会延迟关闭 → fd 耗尽。所以必须用 with。
```

## 5. 对象缓存与驻留

CPython 用一批缓存避免重复创建常见小对象：

```python
# ① 小整数缓存 [-5, 256]，解释器启动时预创建
a, b = 256, 256
a is b          #=> True
a, b = 257, 257
a is b          #=> True（同一代码对象内被常量折叠）；分两行在 REPL 里则 False

# ② 单字符字符串与"标识符样"字符串驻留
"a" is "a"                     #=> True
sys.intern("hello world")      # 手动驻留（大量重复字符串做 dict key 时能省内存+加速比较）

# ③ 空 tuple / frozenset 是单例
() is ()        #=> True
(1,) is (1,)    #=> False（在 REPL）

# ④ True/False/None 是单例 —— 所以永远用 `is None`
```

驻留的价值：**字符串比较可以先比指针**（相同指针立即返回相等），dict 查找因此变快。

> 这些**全是 CPython 实现细节**，写代码绝不能依赖。面试里主动指出这一点是加分项。

### 不朽对象（immortal objects）—— 它们的引用计数根本不动

这些缓存/驻留对象里有一批是**静态分配**的，出生时引用计数就被设成天文数字，**永不归零**：

```python
# 3.11.9 实测
sys.getrefcount(0)      #=> 1000000067     ← 基数 999999999
sys.getrefcount('abc')  #=> 1000000023
sys.getrefcount(())     #=> 1000000009
sys.getrefcount(257)    #=> 4              ← 普通对象，对比
sys.getrefcount(None)   #=> 3885           ← 3.11 里 None 还是普通计数

# 3.14.6 实测
sys.getrefcount(None)   #=> 3221225472     = 0xC0000000
sys.getrefcount(1)      #=> 3221225472
sys._is_immortal(None)  #=> True           ← 3.14+ 新增的官方判定方式
```

源头在 3.11 的 `Include/internal/pycore_object.h`：

```c
#define _PyObject_IMMORTAL_INIT(type)     { .ob_refcnt = 999999999, ... }
```

演进：

| 版本 | 机制 |
|---|---|
| 3.11 | 静态分配对象用 `_PyObject_IMMORTAL_INIT`，`ob_refcnt = 999999999`。**只覆盖小整数 / interned 字符串 / 空 tuple / 静态类型**，`None`/`True`/`False` 仍是普通计数 |
| 3.12 | **PEP 683 正式化**，`None`/`True`/`False` 也变成不朽，inc/dec 直接短路 |
| 3.14 | 魔数变成 `0xC0000000`，新增 `sys._is_immortal()` |

> ⚠️ **别背魔数**——每个版本都在变。要判断就用 `sys._is_immortal(obj)`（3.14+）。

两个下游影响：

- **GC 视角**：不朽对象的 `gc_refs` 减完仍是天文数字，**永远是"存活根"**，
  从它可达的一切都被标活。见 [[internals/garbage-collection]] §4.2。
- **free-threading 视角**：PEP 683 是 PEP 703 的前置条件——最热的共享对象不再改计数，
  消除了多线程下的引用计数争用。见 [[internals/gil]]。

## 6. 变量、名字与 `__dict__`

```python
# 模块/类/实例的属性都存在 dict 里（除非用 __slots__）
class C:
    def __init__(self): self.x = 1

c = C()
c.__dict__          #=> {'x': 1}
C.__dict__          #=> mappingproxy({...})  只读视图
globals()           #=> 模块的命名空间（真 dict，可写）
```

实例字典经历过两代优化，**版本别记混**：

- **3.3**：共享键字典（key-sharing / split table，**PEP 412**）。键表挂在类型对象上
  （heap type 的 `ht_cached_keys`），同一个类的所有实例共享一份，每个实例只留一个值数组。
- **3.11**：**内联值数组 + `__dict__` 惰性创建**（faster-cpython 项目，非 PEP 412）。
  值数组与对象本体紧邻存放（3.12 起真正内联进对象的那块分配），且只要你从不访问
  `o.__dict__`，那个 dict 对象**根本不会被创建**。

```python
c1, c2 = C(), C()
c1.__dict__.keys() == c2.__dict__.keys()    #=> True   键的身份与插入顺序都一致
```

这确实削弱了 `__slots__` 的优势——实测每实例 **104 字节 vs `__slots__` 的 64 字节**
（3 个属性，3.11.9 / 64 位，`tracemalloc` 量 20 万实例的人均值），从早年的好几倍缩到 1.6 倍。
**速度优势也一并缩水**：3.11 的 specializing 解释器有 `LOAD_ATTR_INSTANCE_VALUE`，
普通实例读属性在校验键表版本后也是「按固定下标读值数组」，与 `LOAD_ATTR_SLOT` 已很接近。

⚠️ 但这是**机会性优化**，两个脆弱点决定了它在真实代码里经常拿不到：

| 失效场景 | 实测代价 |
|---|---|
| 访问过一次 `o.__dict__`（强制物化成真 dict） | 104 → **168** 字节/实例 |
| 实例间属性名/插入顺序分歧（共享解除，退化成独立 dict） | 88 → **342** 字节/实例 |

触发物化的操作比想象中多：`vars(o)`、默认的 `pickle`/`copy`、不少 ORM 与序列化库、调试器。
导致键序分歧的写法：`__init__` 里条件赋值、事后 `o.extra = 1`、`del o.attr`、
拿外部数据当属性名 `setattr(o, k, v)`。

> **面试落点**：想省实例内存，`__slots__` 仍是唯一**可靠**手段——收益不依赖「有没有碰
> `__dict__`」「键序有没有分歧」这些前提；拿不到优化不会报错，你只会看到内存莫名多 60%。
> 加分补充：PEP 412 是 **3.3** 就有的，3.11 的新东西是内联值 + `__dict__` 惰性创建，别混。

## 7. 局部变量不走字典

函数的局部变量**不存在 dict 里**，而是编译期分配的**数组槽位**：

```python
def f():
    x = 1
    return x

f.__code__.co_varnames    #=> ('x',)
f.__code__.co_nlocals     #=> 1
import dis; dis.dis(f)
#   LOAD_CONST 1
#   STORE_FAST 0 (x)      ← 数组索引 0，不是哈希查找
#   LOAD_FAST  0 (x)
```

这就是**局部变量比全局变量快**的原因（`LOAD_FAST` 是数组访问，`LOAD_GLOBAL` 是两次字典查找）。

```python
# 热循环里的经典微优化
def f(items):
    _len = len              # 把全局/内建绑定成局部
    return [_len(x) for x in items]
```

> **面试落点**：「为什么局部变量访问比全局快？」——`LOAD_FAST` 走数组索引，
> `LOAD_GLOBAL` 要查模块 dict 再查 builtins dict。3.11 起 `LOAD_GLOBAL` 加了内联缓存，差距缩小但仍在。

## 8. CPython 之外

| 实现 | 特点 |
|---|---|
| **CPython** | 参考实现，C 写的，引用计数 + GIL |
| **PyPy** | JIT 编译，纯 GC（无引用计数），长时间运行的纯 Python 快 5~50 倍；C 扩展兼容性差 |
| **Jython / IronPython** | JVM / .NET 上，无 GIL，落后于主线版本 |
| **MicroPython** | 嵌入式 |
| **Cython / mypyc** | 把 Python 编译成 C 扩展 |

3.13 起 CPython 有了**自由线程（free-threading）构建**和**实验性 JIT**，见 [[internals/gil]]。

## 相关

- [[internals/garbage-collection]] —— 循环引用与分代 GC
- [[internals/gil]] —— 引用计数为什么导致 GIL
- [[internals/memory-model]] —— pymalloc 与内存分配层级
- [[internals/bytecode-execution]] —— LOAD_FAST/LOAD_GLOBAL 与求值循环
- [[language/data-model]] —— dunder 与 slot 的对应关系
- [[stdlib/builtin-data-structures]] —— list/dict 的 C 层布局
- [[interview/question-bank-internals-concurrency]] —— 相关面试题
