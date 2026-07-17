# React Native — Index

按类别组织的全部 wiki 页面目录,每次 ingest 更新。

## Overview

- [[home]] — 顶层综合与导航

## Concepts

- [[concepts/01-new-architecture]] — 新架构综合:JSI/Fabric/TurboModules/Codegen/Yoga 的关系与版本演进
- [[concepts/02-fabric-render-pipeline]] — Fabric 与 Render→Commit→Mount 管线、三棵树、结构共享、View Flattening
- [[concepts/03-threading-model]] — JS 线程 vs UI 线程、帧预算、不可变性带来的无锁线程安全
- [[concepts/04-performance-optimization]] — 先测量、按影响排序的性能方法论 + 量化预算
- [[concepts/05-glossary]] — 新架构术语表
- [[concepts/06-legacy-bridge]] — 旧桥接架构与瓶颈;JSI 如何让 JS↔C++ 免序列化直接通信
- [[concepts/07-jsi]] — JSI 双向通信:JS↔C++ 机制、C++ 何时调 JS、JS 线程约束、`fetchUserName` 代码例子

## Entities

- [[entities/01-hermes]] — 默认 JS 引擎:bytecode 预编译、Bundled/Static Hermes
- [[entities/02-expo]] — React Native Framework:定位、EAS/Expo SDK/Expo Router 工具链、Prebuild/CNG

## Sources

- [[sources/01-official-getting-started]] — 官方入门/学习主线摘要
- [[sources/02-official-new-architecture]] — 官方新架构专章摘要
- [[sources/03-official-architecture-deep-dive]] — 官方 architecture 五子页(Fabric/管线/线程/Flattening/术语)
- [[sources/04-official-performance]] — 官方 Performance Overview
- [[sources/05-rn-advanced-guide]] — 开源进阶书 12 章路线图
- [[sources/06-rapidnative-performance-playbook]] — 第三方性能 playbook(量化预算)

## Analysis

- [[analysis/01-source-backlog]] — 精选源清单、抓取进度与建议学习顺序
