---
name: poem-openapi
description: Build type-safe REST APIs with the Poem web framework and poem-openapi for Rust. Use this skill whenever the user mentions building REST APIs, HTTP services, or OpenAPI/Swagger-documented APIs in Rust, especially when they mention poem, poem-openapi, or want auto-generated API documentation. Also use when the user asks about Rust web frameworks, CRUD APIs, JWT/API-key authentication in Rust, file upload endpoints, or server-sent events — if the conversation suggests poem is a fit, this skill should be loaded. Covers route definition, derive macros (Object, ApiResponse, SecurityScheme, Multipart, Union, Tags), request/response types, middleware, error handling, and the OpenApiService builder pattern with Swagger UI / Scalar / Redoc.
---

# Building REST APIs with Poem and poem-openapi

This skill guides you through building production-ready REST APIs in Rust using the **poem** web framework with **poem-openapi** for automatic OpenAPI v3 specification generation.

Poem is a full-featured, async web framework built on hyper 1.0. The `poem-openapi` crate adds declarative, type-safe OpenAPI spec generation through derive macros — your Rust type signatures *become* the spec.

## When to read reference files

- `references/derive-macros.md` — Full attribute tables for all derive macros (OpenApi, Object, ApiResponse, SecurityScheme, Multipart, Union, Enum, NewType, Tags, ApiRequest, Webhook). Read this when you need to know exact attribute syntax.
- `references/poem-core.md` — Poem core framework reference: routing, middleware, extractors, listeners, error handling. Read this when you need poem-specific patterns outside of OpenAPI.
- `references/auth-patterns.md` — Authentication patterns: Basic auth, API keys, Bearer tokens, OAuth2, optional auth, multiple schemes. Read this when adding authentication.
- `references/payload-types.md` — All payload and parameter types: Json, PlainText, Html, Binary, Upload, Attachment, EventStream, Path, Query, Header, Cookie. Read this when choosing request/response types.

---

## Project Setup

### Cargo.toml

```toml
[dependencies]
poem = "3"
poem-openapi = { version = "5", features = ["swagger-ui"] }
tokio = { version = "1", features = ["macros", "rt-multi-thread"] }
serde = { version = "1", features = ["derive"] }
```

Common features for `poem-openapi`: `swagger-ui`, `scalar`, `redoc`, `rapidoc`, `chrono`, `uuid`, `email`, `hostname`, `sonic-rs`.

Common features for `poem`: `websocket`, `multipart`, `tempfile`, `compression`, `cookie`, `session`, `rustls`, `static-files`, `csrf`, `i18n`, `acme`.

---

## Core Architecture

The mental model is simple: you define a struct with an `#[OpenApi]` impl block. Each method becomes an API operation. The type signatures of your parameters and return types define the OpenAPI spec automatically.

```
User defines:  struct + #[OpenApi] impl with typed methods
         ↓
poem-openapi generates:  OpenAPI v3 spec + poem routes
         ↓
OpenApiService wraps:    everything into a poem Endpoint
         ↓
poem Route mounts:       the service + UI + middleware
```

### The Three-Layer Pattern

Every poem-openapi project has these layers:

1. **Types layer** — `#[derive(Object)]` structs for request/response bodies, `#[derive(Enum)]` for enum types, `#[derive(Union)]` for polymorphic types
2. **API layer** — `#[OpenApi]` impl block with `#[oai(path, method)]` handlers
3. **Server layer** — `OpenApiService` builder + poem `Route` + `Server`

---

## Quick Start — Minimal API

```rust
use poem::{listener::TcpListener, Route, Server};
use poem_openapi::{param::Query, payload::PlainText, OpenApi, OpenApiService};

struct Api;

#[OpenApi]
impl Api {
    #[oai(path = "/hello", method = "get")]
    async fn hello(&self, name: Query<Option<String>>) -> PlainText<String> {
        match name.0 {
            Some(name) => PlainText(format!("hello, {name}!")),
            None => PlainText("hello!".to_string()),
        }
    }
}

#[tokio::main]
async fn main() -> Result<(), std::io::Error> {
    let api_service = OpenApiService::new(Api, "My API", "1.0")
        .server("http://localhost:3000/api");
    let ui = api_service.swagger_ui();

    Server::new(TcpListener::bind("0.0.0.0:3000"))
        .run(Route::new().nest("/api", api_service).nest("/", ui))
        .await
}
```

