# Source Capture — RapidNative: RN Performance Optimization 2026 Playbook

- URL: https://www.rapidnative.com/blogs/react-native-performance-optimization-2026-playbook
- Captured: 2026-07-09
- Type: Practitioner playbook (third-party; numbers to verify independently)

---

## Performance targets (numeric budgets)

- Cold start (TTI): < 2.0s mid-tier Android (Pixel 6a); < 1.2s iPhone 13.
- Sustained scroll FPS: 58+ on 99th-percentile device.
- Interaction latency: < 100ms tap → visual feedback.
- JS memory working set: < 180MB (normal-complexity app).
- Install size: < 30MB initial APK/IPA.
- Main JS bundle: < 4MB.

## Optimization priorities (in order)

1. **Profile first** — Hermes Sampling Profiler, React DevTools, Flashlight, Perfetto/Instruments.
2. **Enable New Architecture** (Fabric + TurboModules + JSI) — claims ~40× bridge latency, ~35% rendering, ~25% memory.
3. **Keep Hermes defaults** (standard JS engine since RN 0.84).
4. **FlatList → FlashList** — "5× faster on data-heavy lists".
5. **Eliminate unnecessary re-renders** — memoization, stable props, context splitting.
6. **Move animations to UI thread** — Reanimated 4 worklets.
7. **Optimize cold start** — bundle size, lazy imports, font loading, splash strategy.
8. **Fix memory leaks** — listeners, image caches, timers.
9. **Cache & parallelize network** — TanStack Query, HTTP/2, optimistic updates.

## Key techniques

- Re-render: lift objects/arrays out of props or `useMemo`; `useCallback`; split fast/slow-changing contexts; `createSelector` (Reselect) for Redux/Zustand.
- Lists: FlashList needs accurate `estimatedItemSize`; FlatList fallback `initialNumToRender=10`, `maxToRenderPerBatch=5`, `windowSize=21`, `getItemLayout` (highest impact).
- Animation/graphics: Reanimated 4 for layout-affecting animations; Gesture Handler for gestures; Skia (`@shopify/react-native-skia`) for custom rendering.
- Cold start: `source-map-explorer` to audit bundle; lazy-load analytics/AB/error reporters via `InteractionManager.runAfterInteractions`; build Hermes bytecode in CI (verify `.hbc`); ProGuard/R8 on Android.

## Measurement workflow

"Don't optimize anything that doesn't appear in a profiler trace."
- Hermes Sampling Profiler (JS bottlenecks), React DevTools Profiler (re-render patterns; "Highlight updates"), Flashlight (score across commits), Perfetto/Instruments (platform tracing), why-did-you-render, Xcode Allocations / Android Studio Memory Profiler.
- Capture `performance.now()` from index to first screen `onLayout`; log per release; force GC before memory diff.
