# GPUI Architecture & Rendering Pipeline

A comprehensive mental model and technical deep-dive into how GPUI drives frames, tracks state, manages invalidation propagation, and executes its rendering and input pipelines.

---

## Chapter 1: The Hybrid Rendering Model & Invalidation

GPUI combines retained state with immediate-mode drawing. Understanding how GPUI stores data, manages invalidation, and schedules frame execution is key to building performant applications.

### 1. Retained State vs. Immediate-Mode Drawing

If you are coming from traditional retained-mode UI frameworks (such as React, SwiftUI, Flutter, or Qt), GPUI flips your expectations:

**GPUI reconstructs the UI tree and draws vector commands from scratch every frame.**

There is no persistent Document Object Model (DOM) or display list retained in memory between frames. Instead, GPUI decouples state management from rendering across two concepts:

```
┌────────────────────────────────────────────────────────────────────────┐
│ GPUI HYBRID ARCHITECTURE                                               │
├───────────────────────────────────┬────────────────────────────────────┤
│ Retained State (Entities)         │ Immediate Drawing (Everything Else)│
├───────────────────────────────────┼────────────────────────────────────┤
│ • Allocated on the heap inside    │ • Stateless elements (div, img)    │
│   central EntityMap               │   constructed fresh each frame     │
│ • Identified by stable EntityId   │ • Taffy layout tree computed fresh │
│ • Persists across frame redraws   │ • Quads, glyphs, and paths built   │
│ • Mutated via cx.notify()         │   fresh into a new gpui::Scene     │
└───────────────────────────────────┴────────────────────────────────────┘
```

#### Retained State (Entities)

- An `Entity<T>` is a reference-counted handle to application data managed by GPUI's central context.
- Entities persist across frames and carry a globally unique `EntityId`.
- Stateful views implement `Render` on their data struct `T`, allowing their `Entity<T>` handles to enter the UI tree as views.

#### Immediate Drawing (Everything Else)

- Structural builders like `div()`, `button()`, and `img()` are ephemeral value types. They are constructed on the stack during a render pass and discarded immediately after.
- On every active frame, GPUI evaluates views, calculates layout parameters using the Taffy layout engine, and generates vector graphics commands into a clean `gpui::Scene`.
- The `gpui::Scene` is not retained between frames—only low-level GPU assets persist.

### 2. What GPUI Actually Retains

To maintain high frame rates without storing an internal DOM, GPUI isolates persistent memory to three specific areas:

1. **Entity State:** Application data structures residing inside GPUI's central `EntityMap`.
2. **GPU Textures & Glyph Atlases:** Vector glyphs and raster images uploaded to VRAM for reuse across drawing cycles.
3. **Cached Subtree Layout & Paint Output:** Resolved Taffy layout nodes and rendered drawing commands preserved strictly when views explicitly opt in via `.cached()`.

Everything else—element nodes, layout constraints, transient event listeners, and drawing primitive collections—is instantiated during frame evaluation and dropped immediately after submission.

### 3. Invalidation Mechanics & Execution Flow

When application state changes, GPUI coordinates window updates through a precise, push-based invalidation mechanism followed by a top-down execution pass.

```
[State Update] ──► cx.notify() ──► Context::notify() ──► App::notify(entity_id)
                                                           │
                                                           ▼
                                            For each WindowInvalidator tracking this entity:
                                            invalidator.invalidate_view(entity_id, cx)
                                                           │
                                                           ▼
                                            Sets invalidator.dirty = true & inserts EntityId into
                                            WindowInvalidatorInner.dirty_views (flat set, entity-keyed only)
                                                           │
                                                           ▼
[Window Draw Time] ──► Window::draw() ──► invalidate_entities()
                                              │
                                              ▼
                                            Takes the flat dirty_views set from the invalidator,
                                            then for each entity calls mark_view_dirty(entity)
                                              │
                                              ▼
                                            mark_view_dirty() walks the ancestor chain via
                                            view_path_reversed() and populates Window.dirty_views
                                            (a *separate*, ancestor-expanded set), breaking early
                                            once an ancestor is already marked
                                              │
                                              ▼
[Render Execution] ──► draw_roots() ──► Top-down traversal from root
                       • Uncached ViewElements: request_layout() calls
                         render() unconditionally every frame
                       • .cached() ViewElements: prepaint() checks
                         Window.dirty_views (not the invalidator's set) plus
                         cache_key match to decide reuse vs re-render
```

#### Step-by-Step Breakdown of the Invalidation Pipeline

1. **Notification Trigger (`cx.notify()`):** Calling `cx.notify()` forwards to `Context::notify()`, which invokes `App::notify(entity_id)`. GPUI locates all active windows tracking that entity and calls `invalidator.invalidate_view(entity_id, cx)`.

