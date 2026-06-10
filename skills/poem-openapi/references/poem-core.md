# Poem Core Framework Reference

Poem is the underlying web framework that poem-openapi builds on. This reference covers the core concepts you'll need when building APIs.

## Table of Contents
- [Routing](#routing)
- [Endpoint Combinators](#endpoint-combinators)
- [Middleware](#middleware)
- [Extractors](#extractors)
- [Error Handling](#error-handling)
- [Listeners](#listeners)
- [Testing](#testing)
- [Static Files](#static-files)
- [Sessions](#sessions)
- [TLS/HTTPS](#tlshttps)

---

## Routing

### Basic Routing

```rust
use poem::{Route, get, post, put, delete};

let app = Route::new()
    .at("/users", get(list_users).post(create_user))
    .at("/users/:id", get(get_user).put(update_user).delete(delete_user))
    .at("/files/*path", get(serve_file)); // wildcard
```

### Nested Routing

```rust
let app = Route::new()
    .nest("/api/v1", api_v1_routes)
    .nest("/api/v2", api_v2_routes);
```

### Domain-Based Routing

```rust
use poem::RouteDomain;

let app = RouteDomain::new()
    .domain("api.example.com", api_routes)
    .domain("www.example.com", web_routes);
```

### Method-Based Routing

```rust
use poem::RouteMethod;

let app = RouteMethod::new()
    .get(get_handler)
    .post(post_handler);
```

---

## Endpoint Combinators

All endpoints implement the `Endpoint` trait and can be composed with `EndpointExt`:

```rust
use poem::EndpointExt;

// Add middleware
endpoint.with(middleware)

// Transform the response
endpoint.map(|resp| resp)

// Inspect before passing to handler
endpoint.before(|req| { /* modify req */ Ok(req) })

// Inspect after handler
endpoint.after(|resp| { /* modify resp */ Ok(resp) })

// Wrap handler entirely
endpoint.around(|ep, req| async move {
    let resp = ep.get_response(req).await;
    Ok(resp)
})

// Catch errors
endpoint.catch_error(|err: MyError| { /* return response */ })
endpoint.catch_all_error(|err| { /* handle any error */ })

// Add data to request state
endpoint.data(my_state)

// Filter requests
endpoint.and_then(|req| async move {
    if valid(&req) { Ok(req) } else { Err(rejection) }
})
```

---

## Middleware

### CORS

```rust
use poem::middleware::Cors;

let cors = Cors::new()
    .allow_origin("https://example.com")
    .allow_methods("GET, POST")
    .allow_headers("Content-Type, Authorization");

let app = Route::new().with(cors);
```

### Compression

```rust
use poem::middleware::Compression;

// Requires "compression" feature
let app = Route::new().with(Compression::new());
// Supports: gzip, brotli, deflate, zstd
```

### Request ID

```rust
use poem::middleware::RequestId;

let app = Route::new().with(RequestId::new());
// Access in handler via Data<&RequestId>
```

### Set Headers

```rust
use poem::middleware::SetHeader;

let app = Route::new().with(
    SetHeader::new()
        .appending("X-Frame-Options", "DENY")
        .appending("X-Content-Type-Options", "nosniff")
);
```

### Force HTTPS

```rust
use poem::middleware::ForceHttps;

let app = Route::new().with(ForceHttps::new());
```

### CSRF Protection

```rust
use poem::middleware::Csrf;
// Requires "csrf" feature
```

### Tracing

```rust
use poem::middleware::Tracing;

let app = Route::new().with(Tracing::new());
```

### OpenTelemetry

```rust
// Requires "opentelemetry" feature
use poem::middleware::OpenTelemetryTracing;
use poem::middleware::OpenTelemetryMetrics;
```

### Panic Catching

```rust
use poem::middleware::CatchPanic;

let app = Route::new().with(CatchPanic::new());
```

### Body Size Limit

```rust
use poem::middleware::SizeLimit;

let app = Route::new().with(SizeLimit::new(10 * 1024 * 1024)); // 10MB
```

### Normalize Path

```rust
use poem::middleware::NormalizePath;

// Adds/removes trailing slashes
let app = Route::new().with(NormalizePath::new());
```

---

## Extractors

These are poem's built-in extractors (used with `#[handler]`, not `#[OpenApi]` — though `Data<&T>` works in both).

### Common Extractors

| Extractor | Source | Feature |
|---|---|---|
| `Path<T>` | URL path params | default |
| `Query<T>` | Query string | default |
| `Form<T>` | URL-encoded body | default |
| `Json<T>` | JSON body | default |
| `Xml<T>` | XML body | `xml` |
| `Yaml<T>` | YAML body | `yaml` |
| `Data<&T>` | Shared state | default |
| `Multipart` | Multipart body | `multipart` |
| `TempFile` | Temp file upload | `tempfile` |
| `CookieJar` | Cookies | `cookie` |
| `Session` | Session data | `session` |
| `WebSocket` | WebSocket upgrade | `websocket` |
| `RemoteAddr` | Client address | default |
| `RealIp` | Real IP (from headers) | default |
| `TypedHeader<T>` | Typed header | default |
| `Accept` | Content negotiation | default |
| `Locale` | i18n locale | `i18n` |
| `String` / `Bytes` / `Vec<u8>` | Raw body | default |
| `Method` / `Uri` / `Version` | HTTP primitives | default |
| `&HeaderMap` | All headers | default |
| `&Request` | Full request access | default |
| `Option<T>` | Optional extraction | default |
| `Result<T>` | Fallible extraction | default |

---

## Error Handling

### Built-in Error Constructors

```rust
use poem::error::{
    BadRequest, NotFound, Forbidden, Unauthorized,
    InternalServerError, NotAcceptable, RequestTimeout,
    Conflict, Gone, PayloadTooLarge, UnsupportedMediaType,
};
```

### Custom Error Types

```rust
use poem::{error::ResponseError, http::StatusCode, Error, Response};

#[derive(Debug, thiserror::Error)]
enum ApiError {
    #[error("User not found: {0}")]
    NotFound(i64),
    #[error("Unauthorized")]
    Unauthorized,
}

impl ResponseError for ApiError {
    fn status(&self) -> StatusCode {
        match self {
            ApiError::NotFound(_) => StatusCode::NOT_FOUND,
            ApiError::Unauthorized => StatusCode::UNAUTHORIZED,
        }
    }
}

// Use with poem::Result
fn handler() -> poem::Result<Json<User>> {
    Err(Error::from(ApiError::NotFound(42)))
}
```

---

## Listeners

### TCP

```rust
use poem::listener::TcpListener;

Server::new(TcpListener::bind("0.0.0.0:3000"))
    .run(app)
    .await
```

### Unix Socket

```rust
use poem::listener::UnixListener;

Server::new(UnixListener::bind("/tmp/app.sock"))
    .run(app)
    .await
```

### TLS (Rustls)

```rust
use poem::listener::{TcpListener, RustlsConfig};

let config = RustlsConfig::new()
    .cert(cert_bytes)
    .key(key_bytes);

Server::new(TcpListener::bind("0.0.0.0:443").rustls(config))
    .run(app)
    .await
```

### Combined Listeners

```rust
use poem::listener::CombinedListener;

// Listen on both IPv4 and IPv6
let listener = CombinedListener::new(
    TcpListener::bind("0.0.0.0:3000"),
    TcpListener::bind("[::]:3000"),
);

Server::new(listener).run(app).await
```

### ACME (Let's Encrypt)

```rust
// Requires "acme" feature
use poem::listener::{TcpListener, AcmeConfig};

let acme = AcmeConfig::new(["example.com"])
    .contact_email("admin@example.com")
    .cache_dir("/tmp/acme_cache");

Server::new(TcpListener::bind("0.0.0.0:443").acme(acme))
    .run(app)
    .await
```

---

## Testing

```rust
use poem::test::TestClient;
use poem::{Route, get, handler};

#[handler]
async fn hello() -> &'static str {
    "hello"
}

#[tokio::test]
async fn test_hello() {
    let app = Route::new().at("/hello", get(hello));
    let cli = TestClient::new(app);

    let resp = cli.get("/hello").send().await;
    resp.assert_status_is_ok();
    resp.assert_text("hello").await;
}
```

TestClient methods:
- `.get(path)`, `.post(path)`, `.put(path)`, `.delete(path)`
- `.query(key, value)` — add query parameter
- `.header(key, value)` — add header
- `.body(body)` — set request body
- `.json(&value)` — set JSON body
- `.form(&value)` — set form body
- `.multipart(multipart)` — set multipart body
- `.send().await` — execute the request

Response assertions:
- `.assert_status_is_ok()`
- `.assert_status(status)`
- `.assert_text(expected).await`
- `.json().await` — parse JSON response

---

## Static Files

```rust
// Requires "static-files" feature
use poem::endpoint::StaticFilesEndpoint;

let files = StaticFilesEndpoint::new("./public")
    .index_file("index.html");

let app = Route::new().nest("/static", files);
```

---

## Sessions

### Cookie Session

```rust
// Requires "cookie" + "session" features
use poem::middleware::CookieJarManager;
use poem::session::{CookieSession, MemoryStorage};

let app = Route::new()
    .with(CookieJarManager::new())
    .with(CookieSession::new(MemoryStorage::new(), "session_key"));
```

### Redis Session

```rust
// Requires "redis-session" feature
use poem::session::RedisStorage;
```

---

## TLS/HTTPS

For production TLS, use rustls:

```toml
[dependencies]
poem = { version = "3", features = ["rustls"] }
```

```rust
use poem::listener::{TcpListener, RustlsConfig};
use std::path::PathBuf;

let config = RustlsConfig::new()
    .cert(PathBuf::from("cert.pem"))
    .key(PathBuf::from("key.pem"));

Server::new(TcpListener::bind("0.0.0.0:443").rustls(config))
    .run(app)
    .await
```
