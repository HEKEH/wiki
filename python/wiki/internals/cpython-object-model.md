---
title: "CPython 对象模型与引用计数"
date: 2026-08-07
tags: [CPython, PyObject, 引用计数, 类型对象, 缓存, 内部实现]
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

## 2. 类型对象与 slot

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

## 3. 引用计数

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

## 4. 对象缓存与驻留

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

## 5. 变量、名字与 `__dict__`

```python
# 模块/类/实例的属性都存在 dict 里（除非用 __slots__）
class C:
    def __init__(self): self.x = 1

c = C()
c.__dict__          #=> {'x': 1}
C.__dict__          #=> mappingproxy({...})  只读视图
globals()           #=> 模块的命名空间（真 dict，可写）
```

CPython 3.11+ 引入了**共享键字典（key-sharing dict，PEP 412）**：
同一个类的所有实例共享一份"键表"，只各存值数组，大幅降低实例内存。
这削弱了 `__slots__` 的相对优势，但 `__slots__` 依然更省更快。

## 6. 局部变量不走字典

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

## 7. CPython 之外

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