---

## Defining Data Types

### Request/Response Objects (`#[derive(Object)]`)

```rust
use poem_openapi::Object;

#[derive(Object)]
struct CreateUser {
    #[oai(validator(max_length = 64))]
    name: String,
    #[oai(validator(max_length = 255))]
    email: String,
    /// Password is write-only — it appears in requests but never in responses
    #[oai(write_only)]
    password: String,
}

#[derive(Object)]
struct User {
    #[oai(read_only)]  // Set by server, not by client
    id: i64,
    name: String,
    email: String,
}

/// For partial updates — all fields optional
#[derive(Object)]
struct UpdateUser {
    name: Option<String>,
    email: Option<String>,
}
```

Key field attributes: `read_only`, `write_only`, `validator(...)`, `default`, `rename`, `skip`, `flatten`, `deprecated`.

### Enums (`#[derive(Enum)]`)

```rust
use poem_openapi::Enum;

#[derive(Enum)]
enum Status {
    Active,
    Pending,
    Archived,
}
```

### Union Types (`#[derive(Union)]`)

For polymorphic types mapped to OpenAPI `oneOf`:

```rust
#[derive(Object)]
struct Dog { breed: String }
#[derive(Object)]
struct Cat { indoor: bool }

#[derive(Union)]
#[oai(discriminator_name = "type")]
enum Pet {
    Dog(Dog),
    Cat(Cat),
}
```

### Newtype Wrappers (`#[derive(NewType)]`)

```rust
use poem_openapi::NewType;

#[derive(NewType)]
struct UserId(i64);
```

---

## API Responses

### Simple Returns

A handler can return plain payload types for the happy path:

```rust
#[oai(path = "/users", method = "get")]
async fn list(&self) -> Json<Vec<User>> {
    Json(vec![])
}
```

### Multi-Status Responses (`#[derive(ApiResponse)]`)

For endpoints with multiple possible status codes, define a response enum:

```rust
use poem_openapi::{ApiResponse, payload::{Json, PlainText}};

#[derive(ApiResponse)]
enum GetUserResponse {
    #[oai(status = 200)]
    Ok(Json<User>),

    #[oai(status = 404)]
    NotFound(PlainText<String>),
}
```

Use it:

```rust
#[oai(path = "/users/:id", method = "get")]
async fn get(&self, id: Path<i64>) -> GetUserResponse {
    match find_user(id.0).await {
        Some(user) => GetUserResponse::Ok(Json(user)),
        None => GetUserResponse::NotFound(PlainText("not found".into())),
    }
}
```

### Error as a Response Type

For endpoints that can fail with typed errors, return `Result<SuccessResponse, ErrorResponse>`:

```rust
#[derive(ApiResponse)]
enum CreateResponse {
    #[oai(status = 201)]
    Created(Json<i64>),
}

#[derive(ApiResponse)]
enum CreateError {
    #[oai(status = 409)]
    AlreadyExists,
    #[oai(status = 400)]
    BadRequest(PlainText<String>),
}

#[oai(path = "/users", method = "post")]
async fn create(&self, user: Json<CreateUser>) -> Result<CreateResponse, CreateError> {
    // ...
}
```

### Uniform Response Wrapping

For APIs that always return `{ "code": ..., "msg": ..., "data": ... }` regardless of status:

```rust
#[derive(Object)]
struct ApiResponse<T> {
    code: i32,
    msg: String,
    data: Option<T>,
}

#[derive(ApiResponse)]
#[oai(bad_request_handler = "handle_bad_request")]
enum MyResponse<T: ParseFromJSON + ToJSON + Send + Sync> {
    #[oai(status = 200)]
    Ok(Json<ApiResponse<T>>),
}

fn handle_bad_request<T>(err: Error) -> MyResponse<T> {
    MyResponse::Ok(Json(ApiResponse {
        code: -1,
        msg: err.to_string(),
        data: None,
    }))
}
```

