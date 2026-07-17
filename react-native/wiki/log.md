# React Native — Activity Log

Append-only chronological record of wiki activity.

## [2026-07-09] ingest | 建库 + 官方文档首批入库

- 由 `_template` 新建 `react-native` 知识库,定制 CLAUDE.md(领域、分类、约定)。
- 抓取并存档两个官方源到 `raw/`:Getting Started、About the New Architecture。
- 新建页面:
  - [[sources/01-official-getting-started]]、[[sources/02-official-new-architecture]]
  - [[concepts/01-new-architecture]](新架构综合)
  - [[analysis/01-source-backlog]](精选源清单与抓取计划)
- 更新 home / index / log。
- 检索确认的候选源(未入库,见 backlog):RN Advanced Guide 开源书、Callstack 优化 ebook、官方 architecture 子页、官方 Performance、awesome-react-native。

## [2026-07-09] ingest | 抓取剩余高质量源(第二批)

- 抓取并存档到 `raw/`:官方 architecture 深入(Fabric/Render-Commit-Mount/Threading/View Flattening/Glossary 五子页合集)、官方 Performance Overview、RN Advanced Guide TOC、RapidNative 性能 playbook。
- 新建 concept 页:[[concepts/02-fabric-render-pipeline]]、[[concepts/03-threading-model]]、[[concepts/04-performance-optimization]]、[[concepts/05-glossary]]。
- 新建 entity 页:[[entities/01-hermes]]。
- 新建 source 页:[[sources/03-official-architecture-deep-dive]]、[[sources/04-official-performance]]、[[sources/05-rn-advanced-guide]]、[[sources/06-rapidnative-performance-playbook]]。
- 更新 [[concepts/01-new-architecture]](补全渲染管线 + 交叉引用)、[[analysis/01-source-backlog]](标记入库进度 + 下一批主题)、index、home。
- 仍待抓取:官方 Cross-Platform Implementation / Bundled Hermes / Architecture Overview、Callstack ebook(表单 gated)、callstack/optimization-best-practices、awesome-react-native;新主题:导航、状态管理、原生模块、测试。

## [2026-07-09] query | 问答沉淀 — 旧桥接架构 + JSI 通信机制

- 由多轮问答综合(源自已入库官方源),新建 [[concepts/06-legacy-bridge]]:Bridge 三线程模型、序列化/异步/批量三特征、为何必须序列化、JSI 如何用 HostObject/HostFunction/`jsi::Value` 让 JS↔C++ 免序列化直接通信、新旧对比、移除时间线。
- 交叉链接:[[concepts/01-new-architecture]] 增加指向本页的引用;更新 index。
- 前几轮的就地细化(01 页图例:原生平台 API vs 原生视图;02 页 Mount mutation 指令集 + "为何用 C++"小节)一并归入本日维护。

## [2026-07-09] query | 问答沉淀 — JSI 双向通信专页

- 综合多轮问答,新建 [[concepts/07-jsi]]:JSI 免序列化原理、JS↔C++ 双向机制(HostObject/HostFunction vs 持有 `jsi::Function` 反向调用)、C++ 何时需要调 JS 的五类场景、"JSI 调用必须在 JS 线程"约束、完整 `fetchUserName(userId, callback)` C++/JS 代码例子(含 CallInvoker/invokeAsync)。
- 交叉链接:[[concepts/06-legacy-bridge]] 增补指向本页;更新 index。
- 修复 06 页 diagram 代码块语言标注(MD040)。
- 缺口标注:JSI 源码级/官方一手资料尚未作为 raw 入库,C++ API 细节暂据通用知识,待抓取佐证。

## [2026-07-17] ingest | Expo(React Native Framework)首页入库

- 由"Expo 起什么作用"的追问触发。抓取官方文档(docs.expo.dev overview / workflow overview / router introduction),
  存档为 `raw/2026-07-17-expo-dev-overview.md`。
- 新建 [[entities/02-expo]]:框架 vs 库定位、核心工具链(Expo SDK / Expo CLI / EAS Build·Update·Submit /
  Expo Go / Development build)、Prebuild + CNG + Config Plugins、Expo Router(文件式路由 vs React Navigation)。
- 交叉链接:[[concepts/05-glossary]] "React Native Framework" 条目指向本页;更新 index、home
  (学习动线加"生态/工具链"、开放问题更新导航覆盖)、[[analysis/01-source-backlog]](导航主题标注部分入库)。
- 仍待:React Navigation 独立深挖、Expo Router 实操、Expo SDK 与新架构兼容细节。
