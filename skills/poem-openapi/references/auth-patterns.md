# Authentication Patterns

## Table of Contents
- [Basic Authentication](#basic-authentication)
- [API Key (Header)](#api-key-header)
- [API Key with Custom Checker](#api-key-with-custom-checker)
- [Bearer Token](#bearer-token)
- [OAuth2 Authorization Code](#oauth2-authorization-code)
- [Multiple Authentication Methods](#multiple-authentication-methods)
- [Optional Authentication](#optional-authentication)
- [Scopes](#oauth-scopes)

---

## Basic Authentication

The simplest pattern. The `Basic` type gives you `username` and `password` as strings.

```rust
use poem_openapi::{auth::Basic, payload::PlainText, SecurityScheme, OpenApi};
use poem::{http::StatusCode, Error, Result};

#[derive(SecurityScheme)]
#[oai(ty = "basic")]
struct MyBasicAuth(Basic);

#[OpenApi]
impl Api {
    #[oai(path = "/protected", method = "get")]
    async fn protected(&self, auth: MyBasicAuth) -> Result<PlainText<String>> {
        if auth.0.username != "admin" || auth.0.password != "secret" {
            return Err(Error::from_status(StatusCode::UNAUTHORIZED));
        }
        Ok(PlainText(format!("hello, {}", auth.0.username)))
    }
}
```

The `Basic` struct has fields:
- `username: String`
- `password: String`

---

## API Key (Header)

Accepts an API key from a header, query parameter, or cookie.

```rust
use poem_openapi::{auth::ApiKey, payload::PlainText, SecurityScheme, OpenApi};

#[derive(SecurityScheme)]
#[oai(ty = "api_key", key_name = "X-API-Key", key_in = "header")]
struct MyApiKeyAuth(ApiKey);

#[OpenApi]
impl Api {
    #[oai(path = "/data", method = "get")]
    async fn get_data(&self, auth: MyApiKeyAuth) -> PlainText<String> {
        // auth.0.key contains the raw key value
        PlainText(format!("key: {}", auth.0.key))
    }
}
```

`key_in` options: `"header"`, `"query"`, `"cookie"`.

The `ApiKey` struct has:
- `key: String`

---

## API Key with Custom Checker

Validate the key and convert it to a user/session type. The checker function receives a `&Request` (for accessing shared state) and the extracted `ApiKey`.

```rust
use poem::{Request, web::Data, EndpointExt};
use poem_openapi::{auth::ApiKey, payload::{Json, PlainText}, Object, SecurityScheme, OpenApi};

#[derive(Clone)]
struct User {
    username: String,
    role: String,
}

#[derive(SecurityScheme)]
#[oai(
    ty = "api_key",
    key_name = "X-API-Key",
    key_in = "header",
    checker = "check_api_key"
)]
struct MyApiKeyAuth(User);

async fn check_api_key(req: &Request, api_key: ApiKey) -> Option<User> {
    // Access shared state from the request
    let db = req.data::<DbPool>()?;
    validate_key(&api_key.key, db).await
}

#[OpenApi]
impl Api {
    #[oai(path = "/me", method = "get")]
    async fn me(&self, auth: MyApiKeyAuth) -> Json<User> {
        Json(auth.0)
    }
}
```

The checker must return `Option<T>` (None = auth failed) or `poem::Result<T>` (Err = custom error response).

---

## Bearer Token

For JWT or other bearer token schemes:

```rust
use poem_openapi::{auth::Bearer, SecurityScheme};

#[derive(SecurityScheme)]
#[oai(ty = "bearer", bearer_format = "JWT")]
struct MyBearerAuth(Bearer);
```

The `Bearer` struct has:
- `token: String`

---

## OAuth2 Authorization Code

Full OAuth2 with authorization code flow:

```rust
use poem_openapi::{auth::Bearer, payload::PlainText, OAuthScopes, SecurityScheme, OpenApi};

#[derive(OAuthScopes)]
enum GithubScopes {
    #[oai(rename = "public_repo")]
    PublicRepo,
    #[oai(rename = "user")]
    User,
}

#[derive(SecurityScheme)]
#[oai(
    ty = "oauth2",
    flows(authorization_code(
        authorization_url = "https://github.com/login/oauth/authorize",
        token_url = "https://github.com/login/oauth/access_token",
        scopes = "GithubScopes"
    ))
)]
struct GithubAuth(Bearer);

#[OpenApi]
impl Api {
    #[oai(path = "/repos", method = "get")]
    async fn repos(
        &self,
        #[oai(scope = "GithubScopes::PublicRepo")] auth: GithubAuth,
    ) -> PlainText<String> {
        // auth.0.token contains the access token
        PlainText(format!("token: {}", auth.0.token))
    }
}
```

---

## Multiple Authentication Methods

Use an enum to accept different auth methods. All must pass for the request to succeed.

```rust
use poem_openapi::{
    auth::{ApiKey, Basic},
    payload::PlainText,
    SecurityScheme, OpenApi,
};

#[derive(SecurityScheme)]
#[oai(ty = "basic")]
struct BasicAuth(Basic);

#[derive(SecurityScheme)]
#[oai(ty = "api_key", key_name = "X-API-Key", key_in = "header")]
struct ApiKeyAuth(ApiKey);

// Both must be provided
#[derive(SecurityScheme)]
enum MultiAuth {
    Basic(BasicAuth),
    ApiKey(ApiKeyAuth),
}
```

---

## Optional Authentication

For endpoints that work for both authenticated and anonymous users. Use `#[oai(fallback)]` on a variant:

```rust
use poem::{Request, Result, Route};
use poem_openapi::{auth::ApiKey, payload::PlainText, SecurityScheme, OpenApi};

#[derive(Clone)]
struct User {
    username: String,
}

#[derive(SecurityScheme)]
#[oai(
    ty = "api_key",
    key_name = "session",
    key_in = "cookie",
    checker = "session_checker"
)]
struct SessionAuth(User);

async fn session_checker(_req: &Request, api_key: ApiKey) -> Option<User> {
    match api_key.key.as_str() {
        "valid-token" => Some(User { username: "alice".into() }),
        _ => None,
    }
}

#[derive(SecurityScheme)]
enum OptionalAuth {
    Session(SessionAuth),
    #[oai(fallback)]
    Anonymous,
}

#[OpenApi]
impl Api {
    /// Returns personalized greeting if logged in, generic greeting otherwise
    #[oai(path = "/hello", method = "get")]
    async fn hello(&self, auth: OptionalAuth) -> PlainText<String> {
        match auth {
            OptionalAuth::Session(auth) => PlainText(format!("hello, {}", auth.0.username)),
            OptionalAuth::Anonymous => PlainText("hello, anonymous".into()),
        }
    }

    /// Returns 401 if not authenticated
    #[oai(path = "/me", method = "get")]
    async fn me(&self, auth: SessionAuth) -> PlainText<String> {
        PlainText(auth.0.username)
    }
}
```

The difference between optional and required auth:
- **Required** (single struct like `SessionAuth`) — returns 401 if validation fails
- **Optional** (enum with `#[oai(fallback)]`) — falls through to the fallback variant instead of 401

---

## OAuth Scopes

Define available scopes for OAuth2 schemes:

```rust
use poem_openapi::OAuthScopes;

#[derive(OAuthScopes)]
enum MyScopes {
    #[oai(rename = "read")]
    Read,
    #[oai(rename = "write")]
    Write,
    #[oai(rename = "admin")]
    Admin,
}
```

Use scopes on individual handlers:

```rust
#[oai(path = "/data", method = "get")]
async fn read_data(
    &self,
    #[oai(scope = "MyScopes::Read")] auth: MyOAuth,
) -> Json<Data> { /* ... */ }

#[oai(path = "/data", method = "post")]
async fn write_data(
    &self,
    #[oai(scope = "MyScopes::Write")] auth: MyOAuth,
) -> Json<Data> { /* ... */ }
```
