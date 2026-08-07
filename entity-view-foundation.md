# GPUI State & UI Foundations: Entities, Views, and Integration

## Chapter 1: Entity

An `Entity<T>` is a reference-counted handle to state managed centrally by GPUI. You use entities whenever you need to store application state that persists across frames, reacts to events, or communicates between different parts of your application.

All state inside an `Entity<T>` is owned by `App`. GPUI provides access to this state through types that implement the `AppContext` trait—most commonly `App` and `Context<T>`. Because context wrappers dereference down to `App`, you can create, read, and mutate entities anywhere an `AppContext` handle is available.

> **Note:** Other specialized types also implement `AppContext` or wrap its methods for background threads and asynchronous tasks; see the async GPUI documentation for details.

When an entity's inner type `T` implements the `Render` trait, the `Entity<T>` handle itself implements `View` and can be passed directly into the element tree.

### Creating an Entity

To allocate state inside GPUI, pass an initialization closure to `cx.new()`. This returns a strong, reference-counted `Entity<T>` handle while storing the underlying `T` inside GPUI.

```rust
{{ #include snippets/creating_a_entity.rs }}
```

### Reading an Entity

To inspect an entity's state without mutating it, use `entity.read(cx)` or `cx.read(&entity)`. This borrows a shared reference (`&T`) to the underlying data for the duration of the closure or call.

```rust
{{ #include snippets/reading_a_entity.rs }}
```

### Updating an Entity

To mutate an entity's state, use `entity.update(cx, |state, cx| { ... })`. Inside the closure, you receive a mutable reference (`&mut T`) to the inner state along with a mutable context handle. If the mutation changes UI state, calling `cx.notify()` inside the update closure marks the entity as dirty so GPUI re-renders any dependent views on the next frame.

```rust
{{ #include snippets/updating_a_entity.rs }}
```

### Downgrading an Entity

Because `Entity<T>` is a strong handle, holding mutual `Entity<T>` references across components creates reference cycles that prevent memory from being freed. To break cyclic references—such as in callbacks, observers, or parent-child back-references—call `entity.downgrade()` to convert the handle into a `WeakEntity<T>`.

A `WeakEntity<T>` does not prevent the underlying entity from being dropped. To access its data later, call `.upgrade()` to temporarily obtain a strong `Entity<T>`.

```rust
{{ #include snippets/downgrading_a_entity.rs }}
```

---

## Chapter 2: Rendering (Render, RenderOnce, and View)

In this chapter, you will learn how to compose elements to build user interfaces in GPUI.

Architecturally, GPUI combines retained state with immediate-mode drawing. At the lowest level, GPUI relies on the `Element` trait and its `paint()` method to emit raw vector graphics and backend GPU drawing commands. However, for constructing application UI and component trees, GPUI provides three primary high-level abstractions:

- **`Render`** for persistent, stateful entities
- **`RenderOnce`** for ephemeral, stateless components
- **`View`** as the underlying trait that unifies both under a single interface

> **Note:** This chapter focuses on these three high-level building blocks; lower-level direct painting with `Element` and `paint` is covered in later chapters.

### Render

The `Render` trait is used to build a stateful view. A view pairs application state with UI generation, backed by an `Entity<T>` stored in GPUI's global application state.

- **Lifecycle:** Persistent. The underlying `Entity<T>` lives across multiple frames.
- **Method Signature:** Takes a mutable reference (`&mut self`), allowing the view to update its internal state during render passes.
- **Reactivity & Invalidation:** Calling `cx.notify()` inserts the entity into a flat set of modified views and sets a window-wide dirty flag. When the window redraws, GPUI traverses top-down from the root, and `render()` executes unconditionally for every uncached view in the tree. The framework does not prune uncached branches during the `render()` pass; ancestor path marking occurs specifically to inform the `.cached()` opt-in reuse check.

**Implementation**

```rust
{{ #include snippets/render.rs }}
```

### RenderOnce

The `RenderOnce` trait is used to create a stateless, ephemeral component.

> **Note on Naming:** The name `RenderOnce` can be misleading. It does not mean "this component renders only once when the app starts," nor does it mean "it renders once per frame." The *Once* signifies its single-pass object lifetime: the component is constructed on the fly during a render pass, executed once, and immediately dropped. It functions like a pure, stateless functional component (similar to a controlled component in React). Because it is ephemeral, it cannot handle internal state or events directly—it is entirely driven by props passed down from its parent. Stateful or uncontrolled components must use `Render` instead.

- **Lifecycle:** Ephemeral. Allocated on the stack during the render pass and discarded immediately after.
- **Method Signature:** Takes `self` by value rather than a mutable reference, operating like a read-only parameter passed down by the caller during construction.

**Implementation**

```rust
{{ #include snippets/render_once.rs }}
```

### View

While `Render` manages persistent entities and `RenderOnce` handles ephemeral components, `View` is the underlying unified trait that powers GPUI's high-level rendering engine:

