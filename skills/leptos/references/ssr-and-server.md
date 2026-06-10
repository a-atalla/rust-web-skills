# SSR and Server Functions Reference

## Table of Contents
- [cargo-leptos Setup](#cargo-leptos-setup)
- [Project Structure](#project-structure)
- [Server Entry Point (Axum)](#server-entry-point-axum)
- [Server Entry Point (Actix)](#server-entry-point-actix)
- [Server Functions](#server-functions)
- [Resources](#resources)
- [Actions](#actions)
- [Extractors](#extractors)
- [Responses and Redirects](#responses-and-redirects)
- [File Uploads](#file-uploads)
- [Streaming](#streaming)
- [Custom Serialization](#custom-serialization)
- [Page Load Lifecycle](#page-load-lifecycle)
- [Deployment](#deployment)

---

## cargo-leptos Setup

```bash
cargo install --locked cargo-leptos
cargo leptos new --git https://github.com/leptos-rs/start-axum
cd my-project
rustup target add wasm32-unknown-unknown
cargo leptos watch    # dev server with hot reload
cargo leptos build    # production build
```

### Cargo.toml Configuration

```toml
[package.metadata.leptos]
site-root = "target/site"          # static output dir
site-pkg-dir = "pkg"               # WASM output dir
style-file = "style/main.scss"     # global styles
tailwind-input-file = "style/tailwind.css"  # tailwind input
lib-profile-release = "wasm-release"  # optimize WASM size

[package.metadata.leptos.watch]
watch-additional-files = ["additional_files"]
```

---

## Project Structure

A typical full-stack Leptos project:

```
my-project/
├── src/
│   ├── main.rs          # Server entry point (runs natively)
│   ├── lib.rs           # Shared module, exports app + hydrate
│   ├── app.rs           # Root component, routes
│   └── components/      # Your components
├── style/
│   └── main.css         # Styles
├── Cargo.toml
└── LEPTOS.toml          # (optional) alternative config location
```

The server binary (`main.rs`) runs natively. The client code compiles to WASM and is loaded in the browser for hydration.

---

## Server Entry Point (Axum)

```rust
use axum::Router;
use leptos::prelude::*;
use leptos_axum::{generate_route_list, LeptosRoutes};

#[tokio::main]
async fn main() {
    let conf = get_configuration(None).await.unwrap();
    let addr = conf.leptos_options.site_addr;
    let leptos_options = conf.leptos_options;

    // Generate route list from component tree
    let routes = generate_route_list(App);

    let app = Router::new()
        .leptos_routes(&leptos_options, routes, App)
        .fallback(leptos_axum::file_and_error_handler)
        .with_state(leptos_options);

    let listener = tokio::net::TcpListener::bind(addr).await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

### With Additional Context

```rust
let app = Router::new()
    .leptos_routes_with_context(
        &leptos_options,
        routes,
        {
            let pool = pool.clone();
            move || provide_context(pool.clone())
        },
        App,
    )
    .fallback(leptos_axum::file_and_error_handler)
    .with_state(leptos_options);
```

---

## Server Entry Point (Actix)

```rust
use actix_web::{App, HttpServer};
use leptos::prelude::*;
use leptos_actix::{generate_route_list, LeptosRoutes};

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    let conf = get_configuration(None).await.unwrap();
    let addr = conf.leptos_options.site_addr;
    let leptos_options = conf.leptos_options;
    let routes = generate_route_list(App);

    HttpServer::new(move || {
        App::new()
            .leptos_routes(&leptos_options, routes.clone(), App)
            .app_data(web::Data::new(leptos_options.clone()))
    })
    .bind(addr)?
    .run()
    .await
}
```

---

## Server Functions

### Basic

```rust
#[server]
pub async fn get_items() -> Result<Vec<Item>, ServerFnError> {
    // Runs on the server, callable from client
    Ok(vec![])
}
```

### With Arguments

```rust
#[server]
pub async fn create_item(title: String, done: bool) -> Result<i64, ServerFnError> {
    let pool = get_pool();
    let id = sqlx::query("INSERT INTO items (title, done) VALUES (?, ?)")
        .bind(&title)
        .bind(done)
        .execute(pool)
        .await?
        .last_insert_rowid();
    Ok(id)
}
```

### Custom Configuration

```rust
#[server(
    input = Rkyv,           // input serialization: Rkyv, Json, Cbor, MsgPack, Postcard
    output = Json,          // output serialization
    endpoint = "my-endpoint",  // custom URL path
    prefix = "/api",           // URL prefix
)]
pub async fn custom_fn(data: MyData) -> Result<Response, ServerFnError> {
    Ok(Response { /* ... */ })
}
```

### Custom Error Types

```rust
#[derive(Debug, thiserror::Error)]
pub enum AppError {
    #[error("Not found")]
    NotFound,
    #[error("Database error: {0}")]
    Db(#[from] sqlx::Error),
}

impl FromServerFnError for AppError {
    // implement required methods...
}

#[server]
pub async fn get_item(id: i32) -> Result<Item, AppError> {
    // ...
}
```

### Calling from Client

```rust
// Server functions are called like normal async functions
spawn_local(async move {
    match create_item("My Item".into(), false).await {
        Ok(id) => logging::log!("Created item {id}"),
        Err(e) => logging::log!("Error: {e}"),
    }
});
```

---

## Resources

Resources load async data reactively — they refetch when their source signal changes.

```rust
let id = signal(1);

let user = Resource::new(
    move || id.get(),                           // source signal
    move |id| async move { get_user(id).await } // fetcher
);

// Use with Suspense
view! {
    <Suspense fallback=|| view! { <p>"Loading..."</p> }>
        {move || Suspend::new(async move {
            match user.await {
                Ok(u) => view! { <div>{u.name}</div> }.into_any(),
                Err(e) => view! { <p>"Error"</p> }.into_any(),
            }
        })}
    </Suspense>
}

// Manual refetch
user.refetch();
```

### LocalResource (Client-Only)

For data that doesn't need SSR serialization:

```rust
let data = LocalResource::new(move || {
    async move { fetch_from_api().await }
});
```

### OnceResource

For data that loads once:

```rust
let config = OnceResource::new(async { load_config().await });
```

---

## Actions

Actions wrap async mutations and track loading/error state:

```rust
let add_todo = Action::new(|title: &String| async move {
    create_todo(title.clone()).await
});

// Check status
add_todo.pending()   // Signal<bool>
add_todo.value()     // Signal<Option<Result<T, E>>>
add_todo.input()     // Signal<Option<T>>

// Dispatch
add_todo.dispatch("My Todo".to_string());
```

### ServerAction (Typed Server Function Actions)

```rust
let add_todo = ServerAction::<CreateTodo>::new();
let delete_todo = ServerAction::<DeleteTodo>::new();

// Combine with Resource for automatic refetching
let todos = Resource::new(
    move || (add_todo.version().get(), delete_todo.version().get()),
    |_| async move { get_todos().await },
);
```

---

## Extractors

Use Axum/Actix extractors inside server functions:

```rust
#[server]
pub async fn with_headers(
    extract(headers: axum::http::HeaderMap),
) -> Result<String, ServerFnError> {
    let auth = headers.get("authorization")
        .and_then(|v| v.to_str().ok())
        .unwrap_or("none");
    Ok(auth.to_string())
}

// With state
#[server]
pub async fn with_db(
    extract_with_state(pool: sqlx::SqlitePool),
) -> Result<Vec<Item>, ServerFnError> {
    // pool is injected from Axum state
    sqlx::query_as::<_, Item>("SELECT * FROM items")
        .fetch_all(&pool)
        .await
        .map_err(Into::into)
}
```

---

## Responses and Redirects

```rust
use leptos_axum::{ResponseOptions, redirect};

#[server]
pub async fn login(username: String) -> Result<(), ServerFnError> {
    if valid_credentials(&username) {
        redirect("/dashboard");
        Ok(())
    } else {
        Err(ServerFnError::new("Invalid credentials"))
    }
}

// Set status code and headers
#[server]
pub async fn create_resource(data: String) -> Result<String, ServerFnError> {
    let response = use_context::<ResponseOptions>();
    if let Some(response) = response {
        response.set_status(StatusCode::CREATED);
        response.insert_header("X-Custom", "value");
    }
    Ok("created".into())
}
```

---

## File Uploads

```rust
// Requires server_fn multipart feature
#[server]
pub async fn upload_file(
    name: String,
    description: String,
    file: Vec<u8>,  // multipart binary field
) -> Result<String, ServerFnError> {
    // save file...
    Ok(format!("Uploaded {} bytes", file.len()))
}

// In a form:
view! {
    <ActionForm action=upload_action>
        <input name="name" />
        <input name="description" />
        <input type="file" name="file" />
        <button>"Upload"</button>
    </ActionForm>
}
```

---

## Streaming

```rust
use futures::StreamExt;

#[server]
pub async fn stream_data() -> Result<StreamingText<String>, ServerFnError> {
    let stream = async_stream::stream! {
        for i in 0..100 {
            tokio::time::sleep(Duration::from_millis(100)).await;
            yield format!("Item {i}\n");
        }
    };
    Ok(StreamingText::new(stream.boxed()))
}
```

---

## Custom Serialization

Leptos supports multiple serialization formats for server functions:

```toml
[dependencies]
server_fn = { version = "0.8", features = ["rkyv"] }
# or: features = ["cbor"], ["msgpack"], ["postcard"], ["bitcode"]
```

| Format | Feature | Pros |
|---|---|---|
| `SerdeQs` (default) | default | Human-readable, form-compatible |
| `Json` | default | Universal, debuggable |
| `Rkyv` | `rkyv` | Fastest, zero-copy |
| `Cbor` | `cbor` | Binary, compact |
| `MsgPack` | `msgpack` | Binary, compact |
| `Postcard` | `postcard` | Embedded-friendly, compact |

---

## Page Load Lifecycle (SSR + Hydration)

1. **Server receives request** → matches route
2. **Server renders HTML** → runs components, resolves Suspense boundaries
3. **HTML streamed to browser** → user sees content immediately
4. **WASM loads** → hydrates the existing DOM (adds event listeners, signals)
5. **Client takes over** → further interactions are client-side

The key insight: users see content before WASM loads. WASM adds interactivity on top.

---

## Deployment

### Build

```bash
cargo leptos build --release
```

This produces:
- `target/server/release/my-project` — server binary
- `target/site/` — static files (WASM, JS, CSS, HTML)

### Deploy to Production

The server binary serves both the API and static files. Deploy it like any Rust server:

```bash
# Run the server binary
./target/server/release/my-project
```

### Binary Size Optimization

```toml
[profile.wasm-release]
inherits = "release"
opt-level = 'z'
lto = true
codegen-units = 1

[package.metadata.leptos]
lib-profile-release = "wasm-release"
```
