---
title: "经典陷阱题（wtfpython 精选）"
date: 2026-08-07
tags: [陷阱, 面试题, wtfpython, 边界情况]
sources: ["wtfpython.md"]
---

# 经典陷阱题精选

来自 `satwikkansal/wtfpython` 与真实面试。**每一道都能考出你对某个底层机制的理解**，
所以答题时不要只说"结果是 X"，要说清**为什么**。

## 1. 可变默认参数 ★★★

```python
def add(x, lst=[]):
    lst.append(x)
    return lst

add(1)      #=> [1]
add(2)      #=> [1, 2]      ❗
```

**原因**：默认值在**函数定义时求值一次**，存在 `func.__defaults__` 里，所有调用共享。
**解法**：`lst=None` + `if lst is None: lst = []`。
**延伸**：dataclass 用 `field(default_factory=list)`；Pydantic **不受影响**（每实例深拷贝）。

## 2. 循环变量的延迟绑定 ★★★

```python
fns = [lambda: i for i in range(3)]
[f() for f in fns]      #=> [2, 2, 2]      ❗ 不是 [0,1,2]
```

**原因**：lambda 捕获的是**变量 `i` 本身**（cell），不是它当时的值。
**解法**：`lambda i=i: i`（默认参数在定义时求值）。
**加分**：这就是 ES5 `var` 时代 `setTimeout` 的同一个问题，JS 用 `let` 解决了，
**Python 没有 `let`，所以这个坑至今存在**。

## 3. `is` vs `==` 与小整数缓存 ★★★

```python
a = 256; b = 256
a is b          #=> True

a = 257; b = 257
a is b          #=> False（交互式）/ True（同一代码对象内）

"a" * 20 is "aa" * 10       #=> True （编译期常量折叠）
a = "a" * 20; b = "aa" * 10
a is b                       #=> False（运行期构造）
```

**原因**：CPython 缓存 `-5~256` 的小整数；编译器会折叠常量并让相同常量共享。
**结论**：**`is` 比较身份，`==` 比较值；永远不要用 `is` 比较值**。
**加分**：主动指出"这些都是 **CPython 实现细节**，PyPy 上不同，业务代码不能依赖"。

## 4. 链式比较 ★★

```python
False == False in [False]       #=> True      ❗
```

**原因**：链式比较展开为 `(False == False) and (False in [False])` = `True and True`。
同理 `1 < 2 < 3` 是 `1<2 and 2<3`，而不是 `(1<2)<3`。

```python
a = 1
print(a is not None)      #=> True    这是 "is not" 运算符
print(a is (not None))    #=> False   这是 is 比较 True
```

## 5. `finally` 吞异常 / 吞返回值 ★★★

```python
def f():
    try:
        return 1
    finally:
        return 2
f()         #=> 2      ❗ finally 的 return 覆盖了 try 的

def g():
    try:
        raise ValueError
    finally:
        return 3
g()         #=> 3      ❗ 异常被静默吞掉
```

**结论**：**永远不要在 `finally` 里 `return`/`break`/`continue`**。

## 6. 遍历时修改容器 ★★★

```python
# list：不报错，但会【静默漏掉元素】—— 比报错更危险
lst = [1, 2, 2, 3]
for x in lst:
    if x % 2 == 0:
        lst.remove(x)
lst         #=> [1, 2, 3]   ❗ 第二个 2 没被删掉

# 追踪：迭代器持有的是【索引】，删除元素会让后面的元素前移，索引就跳过了一个
#   i=0 → x=1，不删
#   i=1 → x=2，删除 → 列表变成 [1, 2, 3]
#   i=2 → x=3（第二个 2 已经前移到 i=1，被跳过了！）
#   i=3 → 越界，循环结束

lst = [1, 2, 3, 4]      # ⚠️ 这个输入【碰巧】结果正确，容易让人误以为写法没问题
for x in lst:
    if x % 2 == 0: lst.remove(x)
lst         #=> [1, 3]

# dict/set：直接报错（比 list 友好）
d = {"a": 1, "b": 2}
for k in d:
    del d[k]        # ❌ RuntimeError: dictionary changed size during iteration
```