```rust
pub trait View {
    fn entity_id(&self) -> Option<EntityId>;
    fn render(self, window: &mut Window, cx: &mut App) -> impl IntoElement;
}
```

GPUI standardizes how components enter the element tree by providing two key blanket implementations:

#### `impl<T: Render> View for Entity<T>`

The `Entity<T>` handle itself is the view. Because an `Entity<T>` is a reference-counted handle, cloning it and passing it into `.child()` passes the handle by value. Its `entity_id()` method returns `Some(EntityId)`.

#### `impl<T: RenderOnce> View for T`

The bare struct itself is the view. It is passed by value, executed once, and dropped. Its `entity_id()` method returns `None`.

All views converge inside `ViewElement<V: View>`, which inspects `entity_id()` to decide whether to open a reactive/caching boundary or evaluate the element as a plain layout subtree.

### Manual View Implementation

You can manually implement `View` on a custom struct to build reusable composite widgets (such as text inputs, dropdowns, or sliders). This creates a hybrid pattern: you construct the widget inline on the fly with ephemeral props, while binding it to a persistent backing entity for state.

#### The Mental Model: How the Pieces Connect

1. **Parent Entity (ParentView):** Owns a handle to the widget's persistent state as a field (`input_handle: Entity<InputState>`).
2. **Backing Entity (`Entity<InputState>`):** Retains long-lived state across frames (text buffer, selection, focus state).
3. **Custom View Struct (`MyCustomInput`):** An ephemeral wrapper constructed on the stack during rendering and consumed by value in `render(self, ...)`.
4. **No entity is created by the wrapper:** The wrapper struct does not allocate its own state; it receives an existing `Entity<T>` created in a higher context.
5. **Role of `entity_id()`:** Exposes `Some(self.backing_entity.entity_id())` so GPUI's layout and paint caching engines treat this widget with the reactive identity of the backing entity.

**Example**

```rust
// --- 1. CALL SITE (Inside ParentView's render method) ---
impl Render for ParentView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div().child(MyCustomInput {
            // Pass the cloned handle to Entity #2 (persistent state)
            backing_entity: self.input_handle.clone(), 
            // Pass ephemeral UI configuration (props)
            placeholder: "Search...", 
        })
    }
}

// --- 2. WIDGET DEFINITION & VIEW IMPLEMENTATION ---
struct MyCustomInput {
    backing_entity: Entity<InputState>,
    placeholder: &'static str,
}

impl gpui::View for MyCustomInput {
    fn entity_id(&self) -> Option<EntityId> {
        Some(self.backing_entity.entity_id())
    }

    fn render(self, window: &mut Window, cx: &mut App) -> impl IntoElement {
        // Access persistent state via the backing entity handle
        self.backing_entity.update(cx, |state, cx| {
            div()
                .child(self.placeholder)
                .child(state.text.clone())
        })
    }
}
```

### Composition: What Can a View Contain?

A view's render method returns an element tree (`impl IntoElement`) that can freely mix and match different building blocks:

- **Basic layout primitives:** Core elements like `div()`, `span()`, and images that emit Taffy layout nodes and drawing commands (backed by lower-level `Element` implementations).
- **Stateless components (`RenderOnce`):** Ephemeral UI helpers, buttons, or badges.
- **Stateful views or widgets (`Entity<T>`, `AnyView`, or custom `View` types):** Full views nested via `.child()`. During a window redraw, GPUI evaluates their output top-down, allowing subtrees to opt into layout and paint primitive caching via `.cached()`.

---

## Chapter 3: Bridging State and UI (Entities and Views)

Up to this point, we have looked at `Entity<T>` as GPUI's engine for managing persistent application state, and `View` as the interface for rendering UI elements.

This chapter explains how these two systems connect, why GPUI decouples them, and how Rust's ownership model shapes the architecture.

### 1. The Architectural Shift: Rust vs. Traditional OOP

To understand how GPUI bridges state and UI, it helps to contrast it with traditional OOP frameworks (like Qt, WPF, or UIKit).

In OOP frameworks, a widget is a monolithic class that inherits from a base `UIElement`. It holds its own state, handles its own event dispatch, manages pointers to child widgets, and holds parent pointers to trigger callbacks. In Rust, cyclic parent-child references (`&mut`) break aliasing rules and lead to borrow checker fights.

GPUI avoids this by cleanly separating three concerns that OOP conflates:

- **Domain Data (`struct T`):** Plain, passive Rust structs that hold application state. They do not inherit from a base `Widget` class or manage GPU drawing logic.
- **Identity & Storage (`Entity<T>`):** Light, reference-counted handles stored in GPUI's central `EntityMap`.
- **Behavior & Rendering (`Render` / `View`):** Immediate-mode functions that consume or borrow data to emit an element tree description.

```
┌────────────────────────────────────────────────────────────────────────┐
│ OOP (Traditional)                                                      │
│  [Monolithic Class] = State + Parent/Child Pointers + Draw Methods     │
└────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────┐
│ GPUI (Idiomatic Rust)                                                  │
│  1. Storage & Identity : Entity<T> (Reference-counted handle)          │
│  2. Application State  : struct T (Plain Rust data)                    │
│  3. UI Presentation    : impl Render for T (Pure transformation)       │
└────────────────────────────────────────────────────────────────────────┘
```

