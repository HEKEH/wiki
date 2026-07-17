# Source Capture — React Native Docs: About the New Architecture

- URL: https://reactnative.dev/architecture/landing-page
- Captured: 2026-07-09
- Type: Official documentation (Meta / React Native)

---

## What Is the New Architecture?

A comprehensive redesign of React Native's core internals that began in 2018, proven at scale in Meta's production apps by 2024. Experimental opt-in in RN 0.68; **default in 0.76+**. Encompasses both the new framework architecture and bringing it to the open-source ecosystem.

## Main Pillars & Key Components

### 1. Fabric Renderer
- New rendering system replacing the legacy renderer.
- Enables synchronous layout and effects.
- Supports concurrent rendering features from React 18+.

### 2. TurboModules
- Faster JavaScript/Native interfacing; works with JSI for direct memory references. Type-safe, lazily loaded.

### 3. JavaScript Interface (JSI)
- Replaces the asynchronous bridge.
- Lets JavaScript hold references to C++ objects and vice versa.
- Direct method invocation without serialization costs.
- Handles high-bandwidth operations (e.g., VisionCamera processing ~2 GB/sec of frame data).

### 4. Codegen
- Type-safe native module/component interface generation.

### 5. Yoga
- Layout engine, with conformance improvements (v2.0.0+).

## The Three Core Improvements

### 1. Synchronous Layout and Effects
- Legacy `onLayout` may apply state updates after painting → visual jumps.
- New Architecture: synchronous measurement + state update in a single commit via `useLayoutEffect` → no visual jump. (Tooltip positioning example.)

### 2. Concurrent Renderer & React 18+ Features
- Automatic Batching (batches state updates from native event handlers).
- Suspense (data fetching), Transitions, `useTransition` (interruptible updates).
- Example: `startTransition` around a slider's `onValueChange` reduces intermediate renders.

### 3. Fast JavaScript/Native Interfacing (JSI)
- Removed: asynchronous bridge requiring serialization.
- Added: JSI for direct memory references between JS and C++.
- Eliminates serialization overhead; improves native component init and re-rendering.

## The Render Pipeline
1. Render → Commit → Mount (`/architecture/render-pipeline`)
2. Cross-Platform Implementation (`/architecture/xplat-implementation`)
3. View Flattening (`/architecture/view-flattening`)
4. Threading Model (`/architecture/threading-model`)

## What Replaced the Old Bridge

| Aspect | Legacy | New Architecture |
|--------|--------|------------------|
| JS/Native Communication | Asynchronous Bridge (serialization) | JSI (direct memory references) |
| Rendering | Legacy Renderer | Fabric Renderer |
| Native Modules | Static modules | TurboModules (faster, type-safe) |
| Event Loop | Basic | Enhanced event loop model |

## Architecture Section Sub-Pages
- Architecture Overview (`/architecture/overview`)
- Fabric (`/architecture/fabric-renderer`)
- Render, Commit, and Mount (`/architecture/render-pipeline`)
- Cross Platform Implementation (`/architecture/xplat-implementation`)
- View Flattening (`/architecture/view-flattening`)
- Threading Model (`/architecture/threading-model`)
- Bundled Hermes (`/architecture/bundled-hermes`)
- Glossary (`/architecture/glossary`)

## Adoption Guidance
- As of 0.76 the New Architecture is default; opt in immediately unless issues arise.
- Opt-out: Android `newArchEnabled=false` (gradle.properties); iOS `ENV['RCT_NEW_ARCH_ENABLED'] = '0'` (Podfile).
- Not a guaranteed performance win without refactoring to leverage new APIs; primarily future-proofing.
