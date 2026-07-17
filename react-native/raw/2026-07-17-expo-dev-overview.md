# Expo — 官方文档摘要(抓取存档)

来源:
- https://docs.expo.dev/ (Expo overview)
- https://docs.expo.dev/workflow/overview/ (Development workflow)
- https://docs.expo.dev/router/introduction/ (Expo Router introduction)

抓取日期:2026-07-17

---

## Expo 是什么

Expo 是构建在 React Native 之上的**框架 / 平台**:"Build one JavaScript/TypeScript project that runs
natively on all your users' devices." 面向 Android、iOS、Web 的通用(universal)应用开发。

React Native 官方现在推荐**用一个 Framework(如 Expo)起步新项目**,而不是裸初始化,因为裸 RN 把大量
原生工程配置留给开发者。

## 核心工具

- **Expo SDK** —— 一组预封装的原生能力模块(Image、Camera、Notifications 等),免去自写/链接原生代码。
- **Expo CLI** —— 命令行工具。`npx create-expo-app@latest` 初始化项目等。
- **EAS (Expo Application Services)** —— 云端服务:EAS Build(云端构建 iOS/Android)、EAS Update
  (JS bundle 空中更新 / OTA)、EAS Submit(提交商店)、EAS Workflows(自动化构建与发布)。
- **Expo Router** —— 基于文件系统的路由库(见下)。
- **Expo Go** —— 扫码即在真机运行的沙盒。"the fastest way to get started",但"a limited playground
  and not useful for building production-grade projects"。生产项目应改用 Development build。

## 开发工作流

- **Development build(开发构建)** —— 含 `expo-dev-client` 库的 debug 构建,比 Expo Go "more flexible,
  reliable, and complete";支持安装原生库、修改原生工程。生产级开发的推荐方式。
- **Prebuild** —— 按需从应用配置(app config)生成原生工程目录 `android/`、`ios/`;这些目录默认加入
  `.gitignore`,可随时删除并重新生成。
- **Continuous Native Generation (CNG)** —— 让原生工程"声明式定义于配置、而非手改源码"的机制,类似
  包管理器管理依赖;简化升级与维护。
- **Config Plugins** —— 在不直接改原生代码的前提下修改原生工程配置。
- 可选择 CNG(声明式生成)或 prebuild 后在 Xcode/Android Studio 手改原生目录;两者混用会破坏再生成能力。

## Expo Router

- "an open-source routing library for Universal React Native applications built with Expo." 文件式路由,
  跨 Android / iOS / Web 共享组件与路由逻辑。
- **文件式路由**:`app/` 目录下的文件自动转为可导航路由,无需手动在代码里定义 navigator/route;支持
  **typed routes**(链接到不存在的目标会报错)。
- **对比 React Navigation**:Expo Router 由目录结构派生路由;React Navigation 需在代码中手动定义
  navigator 与 route。二者取决于项目更适合哪种模型(Expo Router 底层基于 React Navigation)。
- **Universal 路由**:同一套导航结构与组件覆盖三端,路由级别可用平台特定 API。
- **自动深链**:"Every screen in your app is automatically deep linkable" —— 每个屏幕天然可深链、可分享。
