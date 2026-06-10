# Routing Reference

## Table of Contents
- [Setup](#setup)
- [Route Definition](#route-definition)
- [Nested Routes](#nested-routes)
- [Path Params](#path-params)
- [Query Params](#query-params)
- [Navigation](#navigation)
- [Links](#links)
- [Forms](#forms)
- [Route Guards / Protected Routes](#route-guards--protected-routes)
- [Lazy Routes](#lazy-routes)
- [SSR Modes](#ssr-modes)
- [Static Generation (SSG)](#static-generation-ssg)

---

## Setup

```toml
[dependencies]
leptos_router = { version = "0.8" }
```

Wrap your app in `<Router>`:

```rust
use leptos_router::components::{Router, Routes, Route, A};
use leptos_router::path;

#[component]
fn App() -> impl IntoView {
    view! {
        <Router>
            <nav>
                <A href="/">"Home"</A>
                <A href="/about">"About"</A>
            </nav>
            <Routes fallback=|| view! { <p>"Not found."</p> }>
                <Route path=path!("/") view=HomePage />
                <Route path=path!("/about") view=AboutPage />
            </Routes>
        </Router>
    }
}
```

---

## Route Definition

### Path Syntax (in `path!()` macro)

| Pattern | Matches | Example |
|---|---|---|
| `"/"` | Exact root | `path!("/")` |
| `"/about"` | Static path | `path!("/about")` |
| `"/users/:id"` | Dynamic param | `path!("/users/:id")` |
| `"/files/*rest"` | Wildcard (matches rest) | `path!("/files/*rest")` |

### Route Properties

```rust
<Route
    path=path!("/users/:id")
    view=UserPage              // component to render
    ssr=SsrMode::OutOfOrder   // SSR mode (optional)
/>
```

---

## Nested Routes

Parent routes render an `<Outlet/>` where matched child routes appear:

```rust
<Routes fallback=|| view! { <p>"404"</p> }>
    <ParentRoute path=path!("/dashboard") view=DashboardLayout>
        <Route path=path!("/") view=DashboardHome />
        <Route path=path!("/settings") view=DashboardSettings />
        <ParentRoute path=path!("/users") view=UsersLayout>
            <Route path=path!("/") view=UserList />
            <Route path=path!("/:id") view=UserDetail />
        </ParentRoute>
    </ParentRoute>
</Routes>
```

Parent layout component:

```rust
#[component]
fn DashboardLayout() -> impl IntoView {
    view! {
        <div class="dashboard">
            <nav>
                <A href="/dashboard">"Home"</A>
                <A href="/dashboard/settings">"Settings"</A>
                <A href="/dashboard/users">"Users"</A>
            </nav>
            <main>
                <Outlet />
            </main>
        </div>
    }
}
```

For reusable route groups, use `#[component(transparent)]`:

```rust
#[component(transparent)]
fn ContactRoutes() -> impl IntoView {
    view! {
        <ParentRoute path=path!("/contacts") view=ContactList>
            <Route path=path!("/:id") view=ContactDetail />
        </ParentRoute>
    }
}

// Then in your Routes:
<ContactRoutes />
```

---

## Path Params

### Typed Params

```rust
use leptos_router::params::Params;

#[derive(Params, PartialEq)]
struct UserParams {
    id: Option<String>,
    // Multiple params
    tab: Option<String>,
}

#[component]
fn UserPage() -> impl IntoView {
    let params = use_params::<UserParams>();

    let id = move || {
        params.with(|p| p.id.clone().unwrap_or_default())
    };

    view! { <h1>"User: " {id}</h1> }
}
```

### Untyped Params Map

```rust
let params = use_params_map();
let id = move || params.with(|p| p.get("id").cloned().unwrap_or_default());
```

---

## Query Params

```rust
use leptos_router::params::Params;

#[derive(Params, PartialEq)]
struct SearchQuery {
    q: Option<String>,
    page: Option<u32>,
}

#[component]
fn SearchPage() -> impl IntoView {
    let query = use_query::<SearchQuery>();

    let search_term = move || {
        query.with(|q| q.q.clone().unwrap_or_default())
    };

    view! { <p>"Searching: " {search_term}</p> }
}
```

### Query Signal (two-way binding with URL)

```rust
let page = query_signal::<u32, _>("page");
// Reading: page.get() → current page from URL
// Writing: page.set(Some(2)) → updates URL query param
```

---

## Navigation

```rust
use leptos_router::hooks::use_navigate;

let navigate = use_navigate();

// Simple navigation
navigate("/users", Default::default());

// Replace current history entry
navigate("/login", NavigateOptions { replace: true, ..Default::default() });
```

### Redirect (from server functions or route level)

```rust
use leptos_router::components::Redirect;

view! {
    <Redirect path="/login" />
}
```

---

## Links

```rust
// Router-aware link (prevents full page reload)
<A href="/users">"Users"</A>

// With state
<A href="/users" state=User { name: "Alice".into() }>"Users"</A>

// Active class (applied when link matches current route)
<A href="/users" class="nav-link" class:active=true>"Users"</A>
```

---

## Forms

```rust
use leptos_router::components::Form;

view! {
    <Form method=Method::Post action="/api/search">
        <input name="query" />
        <button>"Search"</button>
    </Form>
}
```

`<Form>` from leptos_router provides progressive enhancement — works as a regular HTML form, enhanced with client-side routing when WASM loads.

---

## Route Guards / Protected Routes

```rust
use leptos_router::components::ProtectedRoute;

<Routes fallback=|| view! { <p>"404"</p> }>
    <ProtectedRoute
        path=path!("/admin")
        view=AdminPage
        redirect_path=|| "/login"
        condition=|| {
            // Return true if user can access, false to redirect
            use_context::<AuthState>().map(|auth| auth.is_logged_in()).unwrap_or(false)
        }
    />
</Routes>
```

---

## Lazy Routes

Code-split routes loaded on demand:

```rust
use leptos_router::lazy_route;

<Routes fallback=|| view! { <p>"404"</p> }>
    <Route path=path!("/") view=HomePage />
    <Route
        path=path!("/heavy")
        view=lazy_route!(|| importHeavyModule())
    />
</Routes>
```

---

## SSR Modes

Control server rendering per route:

```rust
use leptos_router::ssr_mode::SsrMode;

<Route path=path!("/") view=Home ssr=SsrMode::OutOfOrder />
<Route path=path!("/about") view=About ssr=SsrMode::InOrder />
<Route path=path!("/api-docs") view=Docs ssr=SsrMode::Async />
<Route path=path!("/blog/:slug") view=BlogPost ssr=SsrMode::Static />
```

| Mode | Behavior | Best for |
|---|---|---|
| `OutOfOrder` | Streams HTML as resolved (default) | Most pages, fastest TTFB |
| `InOrder` | Waits at Suspense boundaries | SEO-critical pages |
| `Async` | Waits for all async work | API docs, fully static content |
| `PartiallyBlocked` | Mixed blocking/streaming | Selective blocking |
| `Static` | Pre-rendered at build time (SSG) | Blog posts, marketing pages |
| `Upgradable` | CSR upgrade | Performance-critical routes |

---

## Static Generation (SSG)

Pre-render routes at build time:

```rust
use leptos_axum::generate_route_list_with_ssg;

let (routes, static_data) = generate_route_list_with_ssg(App);

// In your Axum router:
let app = Router::new()
    .leptos_routes(&leptos_options, routes, App)
    .fallback(leptos_axum::file_and_error_handler)
    .with_state(leptos_options);

// Generate static files
static_data.generate_files("target/site").await;
```
