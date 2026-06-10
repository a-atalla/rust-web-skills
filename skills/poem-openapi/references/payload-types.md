# Payload and Parameter Types Reference

## Table of Contents
- [Parameter Extractors](#parameter-extractors)
- [Request Payload Types](#request-payload-types)
- [Response Payload Types](#response-payload-types)
- [Special Types](#special-types)
- [Multipart Types](#multipart-types)
- [Content Negotiation](#content-negotiation)

---

## Parameter Extractors

These extract values from the request URL, headers, query string, or cookies.

### `Path<T>`

Extract from URL path parameters. Use `:name` in the path pattern.

```rust
use poem_openapi::param::Path;

#[oai(path = "/users/:user_id/posts/:post_id", method = "get")]
async fn get_post(&self, user_id: Path<i64>, post_id: Path<i64>) -> Json<Post> {
    // user_id.0 is i64, post_id.0 is i64
}
```

Supports: all primitives, String, and types implementing `ParseFromParameter`.

### `Query<T>`

Extract from query string. Use `Option<T>` for optional params.

```rust
use poem_openapi::param::Query;

#[oai(path = "/items", method = "get")]
async fn list(
    &self,
    page: Query<Option<u32>>,        // ?page=2
    limit: Query<Option<u32>>,       // ?limit=10
    sort: Query<Option<String>>,     // ?sort=created_at
) -> Json<Vec<Item>> { /* ... */ }
```

### `Header<T>`

Extract from request headers.

```rust
use poem_openapi::param::Header;

#[oai(path = "/data", method = "get")]
async fn get(
    &self,
    #[oai(name = "X-Request-ID")] request_id: Header<String>,
    auth: Header<Option<String>>,
) -> Json<Data> { /* ... */ }
```

### `Cookie<T>`

Extract from cookies.

```rust
use poem_openapi::param::Cookie;

#[oai(path = "/me", method = "get")]
async fn me(&self, session: Cookie<String>) -> Json<User> { /* ... */ }
```

---

## Request Payload Types

### `Json<T>`

Parse a JSON request body. T must implement `Object`.

```rust
use poem_openapi::payload::Json;

#[oai(path = "/users", method = "post")]
async fn create(&self, user: Json<CreateUser>) -> Json<User> {
    // user.0 is the CreateUser struct
}
```

### `PlainText<T>`

Plain text request body.

```rust
use poem_openapi::payload::PlainText;

#[oai(path = "/message", method = "post")]
async fn send(&self, msg: PlainText<String>) -> PlainText<String> {
    // msg.0 is the String
    PlainText(format!("Received: {}", msg.0))
}
```

### `Form<T>`

URL-encoded form data.

```rust
use poem_openapi::payload::Form;

#[oai(path = "/login", method = "post")]
async fn login(&self, creds: Form<LoginRequest>) -> Json<Token> { /* ... */ }
```

---

## Response Payload Types

### `Json<T>`

JSON response. T must implement `Object`.

```rust
Json(vec![user1, user2])  // Vec<User>
Json(user)                 // User
Json(42i64)                // i64
```

### `PlainText<T>`

Plain text response with `text/plain` content type.

```rust
PlainText("hello!".to_string())
PlainText(format!("Hello, {}!", name))
```

### `Html<T>`

HTML response with `text/html` content type.

```rust
use poem_openapi::payload::Html;

Html("<h1>Welcome</h1>".to_string())
```

### `Binary<T>`

Binary response with `application/octet-stream`.

```rust
use poem_openapi::payload::Binary;

Binary(bytes)  // Bytes or Vec<u8>
```

### `Attachment<T>`

File download response with `Content-Disposition` header.

```rust
use poem_openapi::payload::{Attachment, AttachmentType};

Attachment::new(file_bytes)
    .attachment_type(AttachmentType::Attachment)
    .filename("report.pdf")
```

`AttachmentType` values:
- `AttachmentType::Attachment` — prompts download
- `AttachmentType::Inline` — displays in browser

### `EventStream<T>`

Server-Sent Events. T must implement `Object`.

```rust
use poem_openapi::payload::EventStream;
use futures_util::stream::BoxStream;

#[oai(path = "/events", method = "get")]
async fn events(&self) -> EventStream<BoxStream<'static, MyEvent>> {
    EventStream::new(async_stream::stream! {
        loop {
            tokio::time::sleep(Duration::from_secs(1)).await;
            yield MyEvent { value: 42 };
        }
    }.boxed())
}
```

### `Xml<T>`

XML response. Requires `poem` `xml` feature.

```rust
use poem_openapi::payload::Xml;

Xml(data)  // T must implement Serialize
```

### `Yaml<T>`

YAML response. Requires `poem` `yaml` feature.

```rust
use poem_openapi::payload::Yaml;

Yaml(data)
```

---

## Special Types

### `Password`

A string type that renders as a password input in Swagger UI.

```rust
use poem_openapi::types::Password;

#[derive(Object)]
struct User {
    password: Password,
}
```

### `Email`

Email type with built-in validation. Requires `poem-openapi` `email` feature.

```rust
use poem_openapi::types::Email;

#[derive(Object)]
struct User {
    email: Email,
}
```

### `Hostname`

Hostname type with validation. Requires `poem-openapi` `hostname` feature.

### `MaybeUndefined<T>`

Like `Option<T>` but distinguishes between "not provided" and "null".

```rust
use poem_openapi::types::MaybeUndefined;

#[derive(Object)]
struct Update {
    name: MaybeUndefined<String>,  // Can be: undefined, null, or a value
}
```

### `Base64<T>`

Base64-encoded binary data.

```rust
use poem_openapi::types::Base64;

#[derive(Object)]
struct Upload {
    data: Base64<Vec<u8>>,
}
```

---

## Multipart Types

### `Upload`

A file upload in a multipart request.

```rust
use poem_openapi::types::multipart::Upload;

#[derive(Multipart)]
struct UploadPayload {
    name: String,
    description: Option<String>,
    file: Upload,            // Single file
    attachments: Vec<Upload>, // Multiple files
}
```

`Upload` methods:
- `file_name()` → `Option<&str>`
- `content_type()` → `Option<&Mime>`
- `into_vec().await` → `Result<Vec<u8>>`
- `into_temporary_file().await` → `Result<TempFile>`

---

## Content Negotiation

### Using `Accept` header

```rust
use poem::web::Accept;
use poem_openapi::{ApiRequest, ResponseContent, ApiResponse};

#[derive(ApiRequest)]
enum MyRequest {
    Json(Json<Data>),
    Xml(Xml<Data>),
}

#[derive(ResponseContent)]
enum MyContent<T> {
    Json(Json<T>),
    Xml(Xml<T>),
}

#[derive(ApiResponse)]
enum MyResponse<T> {
    #[oai(status = 200)]
    Ok(MyContent<T>),
}

#[oai(path = "/data", method = "get")]
async fn get(&self, accept: Accept) -> MyResponse<Data> {
    let data = Data { /* ... */ };
    // Choose response format based on Accept header
    for mime in &accept.0 {
        match mime.as_ref() {
            "application/json" => return MyResponse::Ok(MyContent::Json(Json(data))),
            "application/xml" => return MyResponse::Ok(MyContent::Xml(Xml(data))),
            _ => {}
        }
    }
    MyResponse::Ok(MyContent::Json(Json(data))) // default
}
```
