# Components and Views Reference

## Table of Contents
- [view! Macro Syntax](#view-macro-syntax)
- [Component Props](#component-props)
- [Children and Slots](#children-and-slots)
- [Control Flow Components](#control-flow-components)
- [Forms](#forms)
- [Parent-Child Communication](#parent-child-communication)
- [NodeRef](#noderef)
- [Directives](#directives)
- [Portal](#portal)
- [Builder Syntax (No Macros)](#builder-syntax-no-macros)

---

## view! Macro Syntax

### Elements and Text

```rust
view! {
    <div>
        <h1>"Title"</h1>
        <p>"Some text " {dynamic_value} " more text"</p>
    </div>
}
```

### Attributes

```rust
view! {
    <div
        class="container"
        id="main"
        style="color: red"
        prop:value=move || input.get()      // DOM property (reactive)
        attr:data-id="42"                    // custom HTML attribute
    >
        "Content"
    </div>
}
```

### Reactive Attributes

```rust
let (active, _) = signal(true);
let (count, _) = signal(5);

view! {
    <div
        class="item"                         // static class
        class:active=move || active.get()    // conditional class
        class:selected=move || count.get() > 3
    >
        "Styled content"
    </div>
}
```

### Event Handlers

```rust
view! {
    <button
        on:click=move |_| logging::log!("clicked")
        on:mouseover=move |_| logging::log!("hover")
        on:submit=move |ev| {
            ev.prevent_default();
            logging::log!("submitted");
        }
    >
        "Click me"
    </button>
}
```

### Spreading Attributes

```rust
let attrs = vec![("id", "my-div"), ("role", "main")];
view! {
    <div ..attrs>"Content"</div>
}
```

### Refs

```rust
let input_ref: NodeRef<html::Input> = NodeRef::new();

view! {
    <input node_ref=input_ref />
    <button on:click=move |_| {
        if let Some(input) = input_ref.get() {
            input.focus().ok();
        }
    }>"Focus"</button>
}
```

---

## Component Props

### Required Props

```rust
#[component]
fn UserCard(name: String, age: i32) -> impl IntoView {
    view! { <div>{name} " is " {age} " years old"</div> }
}
```

### Optional Props

```rust
#[component]
fn UserCard(
    name: String,
    #[prop(optional)]
    bio: Option<String>,
    #[prop(default = "Unknown".to_string())]
    location: String,
) -> impl IntoView {
    view! {
        <div>
            <h3>{name}</h3>
            {bio.map(|b| view! { <p>{b}</p> })}
            <p>"Location: " {location}</p>
        </div>
    }
}
```

### Generic Props (Callbacks)

```rust
#[component]
fn Button(
    children: Children,
    #[prop(into)]
    on_click: impl Fn(MouseEvent) + 'static,
) -> impl IntoView {
    view! {
        <button on:click=on_click>
            {children()}
        </button>
    }
}
```

### Signal Props

```rust
#[component]
fn Counter(
    #[prop(into)]
    value: Signal<i32>,   // Accepts ReadSignal, Signal, or derived
) -> impl IntoView {
    view! { <span>{value}</span> }
}
```

---

## Children and Slots

### Children

```rust
#[component]
fn Card(children: Children) -> impl IntoView {
    view! {
        <div class="card">
            {children()}
        </div>
    }
}

// Usage
view! {
    <Card>
        <p>"Card content"</p>
    </Card>
}
```

### Typed Children

```rust
#[component]
fn Layout(
    header: Children,
    children: Children,
    footer: Children,
) -> impl IntoView {
    view! {
        <header>{header()}</header>
        <main>{children()}</main>
        <footer>{footer()}</footer>
    }
}
```

### Slots

```rust
#[slot]
struct Tab {
    label: String,
    children: Children,
}

#[component]
fn Tabs(tabs: Vec<Tab>) -> impl IntoView {
    // Render tabs with their children
    view! {
        <div class="tabs">
            <For each=move || tabs.clone() key=|tab| tab.label.clone() let:tab>
                <div>
                    <h3>{tab.label.clone()}</h3>
                    {tab.children()}
                </div>
            </For>
        </div>
    }
}
```

---

## Control Flow Components

### `<Show>` — Conditional Rendering

```rust
view! {
    <Show
        when=move || count.get() > 0
        fallback=|| view! { <p>"No items yet"</p> }
    >
        <p>"You have " {count} " items"</p>
    </Show>
}
```

### `<Show when let>` — Pattern Matching

```rust
view! {
    <Show when=move || user.get() let:user>
        <p>"Hello, " {user.name}</p>
    </Show>
}
```

### `<For>` — Keyed List Rendering

```rust
let (items, set_items) = signal(vec![
    Todo { id: 1, title: "Learn Leptos".into() },
    Todo { id: 2, title: "Build app".into() },
]);

view! {
    <For
        each=move || items.get()
        key=|item| item.id
        let:item
    >
        <div>
            <span>{item.title.clone()}</span>
            <button on:click=move |_| {
                set_items.update(|items| items.retain(|i| i.id != item.id));
            }>"Delete"</button>
        </div>
    </For>
}
```

**Always provide `key`** — without it, Leptos can't efficiently diff the list.

### `<Suspense>` — Async Boundary

```rust
view! {
    <Suspense fallback=|| view! { <p>"Loading..."</p> }>
        {move || Suspend::new(async move {
            let data = resource.await;
            view! { <div>{format!("{:?}", data)}</div> }.into_any()
        })}
    </Suspense>
}
```

### `<Transition>` — Pending State

```rust
view! {
    <Transition
        fallback=move || view! { <p>"Loading..."</p> }
    >
        {move || Suspend::new(async move {
            // Shows old data while new data loads
            let data = resource.await;
            view! { <div>{data}</div> }.into_any()
        })}
    </Transition>
}
```

### `<ErrorBoundary>` — Error Handling

```rust
view! {
    <ErrorBoundary fallback=|errors| view! {
        <div class="error">
            <p>"Something went wrong"</p>
            <pre>{format!("{:?}", errors)}</pre>
        </div>
    }>
        {may_fail()}
    </ErrorBoundary>
}
```

### `<Await>` — One-off Async

```rust
view! {
    <Await
        future=fetch_data()
        let:data
    >
        <p>"Got: " {data.to_string()}</p>
    </Await>
}
```

---

## Forms

### ActionForm (Progressive Enhancement)

Submits to a server action. Works without JS, enhanced with JS:

```rust
let create_todo = ServerAction::<CreateTodo>::new();

view! {
    <ActionForm action=create_todo>
        <input type="text" name="title" placeholder="New todo" />
        <button type="submit">"Add"</button>
    </ActionForm>

    // Show loading state
    {move || {
        if create_todo.pending().get() {
            Some(view! { <p>"Creating..."</p> })
        } else {
            None
        }
    }}
}
```

### MultiActionForm

For forms that can submit multiple times concurrently:

```rust
let add_comment = ServerMultiAction::<AddComment>::new();

view! {
    <MultiActionForm action=add_comment>
        <input name="text" />
        <button>"Comment"</button>
    </MultiActionForm>
}
```

### Regular HTML Form

```rust
view! {
    <form on:submit=move |ev| {
        ev.prevent_default();
        let data = get_form_data(&ev);
        // handle manually
    }>
        <input name="name" />
        <button>"Submit"</button>
    </form>
}
```

---

## Parent-Child Communication

Four common patterns:

### 1. Pass WriteSignal

```rust
#[component]
fn Child(set_count: WriteSignal<i32>) -> impl IntoView {
    view! { <button on:click=move |_| set_count.update(|c| *c += 1)>"+1"</button> }
}
```

### 2. Callback Prop

```rust
#[component]
fn Child(on_change: impl Fn(i32) + 'static) -> impl IntoView {
    view! { <button on:click=move |_| on_change(42)>"Change"</button> }
}
```

### 3. Custom Event

```rust
#[component]
fn Child() -> impl IntoView {
    view! { <button on:click=move |_| {}>"Click"</button> }
}

// Parent listens:
view! {
    <Child on:custom_event=move |ev| { /* handle */ } />
}
```

### 4. Context

```rust
// Parent provides
provide_context(my_signal);

// Child accesses
let parent_signal = use_context::<RwSignal<i32>>().unwrap();
```

---

## NodeRef

Reference a DOM element directly:

```rust
let input_ref: NodeRef<html::Input> = NodeRef::new();

view! {
    <input
        node_ref=input_ref
        type="text"
    />
    <button on:click=move |_| {
        if let Some(el) = input_ref.get() {
            el.focus().ok();
        }
    }>"Focus"</button>
}
```

---

## Directives

Custom attribute directives for reusable DOM behavior:

```rust
#[directive(cx)]
fn auto_focus(element: HtmlElement<html::Input>) {
    element.focus().ok();
}

view! {
    <input type="text" use:auto_focus />
}
```

---

## Portal

Render children outside the current DOM tree:

```rust
view! {
    <Portal target="#modal-container">
        <div class="modal">"Modal content"</div>
    </Portal>
}
```

---

## Builder Syntax (No Macros)

For environments where you can't use the `view!` macro:

```rust
use leptos::html;

fn counter() -> impl IntoView {
    let (count, set_count) = signal(0);

    html::div()
        .child(
            html::button()
                .on(ev::click, move |_| set_count.set(0))
                .child("Clear"),
        )
        .child(
            html::span()
                .child(move || count.get()),
        )
}
```
