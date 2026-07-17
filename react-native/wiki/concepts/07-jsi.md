---
title: JSI — JS 与 C++ 的双向通信
date: 2026-07-09
tags: [jsi, c++, turbomodules, threading, host-function, host-object]
sources: [2026-07-09-reactnative-dev-new-architecture.md]
---

# JSI:JS 与 C++ 的双向通信

JSI(JavaScript Interface)是新架构的地基:一套**轻量、引擎无关的 C++ API**,把 JS 引擎(Hermes/JSC/V8)**嵌入到 C++ 程序**,让两侧**同进程、同线程**地直接互操作。它取代了旧的序列化异步桥(见 [[concepts/06-legacy-bridge]])。

## 为什么免序列化

旧 Bridge 是异步消息队列,JS 与原生是两个隔离内存空间,只能把数据拍平成 **JSON 文本**跨边界传递。JSI 把边界拆掉:C++ 直接握着引擎句柄(`jsi::Runtime`),双方能**直接持有对方引用、直接调用对方函数**,不需要序列化。例如 JS 传数字给 C++,C++ 收到的是直接指向引擎中该值的 `jsi::Value`,`.getNumber()` 当场读出,全程无 JSON。

## 双向:两个方向都能

JSI 不是单行道。本质是"C++ 能操控 JS 引擎",在此之上两条路都打通:

| 方向 | 机制 | 例子 |
|------|------|------|
| **JS → C++** | HostObject / HostFunction:C++ 把对象/函数"安装"进 JS 运行时,JS 一调用引擎就直接执行 C++ | TurboModule 方法调用 |
| **C++ → JS** | C++ 持有 `jsi::Function` / `jsi::Object` 引用后直接调用/读写 | 事件回调、Promise 兑现、Fabric 触发渲染 |

## C++ 何时需要调用 JS

只要"结果/事件是原生这边先产生、需要主动通知 JS",就是 C++→JS:

1. **异步结果回传**(最常见):TurboModule 做网络/文件/DB 等耗时操作,完成后 resolve Promise 或调用 JS 传入的 callback。
2. **事件分发**:原生 UI 事件(触摸、滚动、文本变化)或原生模块 `emit` 事件,回到 JS 监听器。
3. **定时器**:native timer 到点回调 JS。
4. **Fabric 驱动 React**:调度器触发 JS 跑 render/commit。
5. **动画/手势**(Reanimated):C++ 在 UI 线程的独立 JS runtime 上触发 worklet。

## 关键约束:JSI 调用必须在 JS 线程

> `jsi::Runtime` 与 `jsi::Value`/`Function` **不是线程安全的**。

若 C++ 在**后台线程**做完活,**不能**直接 `callback.call(...)`,必须通过 **`CallInvoker`** 把"调用 JS"排回 **JS 线程**执行。这是真实代码最易踩错的点,也决定了下面例子为何要用 `invokeAsync`。

## 完整例子:异步 TurboModule 方法回调 JS

场景:JS 调 `fetchUserName(userId, callback)`,C++ 在后台线程"查库",查完回调 JS。

**C++ 侧:**

```cpp
#include <jsi/jsi.h>
#include <ReactCommon/CallInvoker.h>
#include <thread>
#include <memory>
#include <string>

using namespace facebook;
using namespace facebook::jsi;

// jsInvoker 由 TurboModule 基础设施注入,负责把 lambda 派发回 JS 线程
void installFetchUserName(Runtime& runtime,
                          std::shared_ptr<react::CallInvoker> jsInvoker) {

  auto fetchUserName = Function::createFromHostFunction(
      runtime,
      PropNameID::forAscii(runtime, "fetchUserName"),
      2,  // 参数个数:(userId, callback)
      [jsInvoker](Runtime& rt,
                  const Value& thisVal,
                  const Value* args,
                  size_t count) -> Value {

        // (1) 直接从引擎读取 JS 参数 —— 没有 JSON 解析
        int userId = static_cast<int>(args[0].asNumber());

        // (2) 取到 JS 回调函数的引用;move 进 shared_ptr 以便跨线程持有
        auto callback = std::make_shared<Function>(
            args[1].asObject(rt).asFunction(rt));

        // (3) 后台线程做耗时工作 —— 注意:这里绝不能碰 rt / callback.call
        std::thread([userId, callback, jsInvoker]() {
          std::string name = "user_" + std::to_string(userId);  // 模拟查询结果

          // (4) 把"调用 JS 回调"排回 JS 线程执行(C++ → JS 的关键一步)
          jsInvoker->invokeAsync([callback, name](Runtime& rt) {
            // 此刻已在 JS 线程,安全调用 JS 函数
            callback->call(rt, String::createFromUtf8(rt, name));
          });
        }).detach();

        // (5) 立即返回;真正的结果通过回调异步给出
        return Value::undefined();
      });

  runtime.global().setProperty(runtime, "fetchUserName", std::move(fetchUserName));
}
```

**JS 侧:**

```js
fetchUserName(42, (name) => {
  console.log('got', name);  // → got user_42
});
```

**对着代码看方向:**

- **JS→C++** = 第 (1)(2) 步:JS 调 `fetchUserName`,引擎直接执行这段 C++ HostFunction。
- **C++→JS** = 第 (4) 步:C++ 持有的 `jsi::Function`(JS 回调)经 `callback->call(rt, ...)` 主动回调进 JS。
- (3)(4) 的 `std::thread` + `invokeAsync` 正是为满足"JSI 调用必须在 JS 线程"这条约束。

> 简化情形:若 C++ 的活是**同步的、且本就在 JS 线程**上跑,可省掉线程与 `invokeAsync`,直接 `callback.call(rt, ...)`。异步版是真实模块的常态。

## 注意事项 / 待核实

- `CallInvoker::invokeAsync` 签名**随 RN 版本变过**:新版为 `std::function<void(jsi::Runtime&)>`(上例所用),旧版为 `std::function<void()>`(需自行捕获 runtime 指针)。落地请对齐项目的 RN 版本。
- 本页 C++ API 细节(`jsi::Runtime`/`HostFunction`/`CallInvoker`)源自通用知识;官方 architecture 摘要对 JSI 展开较浅,仅给出"直接引用、免序列化"的结论。待抓取 JSI 源码级/官方一手资料作 raw 佐证。

## 交叉引用

- 它取代的旧桥接:[[concepts/06-legacy-bridge]]
- 新架构总览:[[concepts/01-new-architecture]]
- JSI 之上的渲染:[[concepts/02-fabric-render-pipeline]]
- 线程约束背景:[[concepts/03-threading-model]]
- 术语(JSI / Codegen):[[concepts/05-glossary]]
