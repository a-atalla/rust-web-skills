# Derive Macro Reference

Complete attribute tables for all poem-openapi derive macros.

## Table of Contents
- [OpenApi](#openapi)
- [Object](#object)
- [Enum](#enum)
- [Union](#union)
- [ApiResponse](#apiresponse)
- [ApiRequest](#apirequest)
- [Multipart](#multipart)
- [SecurityScheme](#securityscheme)
- [NewType](#newtype)
- [Tags](#tags)
- [Webhook](#webhook)
- [Validators](#validators)

---

## OpenApi

Applied to an `impl` block. Each method becomes an API operation.

### Macro-level attributes (`#[OpenApi(...)]`)

| Attribute | Description | Type | Optional |
|---|---|---|---|
| `prefix_path` | Prefix for all operation paths. May contain shared path parameters. | string | Y |
| `tag` | Default tag for all operations. Must be a variant of a `Tags` enum. | Tags | Y |
| `response_header` | Extra response header on all operations. | ExtraHeader | Y |
| `request_header` | Extra request header on all operations. | ExtraHeader | Y |
| `ignore_case` | Ignore case when matching parameter names. | bool | Y |

### Operation attributes (`#[oai(...)]`)

| Attribute | Description | Type | Optional |
|---|---|---|---|
| `path` | URI path with optional path parameters (`:name`) | string | N |
| `method` | HTTP method: get, post, put, delete, head, options, connect, patch, trace | string | N |
| `tag` | Tag for this operation | Tags | Y |
| `deprecated` | Mark operation as deprecated | bool | Y |
| `operation_id` | Unique operation identifier | string | Y |
| `transform` | Function to transform the endpoint (for middleware) | string | Y |
| `response_header` | Extra response header | ExtraHeader | Y |
| `request_header` | Extra request header | ExtraHeader | Y |
| `hidden` | Hide operation from docs | bool | Y |
| `ignore_case` | Ignore case when matching parameter names | bool | Y |
| `code_samples` | Code samples for the operation | object | Y |
| `actual_type` | Specifies the actual response type | string | Y |
| `external_docs` | External documentation URL | string | Y |

### Operation argument attributes

Applied to individual parameters with `#[oai(...)]`:

| Attribute | Description | Type | Optional |
|---|---|---|---|
| `name` | Override parameter name | string | Y |
| `ignore_case` | Ignore case when matching | bool | Y |
| `deprecated` | Mark parameter as deprecated | bool | Y |
| `default` | Default value | bool or string | Y |
| `explode` | Explode arrays/objects into separate params | bool (default: true) | Y |
| `validator.*` | See Validators section below | varies | Y |

### Example

```rust
#[derive(Tags)]
enum MyTags { V1, V2 }

struct Api;

#[OpenApi(prefix_path = "/v1/:customer_id", tag = "MyTags::V1")]
impl Api {
    #[oai(path = "/hello", method = "get")]
    async fn hello(&self, customer_id: Path<String>) -> PlainText<String> {
        PlainText(format!("Hello {}!", customer_id.0))
    }
}
```

---

## Object

Defines a JSON schema for request/response bodies.

### Macro attributes

| Attribute | Description | Type | Optional |
|---|---|---|---|
| `rename` | Rename the object | string | Y |
| `rename_all` | Rename fields: lowercase, UPPERCASE, PascalCase, camelCase, snake_case, SCREAMING_SNAKE_CASE, kebab-case, SCREAMING-KEBAB-CASE | string | Y |
| `default` | Default value | bool or string | Y |
| `deprecated` | Schema deprecated | bool | Y |
| `read_only_all` | Set all fields readOnly | bool | Y |
| `write_only_all` | Set all fields writeOnly | bool | Y |
| `deny_unknown_fields` | Error on unknown fields during parsing | bool | Y |
| `example` | Type implements `Example` trait | bool | Y |
| `external_docs` | External documentation URL | string | Y |
| `remote` | Derive for a remote type | string | Y |
| `skip_serializing_if_is_none` | Skip serializing None fields | bool | Y |
| `skip_serializing_if_is_empty` | Skip serializing empty fields | bool | Y |

### Field attributes

| Attribute | Description | Type | Optional |
|---|---|---|---|
| `skip` | Skip this field | bool | Y |
| `rename` | Rename the field | string | Y |
| `default` | Default value | bool or string | Y |
| `read_only` | Field is readOnly (server-set) | bool | Y |
| `write_only` | Field is writeOnly (client-set only) | bool | Y |
| `deprecated` | Mark as deprecated | bool | Y |
| `flatten` | Flatten into parent (like serde) | bool | Y |
| `skip_serializing_if_is_none` | Skip if None | bool | Y |
| `skip_serializing_if_is_empty` | Skip if empty | bool | Y |
| `skip_serializing_if` | Custom skip function | string | Y |
| `validator.*` | See Validators | varies | Y |

---

## Enum

Defines an OpenAPI enum type.

### Macro attributes

| Attribute | Description | Type | Optional |
|---|---|---|---|
| `rename` | Rename the enum | string | Y |
| `rename_all` | Rename variants | string | Y |
| `deprecated` | Schema deprecated | bool | Y |
| `external_docs` | External documentation | string | Y |
| `remote` | Derive for a remote enum | string | Y |

### Item attributes

| Attribute | Description | Type | Optional |
|---|---|---|---|
| `rename` | Rename the variant | string | Y |

---

## Union

Defines an OpenAPI `oneOf` type.

### Macro attributes

| Attribute | Description | Type | Optional |
|---|---|---|---|
| `discriminator_name` | Property name for discriminator | string | Y |
| `externally_tagged` | Use externally tagged format | bool | Y |
| `one_of` | Validate against exactly one subschema | bool | Y |
| `external_docs` | External documentation | string | Y |
| `rename_all` | Rename mapping names | string | Y |

### Item attributes

| Attribute | Description | Type | Optional |
|---|---|---|---|
| `mapping` | Rename the payload value (default: object name) | string | Y |

---

## ApiResponse

Defines a response with multiple possible status codes.

### Macro attributes

| Attribute | Description | Type | Optional |
|---|---|---|---|
| `bad_request_handler` | Custom handler to convert errors to this response type | string | Y |
| `header` | Extra header | ExtraHeader | Y |
| `display` | Use Display trait for error messages | bool | Y |

### Item attributes

| Attribute | Description | Type | Optional |
|---|---|---|---|
| `status` | HTTP status code | u16 | Y |
| `status_range` | Range of status codes (e.g., "2XX") | string | Y |
| `content_type` | Override content type | string | Y |
| `actual_type` | Actual response type | string | Y |
| `header` | Extra header | ExtraHeader | Y |

### ExtraHeader parameters

| Attribute | Description | Type | Optional |
|---|---|---|---|
| `name` | Header name | string | N |
| `ty` | Header type | string | N |
| `description` | Description | string | Y |
| `deprecated` | Deprecated | bool | Y |

### Response header in variant

```rust
#[derive(ApiResponse)]
enum MyResponse {
    #[oai(status = 200)]
    Ok(Json<User>, #[oai(header = "X-Total")] i64),
}
```

### bad_request_handler example

```rust
fn bad_request_handler(err: Error) -> MyResponse {
    MyResponse::BadRequest(PlainText(err.to_string()))
}

#[derive(ApiResponse)]
#[oai(bad_request_handler = "bad_request_handler")]
enum MyResponse {
    #[oai(status = 200)]
    Ok(Json<User>),
    #[oai(status = 400)]
    BadRequest(PlainText<String>),
}
```

---

## ApiRequest

Defines a request that can accept multiple content types.

### Item attributes

| Attribute | Description | Type | Optional |
|---|---|---|---|
| `content_type` | Override content type | string | Y |

```rust
#[derive(ApiRequest)]
enum CreatePetRequest {
    CreateByJSON(Json<Pet>),
    CreateByPlainText(PlainText<String>),
}
```

---

## Multipart

Defines a multipart form payload.

### Macro attributes

| Attribute | Description | Type | Optional |
|---|---|---|---|
| `rename_all` | Rename fields | string | Y |
| `deny_unknown_fields` | Error on unknown fields | bool | Y |

### Field attributes

Same as Object field attributes (skip, rename, default, validators).

```rust
#[derive(Multipart)]
struct UploadImages {
    name: String,
    files: Vec<Upload>,
}
```

---

## SecurityScheme

Defines authentication for the API.

### Macro attributes

| Attribute | Description | Type | Optional |
|---|---|---|---|
| `rename` | Rename the scheme | string | Y |
| `ty` | Type: api_key, basic, bearer, oauth2, openid_connect | string | N |
| `key_in` | Location (api_key): query, header, cookie | string | Y |
| `key_name` | Param name (api_key) | string | Y |
| `bearer_format` | Token format hint (bearer) | string | Y |
| `flows` | OAuth2 flow config | OAuthFlows | Y |
| `openid_connect_url` | OpenID Connect URL | string | Y |
| `checker` | Validation function returning `Option<T>` or `Result<T>` | string | Y |

### Multiple auth methods (enum)

```rust
#[derive(SecurityScheme)]
enum MyAuth {
    Basic(BasicAuth),
    ApiKey(ApiKeyAuth),
}
```

### Optional auth (enum with fallback)

```rust
#[derive(SecurityScheme)]
enum OptionalAuth {
    Session(SessionAuth),
    #[oai(fallback)]
    Anonymous,
}
```

### OAuth2 flows

```rust
#[derive(SecurityScheme)]
#[oai(
    ty = "oauth2",
    flows(authorization_code(
        authorization_url = "https://provider.com/authorize",
        token_url = "https://provider.com/token",
        scopes = "MyScopes"
    ))
)]
struct OAuth2Auth(Bearer);
```

---

## NewType

Wraps a primitive type.

### Macro attributes

| Attribute | Description | Type | Optional |
|---|---|---|---|
| `from_json` | Implement ParseFromJSON (default: true) | bool | Y |
| `from_parameter` | Implement ParseFromParameter (default: true) | bool | Y |
| `from_multipart` | Implement ParseFromMultipartField (default: true) | bool | Y |
| `to_json` | Implement ToJSON (default: true) | bool | Y |
| `to_header` | Implement ToHeader (default: true) | bool | Y |
| `external_docs` | External documentation | string | Y |
| `example` | Type implements Example trait | bool | Y |
| `rename` | Rename the type | string | Y |

---

## Tags

Groups operations in the API documentation.

### Macro attributes

| Attribute | Description | Type | Optional |
|---|---|---|---|
| `rename_all` | Rename variants | string | Y |

### Item attributes

| Attribute | Description | Type | Optional |
|---|---|---|---|
| `rename` | Rename tag | string | Y |
| `external_docs` | External documentation | string | Y |

```rust
#[derive(Tags)]
enum ApiTags {
    /// User operations
    Users,
    /// Order operations
    Orders,
}
```

---

## Webhook

Defines OpenAPI webhooks.

### Macro attributes

| Attribute | Description | Type | Optional |
|---|---|---|---|
| `tag` | Tag for all operations | string | Y |

### Operation attributes

| Attribute | Description | Type | Optional |
|---|---|---|---|
| `name` | Webhook key name | bool | Y |
| `method` | HTTP method | string | N |
| `deprecated` | Operation deprecated | bool | Y |
| `tag` | Operation tag | Tags | Y |
| `operation_id` | Unique identifier | string | Y |

---

## Validators

Available on Object fields, operation arguments, and Multipart fields via `#[oai(validator(...))]`:

| Validator | Description | Type |
|---|---|---|
| `multiple_of` | Must be divisible by this value (positive) | number |
| `maximum` | Upper bound: `{ value: N, exclusive: bool }` | object |
| `minimum` | Lower bound: `{ value: N, exclusive: bool }` | object |
| `max_length` | Max string length | usize |
| `min_length` | Min string length | usize |
| `pattern` | Regex pattern (ECMA 262) | string |
| `max_items` | Max array size | usize |
| `min_items` | Min array size | usize |
| `unique_items` | All array elements unique | bool |
| `max_properties` | Max number of object properties | usize |
| `min_properties` | Min number of object properties | usize |
| `email` | Validate as email | (flag) |
| `url` | Validate as URL | (flag) |

Example:
```rust
#[derive(Object)]
struct User {
    #[oai(validator(max_length = 64, min_length = 1))]
    name: String,
    #[oai(validator(maximum = { value = 150, exclusive = false }))]
    age: i32,
}
```