---

## Request Parameters

poem-openapi provides typed extractors for each parameter location:

```rust
use poem_openapi::param::{Path, Query, Header, Cookie};
use poem_openapi::payload::{Json, PlainText};

#[oai(path = "/users/:user_id/posts/:post_id", method = "get")]
async fn get_post(
    &self,
    user_id: Path<i64>,           // Path parameter: /users/42/posts/7
    post_id: Path<i64>,           // Path parameter
    expand: Query<Option<bool>>,  // Query parameter: ?expand=true
    token: Header<String>,        // Header: Authorization: ...
) -> Json<Post> { /* ... */ }
```

- **`Path<T>`** — Extract from URL path (`:name` patterns)
- **`Query<T>`** — Extract from query string (use `Option<T>` for optional)
- **`Header<T>`** — Extract from request headers (use `#[oai(name = "X-Custom")]` to customize)
- **`Cookie<T>`** — Extract from cookies
- **`Json<T>`** — Parse JSON request body (T must implement `Object`)
- **`PlainText<T>`** — Plain text body
- **`Data<&T>`** — Shared application state (poem extractor, works inside OpenAPI handlers)

All parameter types support validators via `#[oai(validator(...))]`.

---

## Shared State

Pass state through poem's `Data` extractor. Add state to the route with `.data()`:

```rust
use poem::{web::Data, EndpointExt};
use tokio::sync::Mutex;
use std::sync::Arc;

struct AppState {
    db: DbPool,
    // ...
}

#[OpenApi]
impl Api {
    #[oai(path = "/items", method = "get")]
    async fn list(&self, state: Data<&AppState>) -> Json<Vec<Item>> {
        // Access state via state.0
        let items = state.0.db.fetch_all().await;
        Json(items)
    }
}

// In main:
let state = Arc::new(AppState { db: pool });
let route = Route::new()
    .nest("/api", api_service)
    .data(state);
```

For state internal to the API struct itself (like an in-memory store), put it directly in the struct:

```rust
#[derive(Default)]
struct Api {
    users: Mutex<Slab<User>>,
}
```

---

## File Uploads

### Define a multipart payload

```rust
use poem_openapi::{Multipart, types::multipart::Upload};

#[derive(Multipart)]
struct UploadPayload {
    description: String,
    file: Upload,            // Single file
    attachments: Vec<Upload>, // Multiple files
}
```

### Handle upload and download

```rust
use poem_openapi::payload::{Attachment, AttachmentType};

#[derive(ApiResponse)]
enum FileResponse {
    #[oai(status = 200)]
    Ok(Attachment<Vec<u8>>),
    #[oai(status = 404)]
    NotFound,
}

#[oai(path = "/files", method = "post")]
async fn upload(&self, upload: UploadPayload) -> Result<Json<u64>> {
    let data = upload.file.into_vec().await?;
    // save data...
    Ok(Json(id))
}

#[oai(path = "/files/:id", method = "get")]
async fn download(&self, id: Path<u64>) -> FileResponse {
    match load_file(id.0) {
        Some((name, data)) => FileResponse::Ok(
            Attachment::new(data)
                .attachment_type(AttachmentType::Attachment)
                .filename(&name)
        ),
        None => FileResponse::NotFound,
    }
}
```

Requires `poem = { features = ["multipart", "tempfile"] }`.

---

## Authentication

Read `references/auth-patterns.md` for full details. Here are the common patterns:

### Basic Auth

```rust
use poem_openapi::{auth::Basic, SecurityScheme};

#[derive(SecurityScheme)]
#[oai(ty = "basic")]
struct MyAuth(Basic);

#[oai(path = "/protected", method = "get")]
async fn protected(&self, auth: MyAuth) -> PlainText<String> {
    // auth.0.username, auth.0.password
    PlainText(format!("hello {}", auth.0.username))
}
```