2. **Window Invalidator Recording:** `invalidate_view` sets `dirty = true` on the window's inner state and inserts the entity into `WindowInvalidatorInner.dirty_views` (a flat, entity-keyed set).

3. **Ancestor Expansion (`invalidate_entities()`):** At the start of `Window::draw()`, GPUI drains the invalidator's flat set and runs `mark_view_dirty` for each entity. This method walks the dispatch tree backward (`view_path_reversed`), inserting every ancestor view into `Window.dirty_views` (a separate, ancestor-expanded set). The walk terminates early if an ancestor is already marked, as its parents are guaranteed to be dirty as well.

4. **Execution and Cache Consultation:** GPUI traverses the tree top-down from the roots. Uncached view elements execute their `render()` method unconditionally when layout runs. However, views wrapped in `.cached()` consult `Window.dirty_views` and their cache keys during `prepaint()` to determine whether to reuse previous layout and paint primitives or re-evaluate.

---

## Chapter 2: Frame Execution & The Four-Phase Pipeline

GPUI executes frame updates through a four-phase rendering pipeline. Understanding these phases and their performance implications is essential for maintaining consistent frame timing.

### 1. The Three Sequential Frame Costs

Each frame tick imposes three distinct computational costs on the CPU:

```
┌─────────────────────────────────────────────────────────────────────────┐
│ SEQUENTIAL FRAME COSTS                                                  │
├───────────────────┬─────────────────────┬───────────────────────────────┤
│ 1. User Code      │ 2. Layout Phase     │ 3. Paint & Hit-Testing        │
│    (render())     │    (Taffy Engine)   │    (Scene & Spatial Index)    │
├───────────────────┼─────────────────────┼───────────────────────────────┤
│ Executes view     │ Resolves flexbox/   │ Generates drawing commands,   │
│ logic & builds    │ grid layout trees   │ updates glyphs/quads, and     │
│ element subtrees  │ for element nodes   │ registers spatial hitboxes    │
└───────────────────┴─────────────────────┴───────────────────────────────┘
```

1. **User Code (`render()` Execution):** Evaluates `render()` across active views to build the element tree. Expensive business logic, data filtering, or allocation inside `render()` blocks the pipeline before layout or painting can start.

