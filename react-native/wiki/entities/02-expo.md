---
title: Expo(React Native Framework)
date: 2026-07-17
tags: [expo, framework, expo-router, eas, tooling, navigation]
sources: [2026-07-17-expo-dev-overview.md]
---

# Expo

构建在 React Native 之上的**框架 / 平台**:一套 JS/TS 项目即可原生运行于 Android、iOS、Web。
是 [[concepts/05-glossary]] 中 "React Native Framework" 的典型代表。

## 定位:框架 vs 库

- **React Native 本身是库/渲染器** —— 提供 Host Component(`<View>`/`<Text>`)与 Fabric / TurboModules /
  JSI / Codegen / Yoga 这套"把 React 渲染到原生视图"的机制(见 [[concepts/01-new-architecture]]),但**不管**
  构建打包、路由、OTA 更新、原生 SDK 接入。
- **Expo 是围绕这个库的工程化平台** —— 补齐"从零到上线一个真实 App"所缺的全部基建。
- 官方现在推荐**新项目用 Framework(Expo)起步**,而非裸初始化,因为裸 RN 把大量原生工程配置留给开发者。

> 类比 Web:React 之于 Next.js ≈ React Native 之于 Expo。

## 核心工具链

| 能力 | 裸 React Native | Expo 提供 |
|---|---|---|
| 脚手架/构建 | 手配 Xcode/Gradle | Expo CLI(`create-expo-app`)+ **EAS Build**(云端构建) |
| 路由导航 | 自装 React Navigation | **Expo Router**(文件式路由) |
| 原生能力 | 自写/链接原生代码 | **Expo SDK**(相机、通知、定位、传感器…) |
| 原生工程管理 | 手改 `ios/`、`android/` | **Prebuild + CNG + Config Plugins**(声明式生成) |
| 空中更新 | 自建 CodePush 类方案 | **EAS Update**(JS bundle OTA) |
| 上手运行 | 需本地原生环境 | **Expo Go** 扫码即在真机跑 |

- **Expo SDK** —— 预封装原生能力模块(Image / Camera / Notifications 等)。
- **EAS(Expo Application Services)** —— 云端服务:EAS Build(构建)、EAS Update(OTA)、EAS Submit
  (商店提交)、EAS Workflows(构建/发布自动化)。
- **Expo Go** —— 扫码即跑的沙盒,"最快上手"但**仅为受限 playground,不适合生产级项目**。
- **Development build** —— 含 `expo-dev-client` 的 debug 构建,比 Expo Go 更完整可靠,支持安装原生库、
  改原生工程;生产级开发的推荐方式。

## Prebuild 与 CNG

- **Prebuild** —— 按需从 app config **生成** `android/`、`ios/` 原生目录;这些目录默认进 `.gitignore`,
  可随时删除再生成。
- **Continuous Native Generation (CNG)** —— 让原生工程"声明式定义于配置、而非手改源码"的机制(类比包管理器
  管理依赖),简化 RN 版本升级。
- **Config Plugins** —— 不直接改原生代码即可修改原生工程配置。
- 取舍:走 CNG(声明式) 或 prebuild 后手改原生目录;二者混用会破坏再生成能力。

## Expo Router

- 面向 **Universal RN 应用**的开源**文件式路由**库:`app/` 目录下的文件自动转为可导航路由,无需手写
  navigator/route;支持 **typed routes**(链接到不存在的目标会报错)。
- **对比 React Navigation**:Expo Router 由目录结构派生路由;React Navigation 在代码中手动定义。
  Expo Router 底层构建于 React Navigation 之上。
- **Universal**:同一套导航结构与组件覆盖 Android / iOS / Web,路由级别仍可用平台特定 API。
- **自动深链**:每个屏幕天然可深链、可分享。

## 待深入

- Expo SDK 与新架构(Fabric/TurboModule)的兼容与开箱状态。
- EAS Update 的 OTA 更新与 Hermes bytecode 分发的交互(见 [[entities/01-hermes]])。
- Expo Router vs 纯 React Navigation 在大型项目的取舍实测。

## 交叉引用

- 术语:[[concepts/05-glossary]]
- 架构:[[concepts/01-new-architecture]]
- 引擎:[[entities/01-hermes]]
- 来源:[[analysis/01-source-backlog]]