**解法**：

```python
lst = [x for x in lst if x % 2]           # 新建
for k in list(d):                          # 物化 keys 的快照
    del d[k]
```

## 7. 类作用域不参与闭包 ★★

```python
x = "global"
class A:
    x = "class"
    def get(self):      return x            #=> 'global'  ❗ 不是 'class'
    lst = [x for _ in range(1)]             #=> ['global'] ❗
```

**原因**：类体的命名空间**不是**闭包作用域（不产生 cell），LEGB 的 E 只包括**函数**。
方法里要用类属性必须写 `self.x` 或 `A.x`。

## 8. 可变对象在不可变容器里 ★★★

```python
t = (1, [2])
t[1] += [3]
# ❗ TypeError: 'tuple' object does not support item assignment
t           #=> (1, [2, 3])      ❗ 但是修改【成功了】！
```

**原因**：`t[1] += [3]` 编译成：① `list.__iadd__` 就地扩展（**成功**）；
② `t[1] = 结果` 赋值回 tuple（**失败**）。所以"既报错又生效"。
**加分**：用 `dis` 展示 `BINARY_OP` 后跟着 `STORE_SUBSCR`。
**解法**：`t[1].extend([3])`（只做步骤 ①）。

## 9. `__eq__` 与 `__hash__` 的一致性 ★★

```python
class A:
    def __eq__(self, other): return True

a, b = A(), A()
{a, b}          # ❌ TypeError: unhashable type: 'A'
```

**原因**：定义 `__eq__` 会把 `__hash__` 自动设为 `None`。

```python
some_dict = {5.0: "Ruby", 5: "Python", 5.5: "Go"}
some_dict[5]        #=> 'Python'    ❗ 5.0 的值被 5 覆盖了
hash(5.0) == hash(5)   #=> True     且 5.0 == 5
```

**原因**：dict 的键去重依据是 `hash` + `==`，`5 == 5.0` 且哈希相同 → 视为同一个键
（**保留最先插入的键对象，但更新值**）。同理 `True` 和 `1`、`False` 和 `0`。

```python
{True: "yes", 1: "no", 1.0: "maybe"}    #=> {True: 'maybe'}
```

## 10. `+=` vs `= ... +` ★★

```python
a = [1, 2]; b = a
a += [3]        # 就地修改（__iadd__）
b               #=> [1, 2, 3]     ❗ b 跟着变

a = [1, 2]; b = a
a = a + [3]     # 创建新列表并重新绑定
b               #=> [1, 2]        b 不变
```

## 11. `is` 与 `float("nan")` ★

```python
x = float("nan")
x == x          #=> False     ❗ NaN 不等于自己（IEEE 754）
x is x          #=> True

s = {x, x}
len(s)          #=> 1         ❗ set 先比 is 再比 ==，同一对象直接去重
s2 = {float("nan"), float("nan")}
len(s2)         #=> 2         不同对象，== 又为 False → 两个都留下
```

## 12. 生成器的惰性求值时机 ★★

```python
array = [1, 8, 15]
g = (x for x in array if array.count(x) > 0)
array = [2, 8, 22]
list(g)         #=> [8]       ❗
```

**原因**：**最外层的可迭代对象（`for x in array`）在创建生成器时就求值了**，
但**条件里的 `array` 是在迭代时才求值**——此时 `array` 已经指向新列表。

## 13. `zip` 的有损特性（3.10+ 用 strict） ★

```python
nums = iter([1, 2, 3, 4, 5])
list(zip(nums, "ab"))       #=> [(1,'a'), (2,'b')]
list(nums)                  #=> [4, 5]     ❗ 3 被消耗掉了！
```

**原因**：zip 取到 'b' 后又从 nums 取了 3，发现字符串没了才停——3 被丢弃。
**解法**：`zip(a, b, strict=True)`（3.10+）在长度不等时直接报错。

## 14. 字符串驻留与 `+=` 性能 ★