By placing persistent state in GPUI's central context and passing cheap `Entity<T>` handles through the UI tree, GPUI completely sidesteps Rust's aliasing restrictions.

### 2. The Blanket Bridge

Because data and storage are decoupled, GPUI uses a blanket implementation to turn entity handles into renderable views:

```rust
// Blanket implementation in GPUI core
impl<T: Render> View for Entity<T> {
    fn entity_id(&self) -> Option<EntityId> {
        Some(self.entity_id())
    }

    fn render(self, window: &mut Window, cx: &mut App) -> impl IntoElement {
        // GPUI extracts &mut T from the central EntityMap and calls T::render
    }
}
```

This blanket implementation provides a clean interface:

- **`T` (Your Struct):** Defines state and implements `Render::render(&mut self, ...)` to describe elements.
- **`Entity<T>` (The Handle):** Holds unique identity (`EntityId`), reference counting, and reactive signaling.
- **`View` (The Interface):** Allows parent elements to accept `entity.clone()` by value in `.child(...)` without copying the underlying state struct `T`.

### 3. The Reactive Render Loop

The connection between an `Entity<T>` and its visual representation operates as a push-based reactive loop:

```
[User Action / Event] ──► entity.update(cx, |state, cx| { ...; cx.notify(); })
                                                                │
                                                                ▼ (marks EntityId dirty)
[Next Frame Redraw] ◄── GPUI calls T::render(&mut self) ◄── [Window Invalidation]
```

1. **Mutation:** State changes inside an `entity.update(...)` closure.
2. **Invalidation:** Calling `cx.notify()` flags the entity's `EntityId` as dirty in the window's context.
3. **Scheduling:** GPUI requests a new frame draw from the windowing subsystem.
4. **Execution:** During frame layout, GPUI encounters the dirty `Entity<T>`. It borrows `&mut T` from the internal `EntityMap` and calls `Render::render(&mut self, window, cx)`.
5. **Reconciliation:** The render method returns an element tree (`impl IntoElement`), which GPUI diffs, computes layout for, and paints.

### 4. Two Trees: Ownership vs. Composition

A key concept in GPUI is distinguishing between the Rust Ownership Tree and the Element Tree.

#### The Ownership Tree (Rust Data Flow)

This is where state actually lives in memory. Parents hold `Entity` handles as struct fields.

```rust
pub struct Workspace {
    sidebar: Entity<Sidebar>, // Shared handle owned by Workspace
    editor: Entity<Editor>,   // Shared handle owned by Workspace
}
```

#### The Element Tree (Ephemeral Frame Construction)

This tree is generated on every frame. When `Workspace::render` executes, it passes clones of its entity handles into layout containers:

```rust
impl Render for Workspace {
    fn render(&mut self, _window: &mut Window, _cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .flex()
            .child(self.sidebar.clone()) // Clones Entity handle (cheap), NOT Sidebar data
            .child(self.editor.clone())  // Clones Entity handle (cheap), NOT Editor data
    }
}
```

- **Removing from the Element Tree:** Omitting `self.sidebar` from a conditional `render()` branch hides it visually, but the `Sidebar` state remains alive in memory because `Workspace` still holds `self.sidebar` as a struct field.
- **Removing from Memory:** The `Sidebar` entity is destroyed only when `Workspace` drops `self.sidebar` (or the last `Entity<Sidebar>` handle in the app is dropped).

### 5. Summary Pattern: Bringing It All Together

Here is a complete, minimal example showing an entity modifying its state, notifying GPUI, and being rendered as a view inside a parent:

```rust
use gpui::*;

// 1. STATE DEFINITION
pub struct Counter {
    count: usize,
}

impl Counter {
    pub fn increment(&mut self, cx: &mut Context<Self>) {
        self.count += 1;
        cx.notify(); // Signal GPUI that this view needs a re-render
    }
}

// 2. RENDER IMPLEMENTATION
impl Render for Counter {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let count = self.count;
        
        div()
            .flex()
            .gap_2()
            .child(format!("Count: {}", count))
            .child(
                div()
                    .child("+ Add")
                    .on_mouse_down(MouseButton::Left, cx.listener(|this, _event, _window, cx| {
                        this.increment(cx);
                    }))
            )
    }
}

// 3. COMPOSITION IN PARENT VIEW
pub struct MainApp {
    counter: Entity<Counter>,
}

impl MainApp {
    pub fn new(cx: &mut Context<Self>) -> Self {
        Self {
            counter: cx.new(|_| Counter { count: 0 }),
        }
    }
}

impl Render for MainApp {
    fn render(&mut self, _window: &mut Window, _cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .size_full()
            .child("My Application")
            // Pass Entity<Counter> directly as a child because Entity<T: Render> implements View
            .child(self.counter.clone()) 
    }
}
```
