---
name: leptos
description: Build full-stack reactive web apps with Leptos in Rust. Use this skill whenever the user mentions building web UIs, SPAs, SSR apps, or full-stack websites with Rust, especially when they mention Leptos, signals, reactive programming, WASM, cargo-leptos, or want to build isomorphic web apps with server functions. Also use when the user asks about Rust frontend frameworks, reactive UI, SSR + hydration, nested routing, server functions, or progressive enhancement — if the conversation suggests Leptos is a fit, this skill should be loaded. Covers components, signals, resources, actions, server functions, routing, SSR/hydration, and Axum/Actix integration.
---

# Building Full-Stack Web Apps with Leptos

Leptos is a full-stack, reactive Rust web framework that compiles to WASM for the browser and runs natively on the server. You write one codebase that handles both SSR and client-side interactivity with fine-grained reactivity — no virtual DOM.

## When to read reference files

- `references/reactivity.md` — Signal types, derived values, effects, stores, and the reactive graph. Read this when you need details on signal access patterns, memos, or deeply-nested reactive state.
- `references/routing.md` — Full routing API: nested routes, params, queries, guards, lazy loading, SSG. Read this when you need advanced routing patterns.
- `references/ssr-and-server.md` — SSR modes, cargo-leptos setup, server functions, extractors, responses, and deployment. Read this when building full-stack apps with server-side rendering.
- `references/components-and-views.md` — Component patterns, view macro syntax, control flow components, forms, slots, and directives. Read this for UI building patterns.

---

## Project Setup

### CSR-only (client-side, simplest)

```toml
[dependencies]
leptos = { version = "0.8", features = ["csr"] }
```