```python
# 看起来 O(n) 其实取决于实现
s = ""
for x in items:
    s += x          # CPython 在 refcount==1 时有原地扩容优化，但不可依赖
```

## 15. 默认参数 + 时间 ★★

```python
def log(msg, ts=datetime.now()):        # ❗ 时间被冻结在【导入时刻】
    print(ts, msg)
```

## 16. 布尔是整数的子类 ★

```python
isinstance(True, int)       #=> True
True + True                 #=> 2
sum([True, True, False])    #=> 2       ← 这其实是个有用的技巧（统计 True 个数）
["a", "b"][True]            #=> 'b'
```

## 17. 银行家舍入 ★★

```python
round(0.5)      #=> 0       ❗ 不是 1
round(1.5)      #=> 2
round(2.5)      #=> 2       ❗ 不是 3
round(-0.5)     #=> 0
```

**原因**：Python 3 用 **banker's rounding（四舍六入五成双）**，
减少大量舍入的累积偏差。金额计算要用 `Decimal` + 显式指定舍入模式：

```python
from decimal import Decimal, ROUND_HALF_UP
Decimal("2.5").quantize(Decimal("1"), rounding=ROUND_HALF_UP)   #=> Decimal('3')
```

## 18. 整数字符串转换限制（3.11+） ★

```python
int("1" * 5000)     # ❌ ValueError: Exceeds the limit (4300 digits) for integer
                    #    string conversion
sys.set_int_max_str_digits(0)      # 解除限制
```

**原因**：防止超大整数转换导致的 **DoS 攻击**（转换是二次复杂度）。

## 19. `except` 里的变量会被删除 ★

```python
e = "original"
try:
    raise ValueError
except ValueError as e:
    pass
print(e)        # ❌ NameError: name 'e' is not defined    ❗ 连原来的 e 也没了
```

**原因**：Python 3 在 `except` 块结束时**显式 `del e`**，防止异常对象持有的
traceback 引用整个栈帧造成内存泄漏。需要用就先存到别的变量。

## 20. 类属性 vs 实例属性 ★★★

```python
class A:
    x = 1

a, b = A(), A()
a.x = 2                    # 创建了实例属性，遮蔽类属性
A.x = 3                    # 改类属性
a.x, b.x, A.x              #=> (2, 3, 3)

class B:
    items = []             # ❗ 共享
b1, b2 = B(), B()
b1.items.append(1)
b2.items                   #=> [1]
```

## 21. 元组的逗号 ★

```python
t = 1,          #=> (1,)    ❗ 逗号才是构造 tuple 的关键，不是括号
t = (1)         #=> 1       这只是括号表达式
t = ()          #=> ()      空元组是例外

def f(): return 1,          #=> 返回 (1,) 而不是 1   ← 手滑打逗号的经典 bug
```

## 22. 字符串隐式拼接 ★

```python
names = ["alice", "bob" "charlie"]      # ❗ 少了逗号
len(names)      #=> 2                    'bob' 和 'charlie' 被隐式拼接成 'bobcharlie'
```

**原因**：相邻字符串字面量自动拼接（本是为了写长字符串方便）。
ruff 的 `ISC` 规则能抓住它。

## 面试应对策略

遇到陷阱题时的**标准答题结构**：

1. **先说结果**（"这里会输出 [1, 2] 而不是 [2]"）
2. **再说机制**（"因为默认值在函数定义时求值一次，存在 `__defaults__` 里"）
3. **然后说解法**（"用 None 哨兵"）
4. **最后加一句实现层或对照**（"这是 CPython 的实现细节" / "JS 里用 let 解决了这个问题"）

第 4 步是拉开差距的关键——它证明你不是背题，而是理解了机制。

## 相关

- [[language/objects-mutability]] —— 可变性与 is/==
- [[language/scope-closure]] —— 延迟绑定与类作用域
- [[language/functions-arguments]] —— 默认参数
- [[language/exceptions]] —— finally 与 except 变量删除
- [[stdlib/builtin-data-structures]] —— hash/eq 契约
- [[interview/question-bank-language]] —— 系统的语言面试题
- [[sources/wtfpython]] —— 来源导读