### API Key with Custom Checker

```rust
#[derive(SecurityScheme)]
#[oai(
    ty = "api_key",
    key_name = "X-API-Key",
    key_in = "header",
    checker = "check_api_key"
)]
struct MyApiKey(User);

async fn check_api_key(req: &Request, key: ApiKey) -> Option<User> {
    // Validate the key and return the user
    validate_token(&key.key).await
}
```

### Optional Auth

```rust
#[derive(SecurityScheme)]
enum OptionalAuth {
    Session(SessionAuth),
    #[oai(fallback)]
    Anonymous,
}
```

---

## Server-Sent Events

```rust
use poem_openapi::payload::EventStream;
use futures_util::stream::BoxStream;

#[derive(Object)]
struct Event {
    value: i32,
}

#[oai(path = "/events", method = "get")]
async fn events(&self) -> EventStream<BoxStream<'static, Event>> {
    EventStream::new(
        async_stream::stream! {
            loop {
                tokio::time::sleep(Duration::from_secs(1)).await;
                yield Event { value: 42 };
            }
        }
        .boxed(),
    )
}
```

---

## Organizing Large APIs

### Tags

Group operations in the Swagger UI:

```rust
use poem_openapi::Tags;

#[derive(Tags)]
enum ApiTags {
    /// User operations
    Users,
    /// Order operations
    Orders,
}

#[oai(path = "/users", method = "get", tag = "ApiTags::Users")]
async fn list_users(&self) -> Json<Vec<User>> { /* ... */ }
```

### Combining Multiple APIs

Split your API into multiple structs, then combine them:

```rust
struct UsersApi;
struct OrdersApi;

#[OpenApi]
impl UsersApi { /* ... */ }

#[OpenApi]
impl OrdersApi { /* ... */ }

// Combine into one service:
let api_service = OpenApiService::new(
    (UsersApi, OrdersApi),
    "My API",
    "1.0"
);
```

### Prefix Paths

```rust
#[OpenApi(prefix_path = "/v1")]
impl ApiV1 {
    #[oai(path = "/users", method = "get")]  // becomes /v1/users
    async fn list(&self) -> Json<Vec<User>> { /* ... */ }
}
```

---

## The Server Layer

### OpenApiService Builder

```rust
let api_service = OpenApiService::new(Api, "My API", "1.0.0")
    .server("http://localhost:3000/api")
    .description("A sample API")
    .contact(ContactObject::new().name("Team").email("team@example.com"))
    .license(LicenseObject::new().name("MIT"));
```

### UI Endpoints

```rust
let ui = api_service.swagger_ui();         // Swagger UI
let docs = api_service.scalar();           // Scalar (modern UI)
let redoc = api_service.redoc();           // Redoc
let rapidoc = api_service.rapidoc();       // RapiDoc

// Serve spec endpoints
let spec_json = api_service.spec_endpoint();      // JSON spec
let spec_yaml = api_service.spec_endpoint_yaml(); // YAML spec
```

### Mounting Routes

```rust
let app = Route::new()
    .nest("/api", api_service)         // API endpoints
    .nest("/docs", ui)                 // Swagger UI
    .at("/openapi.json", spec_json)    // Raw spec
    .with(Cors::new())                 // Middleware
    .data(state);                      // Shared state

Server::new(TcpListener::bind("0.0.0.0:3000"))
    .run(app)
    .await
}
```

---

## Middleware

Poem middleware is applied via `.with()` on routes. Works the same with or without OpenAPI.

### Per-Operation Middleware

Use the `transform` attribute on individual operations:

```rust
fn add_custom_header(ep: impl Endpoint) -> impl Endpoint {
    ep.with(SetHeader::new().appending("X-Custom", "value"))
}

#[oai(path = "/special", method = "get", transform = "add_custom_header")]
async fn special(&self) -> PlainText<&'static str> {
    PlainText("ok")
}
```

### Global Middleware on Route

```rust
use poem::middleware::Cors;

let app = Route::new()
    .nest("/api", api_service)
    .with(Cors::new());  // CORS for all routes
```

