---
title: "源：faif/python-patterns"
date: 2026-08-07
tags: [源导读, 设计模式, 架构]
sources: ["python-patterns.md"]
---

# 源：faif/python-patterns

- **原始路径**：`raw/python-patterns.md`（12 KB，仓库 README 索引）
- **来源**：<https://github.com/faif/python-patterns>（40k+ star）
- **抓取日期**：2026-08-07

## 这份材料是什么

**Python 版的设计模式实现集**。README 是一份索引，每个模式对应仓库里一个可运行的
`.py` 文件（本次只抓取了索引，未抓全部实现）。分类：

```text
Creational（创建型）
  abstract_factory  borg  builder  factory  lazy_evaluation
  pool  prototype

Structural（结构型）
  3-tier  adapter  bridge  composite  decorator  facade
  flyweight  front_controller  mvc  proxy

Behavioral（行为型）
  chain_of_responsibility  catalog  chaining_method  command
  iterator  mediator  memento  observer  publish_subscribe
  registry  specification  state  strategy  template  visitor

Design for Testability
  dependency_injection  setter_injection

Fundamental
  delegation_pattern  graph_search

Others
  blackboard  graph_search  hsm
```

## 对 Python 面试的实际价值

**中等偏低，但有几个例外**。原因：

**Python 让很多 GoF 模式变得不必要**——这本身就是一个很好的面试谈资：

| GoF 模式 | Python 里的现实 |
|---|---|
| **Strategy** | 函数是一等对象，直接传函数即可，不需要策略类 |
| **Factory** | 直接传类（类也是对象），或用 `classmethod` 替代构造器 |
| **Singleton** | **模块本身就是单例**（模块只被导入一次）；或 `functools.cache` |
| **Command** | 闭包 / `functools.partial` |
| **Iterator** | **语言内建**（迭代器协议） |
| **Decorator** | **语言内建**（`@` 语法） |
| **Observer** | 回调列表 / `asyncio.Queue` / 第三方 `blinker` |
| **Template Method** | 传函数参数，或 ABC 的抽象方法 |
| **Adapter** | 鸭子类型让大部分适配变成不必要 |
| **Visitor** | `functools.singledispatch` 或 `match` 语句 |

> **面试落点**：被问"你了解哪些设计模式"时，**最好的回答不是背 23 个模式**，而是：
> 「我更关注模式解决的问题而不是模式本身。Python 里很多 GoF 模式因为
> **函数是一等对象、有鸭子类型、模块天然单例、装饰器和迭代器是语言内建**
> 而变得不必要——比如 Strategy 直接传函数、Singleton 用模块级变量、
> Visitor 用 `singledispatch`。**在 Python 里我更常用的是依赖注入、仓储模式、
> 上下文管理器和装饰器**——比如 FastAPI 的依赖注入系统就是可测试性设计的典型。」

## 仍然重要的几个模式

| 模式 | 为什么重要 | 本库对应 |
|---|---|---|
| **Dependency Injection** | FastAPI 的核心机制；可测试性的基础 | [[web/fastapi-di]] |
| **Repository（仓储）** | 隔离数据访问，业务层不依赖 ORM | [[web/fastapi-async-db]]、[[web/fastapi-architecture]] |
| **3-tier / 分层** | 项目组织的基本盘 | [[web/fastapi-architecture]] |
| **Registry** | 插件系统；用 `__init_subclass__` 实现 | [[language/metaclasses]] |
| **Lazy Evaluation** | `cached_property`、生成器 | [[language/descriptors-properties]] |
| **Object Pool** | 数据库连接池 | [[web/fastapi-async-db]] |
| **Proxy** | 中间件、ASGI 包装 | [[web/wsgi-asgi]] |
| **Publish-Subscribe** | 消息队列、事件驱动架构 | [[web/fastapi-architecture]] |

## 局限

- 很多实现是**教学性的**，不是生产写法（比如 Singleton 的元类实现在 Python 里是过度设计）。
- 缺少 Python 特有的"模式"：上下文管理器、描述符、生成器管道、`__init_subclass__` 注册。
- 只抓了 README 索引，具体实现需要时再去仓库看。

## 相关

- [[web/fastapi-di]] —— 依赖注入的实际应用
- [[web/fastapi-architecture]] —— 分层与仓储
- [[language/metaclasses]] —— 注册表模式与单例的 Python 做法
- [[language/decorators]] —— 装饰器作为语言内建的模式
- [[interview/question-bank-web]] —— 架构面试题
