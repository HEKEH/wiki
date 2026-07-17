# Source Capture — React Native Docs: Architecture Deep-Dive (Fabric / Pipeline / Threading / View Flattening / Glossary)

- URLs:
  - https://reactnative.dev/architecture/fabric-renderer
  - https://reactnative.dev/architecture/render-pipeline
  - https://reactnative.dev/architecture/threading-model
  - https://reactnative.dev/architecture/view-flattening
  - https://reactnative.dev/architecture/glossary
- Captured: 2026-07-09
- Type: Official documentation (Meta / React Native)

---

## Fabric Renderer

Fabric is RN's new rendering system. Development began 2018; by 2021 the FB app was fully backed by it. Purpose: unify render logic in C++, improve host-platform interoperability, unlock new capabilities.

- **C++ core, shared across platforms** — a perf improvement on one platform (e.g. view flattening, originally Android) benefits all.
- **Three trees**: React Element Tree (JS) → React Shadow Tree (C++) → Host View Tree (native).
- **Synchronous layout** eliminates layout "jump" when embedding RN views in host views.
- **Lazy host component initialization** by default; faster startup.
- **Codegen**: JS component declarations are the source of truth → generates C++ structs for props; prop mismatches become build errors.
- **JSI** replaces JSON serialization with direct access to JS values.
- Multi-priority events, synchronous event processing, React Suspense + React 18 concurrent features, easier SSR.

## Render Pipeline: Render → Commit → Mount

Executes for initial render and state updates.

### Phase 1: Render (JS thread, synchronous)
- React runs product logic → **React Element Tree** in JS.
- Renderer synchronously creates **React Shadow Tree** in C++ via JSI.
- Each Host Component → a Shadow Node (composite components like `<MyComponent>` create none).
- Element Tree immutable/temporal (React "fibers"); each fiber stores a C++ pointer to its Shadow Node. Shadow Tree immutable → updates require cloning.

### Phase 2: Commit (async, background thread)
- **Layout calculation**: position/size of each Shadow Node via **Yoga**; platform-dependent components (Text, TextInput) call host measurement.
- **Tree promotion**: promotes new Shadow Tree as "next tree to be mounted"; schedules mount for next UI thread tick. Separates layout calc from mounting.

### Phase 3: Mount (UI thread, synchronous)
1. **Tree diffing** (C++): diff previously-rendered tree vs next tree → atomic ops `createView / updateView / removeView / deleteView`; includes View Flattening.
2. **Tree promotion**: next tree → previously-rendered tree (baseline for next diff).
3. **View mounting**: apply mutations to host views on UI thread. Android: `ViewGroup`, `TextView`…; iOS: `UIView`, `NSLayoutManager`…
- Typical tree 600–1000 nodes → ~200 after view flattening.

### State updates
- **React state update**: structural sharing — only affected nodes + ancestors cloned; unchanged nodes shared. Mount diff yields minimal mutations (e.g. `UpdateView(Node, {backgroundColor:'yellow'})`).
- **C++ state update** (e.g. ScrollView offset): skips Render phase, can originate any thread; clone node with new state, commit with retry on collision.

### Immutability & thread safety
Both Element Tree and Shadow Tree immutable (C++ const correctness) → thread-safe synchronous APIs without locks; structural sharing reduces memory.

| Phase | Location | Sync | Purpose |
|-------|----------|------|---------|
| Render | JS & C++ | sync | Create Element & Shadow Trees |
| Commit | C++ | async (bg) | Calculate layout, promote tree |
| Mount | C++ & Host | sync (UI) | Create/update host views |

## Threading Model

Two primary threads:
- **UI (Main) thread** — only thread that can manipulate host views; high-priority event handling; sync render pipeline in specific scenarios.
- **JS thread** — React render phase + layout calculations; most common path.

Thread safety via immutable data structures (C++ const correctness): every update clones new objects → framework exposes thread-safe synchronous APIs without locks.

Render pipeline scenarios: (1) standard JS-thread render; (2) UI-thread render for high-priority events (whole pipeline sync on UI thread); (3) render phase interrupted by low-priority UI event, continues on JS; (4) interrupted by high-priority event, continues sync on UI thread; (5) C++ state updates on UI thread skip render phase.

## View Flattening

Optimization in Fabric that **merges "Layout-Only" nodes** (views that only apply margin/padding/opacity/backgroundColor to children) to reduce host view hierarchy depth.

- React's composition API → deep Element Trees where most nodes only affect layout.
- Algorithm integrates into the **diffing stage** (no extra CPU): detects Layout-Only nodes, merges their styles into parent, eliminates host views with no visual change.
- Implemented in C++ → benefits all platforms; zero visible change to users.

## Glossary

- **React Element / Element Tree** — plain JS object describing what appears on screen (props, styles, children); JS-only; instantiations of Composite or Host Components.
- **React Shadow Node / Shadow Tree** — created by Fabric; represents a Host Component to mount; holds props (from JS) + layout (x, y, w, h); in Fabric lives in C++.
- **Host Component** — component whose view impl is provided by a host view (`<View>`, `<Text>`; in ReactDOM `<div>`, `<p>`).
- **Host View / Host View Tree** — tree of native views (Android `ViewGroup`/`TextView`, iOS `UIView`); size/location from Yoga `LayoutMetrics`.
- **Fabric Renderer** — lets React render to host views instead of DOM; JS targeting C++ interfaces.
- **JSI (JavaScript Interfaces)** — lightweight API to embed a JS engine in a C++ app; Fabric↔React communication.
- **Yoga / Yoga Tree** — Flexbox layout engine; each Shadow Node typically has a Yoga Node (not mandatory).
- **JNI** — Java Native Interface; Fabric C++ core ↔ Android.
- **React Composite Components** — components whose `render` reduces to other composite/host components.
- **React Native Framework** — uses React paradigm to ship to native targets (e.g. Expo).
- **Host Platform** — the platform embedding RN (Android, iOS, macOS, Windows).
- (TurboModule and Codegen not defined on the glossary page itself; see landing-page capture.)