### Around Middleware (Request/Response Inspection)

```rust
app.around(|ep, req| async move {
    let start = Instant::now();
    let resp = ep.get_response(req).await;
    println!("{} {} {}ms", resp.status(), req.uri(), start.elapsed().as_millis());
    Ok(resp)
})
```

---

## Error Handling Patterns

### Using poem's `Result`

```rust
use poem::{error::InternalServerError, Result};

#[oai(path = "/users/:id", method = "get")]
async fn get(&self, pool: Data<&DbPool>, id: Path<i64>) -> Result<Json<User>> {
    let user = sqlx::query_as::<_, User>("SELECT * FROM users WHERE id = ?")
        .bind(id.0)
        .fetch_optional(pool.0)
        .await
        .map_err(InternalServerError)?;
    Ok(Json(user))
}
```

### Typed Error Responses

```rust
#[derive(ApiResponse)]
enum GetUserError {
    #[oai(status = 404)]
    NotFound(PlainText<String>),
    #[oai(status = 500)]
    InternalError,
}

#[oai(path = "/users/:id", method = "get")]
async fn get(&self, id: Path<i64>) -> Result<GetUserResponse, GetUserError> {
    find_user(id.0).await
        .map(|u| GetUserResponse::Ok(Json(u)))
        .ok_or_else(|| GetUserError::NotFound(PlainText("not found".into())))
}
```

---

## Testing

Poem provides a `TestClient` for integration tests:

```rust
use poem::test::TestClient;

#[tokio::test]
async fn test_hello() {
    let app = Route::new().nest("/api", api_service);
    let cli = TestClient::new(app);

    let resp = cli.get("/api/hello").query("name", &"world").send().await;
    resp.assert_status_is_ok();
    let body = resp.json().await;
    assert_eq!(body, "hello, world!");
}
```

---

## Cargo.toml Feature Checklist

Enable only what you need:

**poem-openapi features:**
- `swagger-ui` — Swagger UI documentation
- `scalar` — Scalar documentation UI
- `redoc` — Redoc documentation UI
- `rapidoc` — RapiDoc documentation UI
- `email` — `Email` type with built-in validation
- `hostname` — `Hostname` type
- `chrono` — `DateTime`, `Date`, `Duration` types
- `uuid` — `Uuid` type
- `url` — `Url` type
- `sonic-rs` — Faster JSON with sonic-rs instead of serde_json

**poem features:**
- `server` — TCP server (enabled by default)
- `multipart` — Multipart form data
- `tempfile` — Temp file support for uploads
- `compression` — Response compression (gzip, brotli, zstd)
- `cookie` — Cookie support
- `session` — Session management
- `redis-session` — Redis-backed sessions
- `rustls` / `native-tls` — TLS support
- `static-files` — Static file serving
- `websocket` — WebSocket support
- `csrf` — CSRF protection
- `opentelemetry` — OpenTelemetry tracing
- `acme` — Automatic HTTPS via Let's Encrypt
- `i18n` — Internationalization

---

## Common Patterns to Remember

1. **Handlers take `&self`** — the first parameter is always `&self` (your API struct), followed by extractors.
2. **Extractors are positional** — poem-openapi figures out where each parameter goes based on its type (`Path`, `Query`, `Json`, `Data`, etc.), not on position.
3. **Doc comments become OpenAPI descriptions** — `/// Create a new user` on a handler becomes the operation description in the spec.
4. **`Option<T>` means optional** — for query params, fields in request objects, and anywhere else you want optional values.
5. **Use `Result<SuccessResponse, ErrorResponse>`** for endpoints that can fail in typed ways, or just `poem::Result<T>` when you don't need typed error responses.
6. **`Data<&T>` accesses shared state** — the state is set on the Route via `.data()` and available in any handler.
7. **`OpenApiService::new(...)` accepts tuples** — `(Api1, Api2, Api3)` combines multiple API structs.
8. **The `.0` pattern** — `Path<i64>` wraps an `i64`, access it as `path_param.0`. Same for `Query`, `Json`, etc.
