# Reactivity Reference

## Table of Contents
- [Signal Types](#signal-types)
- [Reading Signals](#reading-signals)
- [Writing Signals](#writing-signals)
- [Derived Signals](#derived-signals)
- [Memos](#memos)
- [Effects](#effects)
- [Stores (Nested State)](#stores-nested-state)
- [Context](#context)
- [Thread Safety](#thread-safety)

---

## Signal Types

| Type | Description |
|---|---|
| `signal(T)` → `(ReadSignal<T>, WriteSignal<T>)` | Split signal (separate read/write) |
| `RwSignal<T>` | Combined read/write signal |
| `ArcRwSignal<T>` | Thread-safe reference-counted signal |
| `Signal<T>` | Read wrapper (can be signal or derived) |
| `SignalSetter<T>` | Write wrapper |

---

## Reading Signals

| Method | Returns | Reactive? | When to use |
|---|---|---|---|
| `signal.get()` | `T` (cloned) | Yes (tracks) | Most common read |
| `signal.read()` | `ReadGuard<T>` | Yes | Need `&T` without cloning |
| `signal.with(\|v\| ...)` | Return value | Yes | Transform without cloning |
| `signal.with_untracked(\|v\| ...)` | Return value | No | Read without tracking |

In `view!`:
- `{signal}` — auto-tracks, equivalent to `.get()`
- `{move || signal.get()}` — explicit reactive closure
- `{signal.get()}` — evaluated once, NOT reactive

---

## Writing Signals

| Method | Description |
|---|---|
| `set_signal.set(value)` | Replace value entirely |
| `set_signal.update(\|v\| *v += 1)` | Mutate in place via callback |
| `*set_signal.write() += 1` | Write guard with deref |

---

## Derived Signals

A closure that reads signals is automatically reactive:

```rust
let (count, _) = signal(5);
let double = move || count.get() * 2;  // recomputes when count changes
```

Use in views: `{double}` or `{move || double()}`

---

## Memos

Cached derived values — only recomputes when dependencies change:

```rust
let expensive: Memo<i32> = Memo::new(move |_| {
    // complex computation
    count.get() * 100
});
```

Use memos when the derivation is expensive and used in multiple places.

---

## Effects

```rust
// Runs immediately and re-runs on dependency changes
Effect::new(move |_| {
    logging::log!("count is {}", count.get());
});

// RenderEffect — returns a value and can be used to drive DOM updates
let description = RenderEffect::new(move |_| {
    match count.get() {
        0 => "zero".to_string(),
        n => format!("count: {n}"),
    }
});
```

**Avoid writing signals from effects** — this creates circular dependencies. Use derived signals or memos instead.

---

## Stores (Nested State)

For complex nested state where you want fine-grained reactivity on individual fields:

```rust
use reactive_stores::{Store, Patch, Field};

#[derive(Store, Patch)]
struct AppState {
    user: User,
    todos: Vec<Todo>,
    settings: Settings,
}

#[derive(Store, Patch)]
struct User {
    name: String,
    email: String,
}

let store = Store::new(AppState {
    user: User { name: "Alice".into(), email: "alice@example.com".into() },
    todos: vec![],
    settings: Settings::default(),
});

// Access nested fields reactively
let name = store.user().name();
name.set("Bob".into());

// Keyed store for list iteration
#[derive(Store, Patch)]
#[store(key: i32 = |todo| todo.id)]
struct Todo {
    id: i32,
    title: String,
    done: bool,
}
```

---

## Context

Share data down the component tree without prop drilling:

```rust
// Provider (parent or app-level)
provide_context(my_state);

// Consumer (any descendant)
let state = use_context::<MyState>().expect("should be provided");
```

Context is scoped to the reactive `Owner` — it's cleaned up automatically.

---

## Thread Safety

Signals require `Send + Sync` values by default. For types that aren't thread-safe:

| Need | Use |
|---|---|
| Non-`Send` value | `signal_local()` or `RwSignal::new_local()` |
| Client-only async | `LocalResource` instead of `Resource` |
| Non-`Send` context | `provide_context` works with any type |
