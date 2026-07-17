# Source Capture — React Native Docs: Performance Overview

- URL: https://reactnative.dev/docs/performance
- Captured: 2026-07-09
- Type: Official documentation

---

## Frame rate fundamentals (60 FPS target)

Target 60 fps → **16.67ms per frame**. Dropped frames make UI feel unresponsive.

Two frame rates to monitor:
- **JS frame rate (JS thread)** — runs the React app, API calls, touch processing, state. Updates batched to native at end of each event-loop iteration. A complex re-render taking 200ms = ~12 dropped frames → JS animations and touch freeze.
- **UI frame rate (main thread)** — native animations (e.g. React Navigation native stack), ScrollView scrolling; runs independent of JS thread, more resilient.

## Common sources of performance problems

1. **Dev mode (`dev=true`)** — degrades JS thread; always test perf in release builds.
2. **console.log** — major JS-thread bottleneck; use `babel-plugin-transform-remove-console` in production.
3. **FlatList on large lists** — implement `getItemLayout` to skip measurement; alternatives FlashList, Legend List.
4. **Heavy JS work during animations** ("slow navigator transitions") — defer with `InteractionManager`; use `LayoutAnimation`; use `Animated` with `useNativeDriver: true`.
5. **Moving views drops UI FPS** (alpha compositing) — Android `renderToHardwareTextureAndroid`, iOS `shouldRasterizeIOS`; profile, overuse costs memory.
6. **Animating image size** — iOS re-crops each change; use `transform:[{scale}]` instead.
7. **Unresponsive TouchableX** — heavy re-renders delay touch; wrap expensive action in `requestAnimationFrame`.

## Tools & guides

- **Perf Monitor** (Dev Menu) — JS + UI frame rates.
- Linked guides: Profiling, Speeding up Build phase, Optimizing FlatList Configuration, Optimizing JavaScript loading, Animations, Debugging.

## Philosophy

RN aims to handle optimizations automatically; manual intervention only where RN can't determine the best approach.
