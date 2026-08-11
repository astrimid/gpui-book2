# Incremental Rendering in GPUI: Three Optimizations for Performance

You hover over a button in a grid of 100. One button changes color. GPUI redraws all 100.

This is how to fix it, and what the fix reveals about how GPUI works internally.

## The Naive Implementation: 100 Buttons, 100 Redraws

```rust
struct ButtonGrid {
    buttons: Vec<ButtonState>,
}

impl Render for ButtonGrid {
    fn render(&mut self, window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let now = Instant::now();

        div()
            .children(self.buttons.iter_mut().enumerate().map(|(index, button)| {
                // Update animations inline
                if button.update_animation(now) {
                    window.request_animation_frame();
                }

                div()
                    .bg(button.current_color())
                    .on_hover(cx.listener(move |this, is_hovered, _window, cx| {
                        if *is_hovered {
                            this.buttons[index].animate_to_next(Instant::now());
                            cx.notify(); // ← This invalidates the entire grid
                        }
                    }))
                    .child(button.label.clone())
            }))
    }
}

```

When `cx.notify()` invalidates `ButtonGrid`, every button re-renders, re-layouts, and re-paints. The inline animation check fires `request_animation_frame()` for as long as any button is animating.

For 60Hz, this happens every 16ms, which might be too often, as a single frame might easily take twice as much time.

**This causes severe lag.**

## Optimization 1: Give Each Button Its Own Entity

Buttons stored as plain structs in a `Vec` have no rendering boundary. To fix this, make each one an entity. This allows each button to manage its own animation state independently:

```rust
struct ButtonGrid {
    buttons: Vec<Entity<ButtonState>>,
}

impl Render for ButtonState {
    fn render(&mut self, window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        // Button manages its own animation tick
        if self.update_animation(Instant::now()) {
            window.request_animation_frame();
            cx.notify(); 
        }

        div()
            .bg(self.current_color)
            .on_hover(cx.listener(move |this, is_hovered, _window, cx| {
                if *is_hovered {
                    this.animate_to_next(Instant::now());
                    cx.notify(); // Only invalidates this button
                }
            }))
            .child(self.label.clone())
    }
}

impl Render for ButtonGrid {
    fn render(&mut self, _window: &mut Window, _cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .children(self.buttons.iter().map(|b| b.clone().into_element()))
    }
}

```

Now, calling `notify()` on a button only invalidates that specific button. The `ButtonGrid` is no longer responsible for checking animation states. Only the animating button re-renders and re-paints.

## Optimization 2: Cache the Layout

GPUI separates layout from paint internally, but there is no explicit API surface that makes this obvious.

If a component has a fixed size (e.g., 150px × 48px), you can prevent Taffy (the layout engine) from recalculating its layout by using `.cached()`:

```rust
let button_style = StyleRefinement {
    size: SizeRefinement {
        width: Some(px(150.0).into()),
        height: Some(px(48.0).into()),
    },
    ..Default::default()
};

div()
    .children(self.buttons.iter().map(|button_entity| {
        button_entity.clone().cached(button_style.clone()).into_element()
    }))

```

`.cached(style)` is a view-level wrapper. It tells the framework to use the provided `StyleRefinement` directly. Taffy skips the layout measurement phase entirely for this subtree. It reads the dimensions and moves on without recursing.

> **Note:** `request_layout()` does the exact same thing at a lower level.

Combined with optimization 1, only the animating button gets measured. The other 99 use their cached dimensions.

## Optimization 3: The Async Throttling Task

Event handlers typically call `notify()` immediately:

```rust
.on_hover(cx.listener(move |this, is_hovered, _window, cx| {
    if *is_hovered {
        this.animate_to_next(Instant::now());
        cx.notify(); 
    }
}))

```

GPUI actually does its own coalescing under the hood, so multiple `notify()` calls in the same frame won't inherently trigger multiple paints. However, relying on arbitrary UI events to drive animations leads to unpredictable pacing.

Instead of managing animations in the render pass or firing `notify()` on every event, spawn a background task on the `ButtonGrid` to manually throttle and control the animation loop:

```rust
fn setup_animation_loop(grid: Entity<ButtonGrid>, cx: &mut Context<Self>) {
    cx.spawn(|this, mut cx| async move {
        let mut interval = interval(Duration::from_millis(16));
        loop {
            interval.tick().await;
            grid.update(&mut cx, |grid, cx| {
                for button in &grid.buttons {
                    let is_animating = button.read(cx).is_animating;
                    if is_animating {
                        // We update the child entity, not the top-level grid
                        button.update(cx, |b, cx| {
                            if b.update_animation(Instant::now()) {
                                cx.notify();
                            }
                        });
                    }
                }
            });
        }
    }).detach();
}

```

The hover handler on the entity now only mutates data:

```rust
.on_hover(cx.listener(move |this, is_hovered, _window, _cx| {
    this.is_animating = *is_hovered;
}))

```

Notice that the loop calls `cx.notify()` explicitly inside `button.update(cx, ...)`. We notify the individual button entities, not the top-level view. If we only notified the `ButtonGrid`, the cached child entities would not re-render because the framework would assume their internal state had not changed.

By controlling the tick, events are throttled predictably. If ten buttons are hovered within 16ms, they are all processed in the next single loop iteration.

## Three Optimizations, Three Different Problems

| Optimization | What It Solves | Why It Matters |
| --- | --- | --- |
| **Optimization 1 (Entity Boundary)** | Paint cascade | Only animating buttons render. |
| **Optimization 2 (Cached Layout)** | Layout cascade | Only animating buttons are measured by Taffy. |
| **Optimization 3 (Throttled Loop)** | Event cascade | Animation ticks are centralized and predictably paced. |

## The Design Choice in GPUI

A 100-button grid runs at single-digit FPS out of the box. Frameworks like React, SwiftUI, Flutter, or Elm maintain 60 FPS for identical naive animations.

**GPUI's default path requires developers to actively manage entities, `.cached()`, and tick throttling.** This is not highlighted in the documentation, meaning standard interactions drop performance without clear feedback on why.

## Implementation Guide

To maintain 60 FPS and low CPU usage:

* ✅ Use `Vec<Entity<T>>`, **not** `Vec<T>`.
* ✅ Wrap fixed-dimension items in `.cached()` with a `StyleRefinement`.
* ✅ Use a spawned async interval loop to throttle animation state updates, explicitly notifying the child entities rather than the parent view.