2. **Layout Calculation (Taffy Engine):** Elements call `request_layout` to register layout nodes and sizing constraints with Taffy (GPUI's Flexbox/Grid layout engine). Taffy computes bounding geometry for the element tree.

3. **Paint, Rasterization & Hit-Testing:** Elements emit vector primitives (quads, text runs, box shadows, clipping masks) into a brand-new `gpui::Scene`. Simultaneously, interactive bounds are indexed in spatial hit-testing structures.

### 2. Request-Driven Execution vs. Free-Running Loops

Unlike game engines that run continuous $60\text{ Hz}$ or $144\text{ Hz}$ event loops, GPUI uses **request-driven rendering**:

- **Idle State:** If no state changes, animations, or OS input events occur, GPUI remains dormant. The window performs no redraws and consumes zero CPU/GPU cycles.

- **Wake Triggers:** GPUI schedules a frame only when requested by:
  - Explicit state mutations (`cx.notify()`).
  - Window refresh requests (`window.refresh()`).
  - Animation frame callbacks (`window.request_animation_frame()`).
  - System interaction events (mouse moves, keypresses, window resizes, timer completions).

### 3. The Four-Phase Rendering Pipeline

When a frame is triggered, GPUI processes the element tree through four sequential phases:

```
┌────────────────────────────────────────────────────────────────────────┐
│ THE FOUR-PHASE PIPELINE                                                │
│                                                                        │
│  [ Phase 1: Layout ]      Calculates element dimensions and positions  │
│          │                via the Taffy Flexbox/Grid engine.           │
│          ▼                                                             │
│  [ Phase 2: Prepaint ]    Finalizes bounds, resolves scroll offsets,   │
│          │                and registers spatial hitboxes.              │
│          ▼                                                             │
│  [ Phase 3: Paint ]       Emits quads, text glyphs, shadows, and paths │
│          │                into a fresh gpui::Scene.                    │
│          ▼                                                             │
│  [ Phase 4: GPU Submit ]  Translates Scene primitives into backend GPU │
│                           commands (Metal, DirectX, wgpu).             │
└────────────────────────────────────────────────────────────────────────┘
```

1. **Layout (`request_layout`):** Measures text bounds and computes layout geometry through Taffy.
2. **Prepaint (`prepaint`):** Finalizes absolute screen coordinates, calculates scroll offsets, and registers spatial hitboxes for interactive components. No visual drawing commands are generated during this phase.
3. **Paint (`paint`):** Traverses element boundaries to emit low-level visual primitives into a new `gpui::Scene`.
4. **GPU Submission:** Serializes the `gpui::Scene` into graphics API calls (Metal on macOS, DirectX on Windows, or Vulkan/wgpu on Linux) and presents the frame to the display server.

### 4. Subtree Caching with `.cached()`

By default, every uncached view in an invalidated window executes `render()`, generates layout nodes, and repaints visual elements. The `.cached()` wrapper provides an opt-in mechanism to skip layout and paint phases for unchanged subtrees:

```rust
// Opting into layout and paint primitive caching for a view handle
self.sidebar_view.clone().cached(style)
```

#### How Caching Works

- **Cache Slot Storage:** When `.cached()` is applied, GPUI stores a cached slot containing the subtree's resolved Taffy layout bounds and emitted `gpui::Scene` primitives.

- **Dirty Check Integration:** During prepaint, GPUI checks whether the cache key matches and whether the target `EntityId` (or any of its ancestors) is absent from `Window.dirty_views` (and `refreshing` is false).

- **Re-evaluation vs. Reuse:**
  - **If dirty:** The cache is bypassed, and GPUI re-runs `render()`, layout calculation, and paint generation for the subtree.
  - **If clean:** GPUI bypasses layout re-computation and paint emission, reusing the stored layout bounds and graphics primitives directly via `reuse_prepaint`.

---

## Chapter 3: Input, Hit-Testing, and Interaction Systems

GPUI routes user input to visual elements using a spatial hit-testing index built during frame construction.

### 1. Prepaint Hitbox Registration

Interactive elements (such as buttons, text fields, and scroll containers) do not maintain persistent event listeners inside a DOM. Instead, hit-testing is rebuilt during the Prepaint phase of every frame:

```
                  PREPAINT PHASE: HITBOX REGISTRATION
┌──────────────────────────────────────────────────────────────────────┐
│ View Element Tree                                                    │
│  └── div().on_click(...)                                             │
│       │                                                              │
│       ▼                                                              │
│ Calculates Absolute Bounding Box: [x: 120, y: 40, w: 80, h: 32]      │
│ Pairs Bounds with EntityId / ElementId                               │
│                                                                      │
│       ▼                                                              │
│ Spatial Hitbox Index (Window Level)                                  │
│ ┌──────────────────────────────────────────────────────────────────┐ │
│ │ Hitbox #104 | Bounds: [120, 40, 200, 72] | EntityId(42)          │ │
│ └──────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────┘
```

#### Registration Steps

1. **Modifier Processing:** When an element applies an interactive modifier (such as `.on_click`, `.on_hover`, or `.on_mouse_down`), GPUI flags the element for hitbox tracking.

2. **Spatial Indexing:** During `prepaint()`, GPUI evaluates absolute layout bounds and registers a spatial hitbox entry containing the calculated bounding rectangle, target `EntityId`, and `ElementId`.

3. **Identity Scoping:** Hitboxes rely on unique `ElementId` paths. Reusing identical `ElementId` values across sibling components can corrupt spatial indexing, causing hover or click events to dispatch to incorrect targets.

### 2. Event Dispatch & Target Resolution

When an OS input event (such as a mouse click, scroll wheel turn, or cursor movement) reaches a window, GPUI processes it using spatial lookup:

```
[OS Mouse Click Event (x: 140, y: 50)]
                  │
                  ▼
   Query Spatial Hitbox Index
                  │
                  ▼
   Identify Top-Most Target Matching Coordinates
   (Hitbox #104 -> EntityId(42))
                  │
                  ▼
   Dispatch Event to Target Listener
```

1. **Spatial Query:** GPUI queries the window's spatial index using the input event's screen coordinates.

2. **Z-Order Resolution:** If multiple hitboxes overlap beneath the cursor, GPUI resolves depth using rendering order, selecting the top-most interactive target.

3. **Event Handler Execution:** GPUI dispatches the event directly to the closure associated with the target's `EntityId`.

### 3. Performance Hazards & Anti-Patterns

Understanding GPUI's hit-testing and invalidation model helps avoid common performance bottlenecks:

#### 1. The Global Refresh Trap

Calling `window.refresh()` or triggering broad `cx.notify()` updates inside cursor movement handlers (like `.on_hover`) forces an unconditional re-render of the window. For hover states, isolate state updates to specific child views so `cx.notify()` invalidates only the relevant component.

#### 2. High-Density Hitbox Trees

Because hitboxes are re-registered during every prepaint pass, building thousands of interactive elements in a single un-virtualized view creates layout and hit-testing overhead. For large collections (such as long file trees or log outputs), use list virtualization to render only visible items.

#### 3. ElementId Collisions in Dynamic Lists

When generating repeating UI elements (such as list items or tab bars), ensure each interactive element carries a unique, deterministic `ElementId` (for example, by using `.id(index)` or `.id(ElementId::Name(...))`). Duplicate IDs within the same parent context corrupt hitbox lookup and event routing.
