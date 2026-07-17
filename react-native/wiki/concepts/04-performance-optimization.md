---
title: 性能优化(系统方法)
date: 2026-07-09
tags: [performance, profiling, flatlist, flashlist, reanimated, cold-start, re-render]
sources: [2026-07-09-reactnative-dev-performance.md, 2026-07-09-rapidnative-performance-playbook.md]
---

# React Native 性能优化

综合官方 Performance Overview 与实践 playbook,给出一套**先测量、按影响排序**的方法论。底层线程认知见 [[concepts/03-threading-model]]。

## 第一原则:先 profile

> "不要优化任何没出现在 profiler trace 里的东西。"

工具矩阵:
- **Perf Monitor**(Dev Menu)— 分别看 JS / UI frame rate。
- **Hermes Sampling Profiler** — 定位 JS 线程瓶颈。
- **React DevTools Profiler** — 看重渲染模式("Highlight updates when components render" 可显性化 render 风暴)。
- **why-did-you-render** — 自动标记不必要重渲染。
- **Flashlight**(跨提交打分)、**Perfetto**(Android)/ **Instruments**(iOS)平台级 trace。
- **Xcode Allocations / Android Studio Memory Profiler** — 查内存泄漏(diff 前先强制 GC)。

## 性能预算(可作为量化目标 — 数值来自第三方 playbook,需按项目复核)

| 指标 | 目标 |
|------|------|
| 冷启动 TTI | < 2.0s 中端 Android;< 1.2s iPhone 13 |
| 持续滚动 FPS | 58+(99 分位设备) |
| 交互延迟 | < 100ms(点击→反馈) |
| JS 内存工作集 | < 180MB |
| 安装体积 | < 30MB |
| 主 JS bundle | < 4MB |

## 优化优先级(按影响排序)

1. **Profile first**。
2. **启用新架构**(Fabric + TurboModules + JSI)。→ [[concepts/01-new-architecture]]
3. **保持 Hermes 默认**(RN 0.84 起标准引擎)。→ [[entities/01-hermes]]
4. **FlatList → FlashList**(数据密集列表快约 5×)。
5. **消除不必要重渲染**。
6. **动画移到 UI 线程**(Reanimated worklets)。
7. **优化冷启动**(bundle、懒加载、字体、splash)。
8. **修复内存泄漏**(监听器、图片缓存、定时器)。
9. **缓存与并行网络**(TanStack Query、HTTP/2、乐观更新)。

## 关键技术

**减少重渲染**
- 把对象/数组移出 props 或用 `useMemo`;`useCallback` 稳定回调引用。
- 拆分快变/慢变 context;Redux/Zustand 用 `createSelector`(Reselect)。

**列表**
- FlashList 需准确 `estimatedItemSize`。
- FlatList 兜底:`initialNumToRender=10`、`maxToRenderPerBatch=5`、`windowSize=21`、`getItemLayout`(影响最大)。

**动画与图形**
- Reanimated 4 worklets 做影响布局的动画;Gesture Handler 处理手势;`@shopify/react-native-skia` 做自定义渲染。
- 官方要点:`Animated` 开 `useNativeDriver:true`;`LayoutAnimation` 适用非可中断动画;移动带透明度的视图用 `renderToHardwareTextureAndroid`/`shouldRasterizeIOS`;动画图片尺寸用 `transform:[{scale}]` 而非改宽高。

**冷启动**
- `source-map-explorer` 审计 bundle;`InteractionManager.runAfterInteractions` 懒加载埋点/AB/错误上报;CI 产出 Hermes bytecode(核对 `.hbc`);Android 开 ProGuard/R8。

## 官方常见坑速查

- 永远在 **release build** 测性能(`dev=true` 严重拖慢 JS 线程)。
- 生产移除 `console.log`(`babel-plugin-transform-remove-console`)。
- 慢导航转场:用 `InteractionManager` 推迟重活。
- TouchableX 卡顿:把昂贵动作包进 `requestAnimationFrame`。

## 交叉引用

- 线程与帧预算:[[concepts/03-threading-model]]
- 新架构:[[concepts/01-new-architecture]]
- 来源:[[sources/04-official-performance]]、[[sources/06-rapidnative-performance-playbook]]