Run with [Trunk](https://trunkrs.dev/): `trunk serve --open`

### Full-stack SSR (recommended)

Use [cargo-leptos](https://github.com/leptos-rs/cargo-leptos):

```bash
cargo install --locked cargo-leptos
cargo leptos new --git https://github.com/leptos-rs/start-axum  # or start-actix
rustup target add wasm32-unknown-unknown
cargo leptos watch
```

The SSR Cargo.toml typically looks like:

```toml
[dependencies]
leptos = { version = "0.8", features = ["nightly"] }
leptos_meta = { version = "0.8" }
leptos_router = { version = "0.8" }
leptos_axum = { version = "0.8" }   # or leptos_actix
server_fn = { version = "0.8", features = ["axum"] }
serde = { version = "1", features = ["derive"] }

[package.metadata.leptos]
site-root = "target/site"
site-pkg-dir = "pkg"
style-file = "style/main.scss"
tailwind-input-file = "style/tailwind.css"
```

---

## Core Architecture

Leptos apps are built from three layers:

1. **Reactive layer** — Signals, memos, effects. The fine-grained reactive graph that drives updates.
2. **View layer** — Components and the `view!` macro that declaratively describes the UI.
3. **Server layer** — Server functions, resources, actions. The RPC bridge between client and server.

```
Signals change → reactive graph tracks dependencies → only affected DOM nodes update
                                    ↓
Server functions are automatically compiled into HTTP endpoints (SSR) / fetch calls (CSR)
```

---

## Quick Start — Counter Component

```rust
use leptos::prelude::*;

#[component]
pub fn SimpleCounter(
    /// The starting value
    initial_value: i32,
    /// Amount to change per click
    step: i32,
) -> impl IntoView {
    let (value, set_value) = signal(initial_value);

    view! {
        <div>
            <button on:click=move |_| set_value.set(0)>"Clear"</button>
            <button on:click=move |_| *set_value.write() -= step>"-1"</button>
            <span>"Value: " {value} "!"</span>
            <button on:click=move |_| set_value.update(|v| *v += step)>"+1"</button>
        </div>
    }
}

// Entry point (CSR)
pub fn main() {
    mount_to_body(|| {
        view! { <SimpleCounter initial_value=0 step=1/> }
    })
}
```

---

## Reactivity

### Signals

Signals are the core primitive — a reactive value that tracks who reads it and notifies them on change.

```rust
// Create a signal
let (count, set_count) = signal(0);

// Reading (all track the dependency)
count.get()          // clones the value
count.read()         // returns a ReadGuard
count.with(|v| *v)   // callback with &T

// Writing
set_count.set(5);              // replace value
set_count.update(|v| *v += 1); // mutate in place
*set_count.write() += 1;       // write guard deref
```

### Derived Values

```rust
let (count, set_count) = signal(0);

// Derived signal (recomputes when count changes)
let double = move || count.get() * 2;

// Memo (cached, only recomputes when dependencies change)
let expensive = Memo::new(move |_| {
    // complex computation based on count
    count.get() * 100
});
```

### Effects

```rust
// Runs whenever dependencies change
Effect::new(move |_| {
    logging::log!("count is now {}", count.get());
});
```

### Stores (Deeply Nested State)

For complex nested state, use `Store` and `Field`:

```rust
use reactive_stores::{Patch, Store};

#[derive(Store, Patch)]
struct AppState {
    user: User,
    todos: Vec<Todo>,
}

let store = Store::new(AppState { /* ... */ });
let user_field = store.user();
user_field.name().set("Alice".into());
```

---

## Components

### Defining Components

```rust
#[component]
fn Greeting(
    /// The name to greet (doc comments become prop documentation)
    name: String,
    /// Optional prop with default
    #[prop(default = true)]
    enthusiastic: bool,
    /// Optional prop (no default = Option<T>)
    title: Option<String>,
    /// Callback prop
    on_click: impl Fn(MouseEvent) + 'static,
) -> impl IntoView {
    let punctuation = if enthusiastic { "!" } else { "." };
    view! {
        <div>
            {title.map(|t| view! { <h2>{t}</h2> })}
            <p>"Hello, " {name} {punctuation}</p>
            <button on:click=on_click>"Click me"</button>
        </div>
    }
}
```

### Using Components

```rust
view! {
    <Greeting
        name="World".to_string()
        title=Some("Welcome".to_string())
        on_click=move |_| logging::log!("clicked")
    />
}
```

### Passing Children

```rust
#[component]
fn Card(children: Children) -> impl IntoView {
    view! {
        <div class="card">
            {children()}
        </div>
    }
}

// Usage:
view! {
    <Card>
        <p>"Some content"</p>
    </Card>
}
```

---

## Control Flow

### Conditional Rendering

```rust
view! {
    <Show
        when=move || count.get() > 0
        fallback=|| view! { <p>"No items"</p> }
    >
        <p>"Items: " {count}</p>
    </Show>
}
```

### List Rendering

```rust
let (items, set_items) = signal(vec![1, 2, 3]);

view! {
    <For each=move || items.get() key=|item| *item let:item>
        <p>"Item: " {item}</p>
    </For>
}
```

The `key` function is critical — it lets Leptos diff efficiently without re-rendering unchanged items.

### Error Handling

```rust
view! {
    <ErrorBoundary fallback=|errors| view! {
        <p>"Error: " {format!("{:?}", errors)}</p>
    }>
        // children that might return Result
        {maybe_failing_component()}
    </ErrorBoundary>
}
```

### Async Rendering

```rust
view! {
    <Suspense fallback=|| view! { <p>"Loading..."</p> }>
        {async_content()}
    </Suspense>
}
```

---

## Routing

### Basic Setup

```rust
use leptos_router::{
    hooks::use_navigate,
    components::{Route, Router, Routes, A},
    path,
};

#[component]
fn App() -> impl IntoView {
    view! {
        <Router>
            <nav>
                <A href="/">"Home"</A>
                <A href="/about">"About"</A>
                <A href="/users/:id">"User"</A>
            </nav>
            <main>
                <Routes fallback=|| view! { <p>"Not found."</p> }>
                    <Route path=path!("/") view=HomePage />
                    <Route path=path!("/about") view=AboutPage />
                    <Route path=path!("/users/:id") view=UserPage />
                </Routes>
            </main>
        </Router>
    }
}
```

### Nested Routes

```rust
<Routes fallback=|| view! { <p>"404"</p> }>
    <ParentRoute path=path!("/users") view=UserLayout>
        <Route path=path!("/") view=UserList />
        <Route path=path!("/:id") view=UserDetail />
        <Route path=path!("/new") view=NewUser />
    </ParentRoute>
</ParentRoute>

// UserLayout renders an <Outlet/> where matched child routes appear
#[component]
fn UserLayout() -> impl IntoView {
    view! {
        <div class="users-layout">
            <Outlet />
        </div>
    }
}
```

### Route Params and Query

```rust
use leptos_router::hooks::{use_params, use_query};

#[derive(Params, PartialEq)]
struct UserParams {
    id: Option<String>,
}

#[component]
fn UserPage() -> impl IntoView {
    let params = use_params::<UserParams>();
    let id = move || params.with(|p| p.id.clone().unwrap_or_default());

    view! { <h1>"User: " {id}</h1> }
}
```

### Navigation

```rust
let navigate = use_navigate();
navigate("/users", Default::default());
```

---

## Server Functions

Server functions let you write async Rust that runs on the server but can be called from client code. Leptos generates the HTTP bridge automatically.

### Basic Server Function

```rust
#[server]
pub async fn get_user(id: i32) -> Result<User, ServerFnError> {
    // This runs on the server only
    let pool = get_db_pool();
    let user = sqlx::query_as::<_, User>("SELECT * FROM users WHERE id = ?")
        .bind(id)
        .fetch_one(pool)
        .await?;
    Ok(user)
}
```

### Using Server Functions with Resources

```rust
#[component]
fn UserProfile(id: i32) -> impl IntoView {
    let user = Resource::new(
        move || id,                              // source signal
        move |id| async move { get_user(id).await }  // fetcher
    );

    view! {
        <Suspense fallback=|| view! { <p>"Loading..."</p> }>
            {move || Suspend::new(async move {
                match user.await {
                    Ok(user) => view! { <div>{user.name}</div> }.into_any(),
                    Err(e) => view! { <p>"Error: " {e.to_string()}</p> }.into_any(),
                }
            })}
        </Suspense>
    }
}
```

### Actions (for Mutations)

```rust
#[server]
pub async fn create_todo(title: String) -> Result<i64, ServerFnError> {
    // insert into DB...
    Ok(id)
}

#[component]
fn TodoForm() -> impl IntoView {
    let add_todo = ServerAction::<CreateTodo>::new();

    view! {
        <ActionForm action=add_todo>
            <input type="text" name="title" />
            <button type="submit">"Add"</button>
        </ActionForm>
    }
}
```

`ActionForm` provides progressive enhancement — works without JS, enhanced with JS.

### Custom Server Function Configuration

```rust
#[server(
    input = Rkyv,              // serialization format
    output = Json,             // response format
    endpoint = "custom-path",  // custom URL
    prefix = "/api",           // URL prefix
)]
pub async fn my_function(data: MyData) -> Result<Response, ServerFnError> {
    // ...
}
```

### File Uploads

```rust
#[server]
pub async fn upload_file(
    name: String,
    #[allow(unused)] file: Vec<u8>,  // multipart field
) -> Result<String, ServerFnError> {
    // save file...
    Ok("uploaded".into())
}
```

---

## Full-Stack App Structure (SSR + Hydration)

### Server Entry Point (Axum)

```rust
// src/main.rs (server binary)
use axum::Router;
use leptos::prelude::*;
use leptos_axum::{generate_route_list, LeptosRoutes};

#[tokio::main]
async fn main() {
    let conf = get_configuration(None).await.unwrap();
    let addr = conf.leptos_options.site_addr;
    let leptos_options = conf.leptos_options;
    let routes = generate_route_list(App);

    let app = Router::new()
        .leptos_routes(&leptos_options, routes, App)
        .fallback(leptos_axum::file_and_error_handler)
        .with_state(leptos_options);

    let listener = tokio::net::TcpListener::bind(addr).await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

### Client Entry Point

```rust
// src/lib.rs or src/app.rs
use leptos::prelude::*;

#[component]
pub fn App() -> impl IntoView {
    view! {
        <Router>
            <Routes fallback=|| view! { <p>"404"</p> }>
                <Route path=path!("/") view=HomePage />
            </Routes>
        </Router>
    }
}

// WASM entry point (called from index.html)
#[wasm_bindgen::prelude::wasm_bindgen]
pub fn hydrate() {
    console_error_panic_hook::set_once();
    leptos::mount_to_body(App);
}
```

### SSR Modes

Control how each route renders on the server:

```rust
<Route path=path!("/") view=HomePage ssr=SsrMode::OutOfOrder />
<Route path=path!("/about") view=About ssr=SsrMode::InOrder />
<Route path=path!("/api-docs") view=Docs ssr=SsrMode::Async />
```

- **OutOfOrder** (default) — streams HTML as it resolves, fastest TTFB
- **InOrder** — waits at Suspense boundaries, preserves document order
- **Async** — waits for all async work before sending anything
- **Static** — pre-renders at build time (SSG)

---

## Context and State Sharing

```rust
// Provide state to all descendants
provide_context(my_database_pool);

// Access it anywhere in the component tree
let pool = use_context::<DbPool>().expect("DbPool should be provided");
```

For global state that persists across route changes, use context at the app level. For deeply nested reactive state, use `Store`.

---

## Metadata (Head Tags)

```rust
use leptos_meta::*;

#[component]
fn MyApp() -> impl IntoView {
    provide_meta_context();

    view! {
        <Title text="My App" />
        <Meta name="description" content="A Leptos app" />
        <Stylesheet href="/style.css" />
        <Link rel="icon" href="/favicon.ico" />
        // ... rest of app
    }
}
```

---

## Styling Approaches

Leptos is styling-agnostic. Common patterns:

1. **Tailwind CSS** — most popular, supported by cargo-leptos
2. **Inline classes** — `class="..."` or `class:name=signal` for conditional classes
3. **CSS Modules / Scoped** — via external tools
4. **CSS-in-Rust** — using `style=...` attribute

```rust
view! {
    <div class="flex items-center gap-4">
        <span class:bold=move || active.get()>"Label"</span>
    </div>
}
```

---

## Forms and Progressive Enhancement

Leptos provides three form components that work without JavaScript and are enhanced when WASM loads:

```rust
// ActionForm — submits to a server action
<ActionForm action=create_todo>
    <input name="title" />
    <button>"Create"</button>
</ActionForm>

// MultiActionForm — submits to a MultiAction (supports concurrent submissions)
<MultiActionForm action=add_comment>
    <input name="text" />
    <button>"Comment"</button>
</MultiActionForm>

// ServerForm — custom form submission behavior
```

---

## Common Patterns

1. **`view!` macro** — declarative HTML-like syntax that compiles to efficient DOM operations. Use `class:cond=signal` for conditional classes, `on:event=handler` for events, `prop:name=value` for DOM properties.

2. **Signal access matters** — `{count}` in a view auto-tracks (reactive). `{count.get()}` clones once and is NOT reactive in that position. Prefer `{count}` or closures `move || count.get()` for reactivity.

3. **Server functions are public API** — every `#[server]` function becomes an HTTP endpoint. Validate inputs and handle errors properly.

4. **`Resource` for reads, `Action` for writes** — Resources fetch data (GET-like), Actions perform mutations (POST-like). Both integrate with `<Suspense>`.

5. **Use `<For each=... key=...>` for lists** — always provide a `key` function for efficient keyed diffing.

6. **Components return `impl IntoView`** — the return type is an opaque view tree. Use `.into_any()` when you need to return different types from branches.

7. **CSR vs SSR vs Hydrate** — the same code compiles differently based on features. `cargo leptos watch` handles this automatically for full-stack apps.

8. **`Suspend` for async in views** — wrap async operations in `Suspend::new(async move { ... })` inside `<Suspense>` boundaries.

---

## Feature Checklist

**leptos:**
- `csr` — client-side rendering only
- `ssr` — server-side rendering
- `hydrate` — hydration mode
- `nightly` — function-call syntax for signals (`.get()` becomes `()`)
- `islands` — islands architecture
- `tracing` — tracing integration
- `nonce` — CSP nonce support

**leptos_router:**
- `ssr` — server-side routing support

**server_fn:**
- `axum` — Axum server backend
- `actix` — Actix server backend
- `multipart` — file upload support

**Alternative serialization for server functions:**
`rkyv`, `bitcode`, `cbor`, `msgpack`, `postcard`, `serde-lite`
