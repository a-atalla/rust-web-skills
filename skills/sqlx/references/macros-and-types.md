# Macros and Types Reference

## Table of Contents
- [Query Macro Variants](#query-macro-variants)
- [Type Override Syntax](#type-override-syntax)
- [FromRow Derive](#fromrow-derive)
- [Type Derive](#type-derive)
- [Encode/Decode Derives](#encodedecode-derives)
- [Placeholder Syntax by Database](#placeholder-syntax-by-database)

---

## Query Macro Variants

| Macro | Compile-time check | Output type | Source |
|---|---|---|---|
| `query!` | Yes | Anonymous struct | Inline SQL |
| `query_as!` | Yes | Named struct | Inline SQL |
| `query_scalar!` | Yes | Single value | Inline SQL |
| `query_file!` | Yes | Anonymous struct | External .sql file |
| `query_file_as!` | Yes | Named struct | External .sql file |
| `query_file_scalar!` | Yes | Single value | External .sql file |
| `query_unchecked!` | No | Anonymous struct | Inline SQL |
| `query_as_unchecked!` | No | Named struct | Inline SQL |
| `query_scalar_unchecked!` | No | Single value | Inline SQL |
| `query_file_unchecked!` | No | Anonymous struct | External .sql file |
| `query_file_as_unchecked!` | No | Named struct | External .sql file |
| `query_file_scalar_unchecked!` | No | Single value | External .sql file |

### Usage Examples

```rust
// query! — anonymous struct with inferred types
let rec = sqlx::query!(
    "SELECT id, name, email FROM users WHERE id = $1",
    user_id
).fetch_one(pool).await?;
// Access: rec.id, rec.name, rec.email

// query_as! — map to your struct
#[derive(FromRow)]
struct User { id: i64, name: String, email: Option<String> }
let user = sqlx::query_as!(User, "SELECT * FROM users WHERE id = $1", id)
    .fetch_optional(pool).await?;

// query_scalar! — single value
let count = sqlx::query_scalar!("SELECT COUNT(*) FROM users")
    .fetch_one(pool).await?;

// query_file! — SQL from external file
// queries/get_user.sql contains the SQL
let user = sqlx::query_file!("queries/get_user.sql", user_id)
    .fetch_optional(pool).await?;

// query_file_as! — file + typed struct
let user = sqlx::query_file_as!(User, "queries/get_user.sql", user_id)
    .fetch_one(pool).await?;

// Unchecked variants — skip type checking
let rec = sqlx::query_unchecked!("SELECT * FROM complex_view")
    .fetch_all(pool).await?;
```

---

## Type Override Syntax

In column aliases, append modifiers after the column name:

```sql
SELECT
    id,
    name as "name!",              -- force NOT NULL
    email as "email?",             -- force nullable
    data as "data: Json<UserMeta>", -- custom type
    meta as "meta!: Json<Meta>",    -- not-null + custom type
    extra as "extra?: Json<Extra>", -- nullable + custom type
    raw as "raw: _"                 -- infer from struct field
FROM records
```

### Common Overrides

| Override | Meaning | Example |
|---|---|---|
| `col!` | Force NOT NULL | `"count!"` → `i64` (not `Option<i64>`) |
| `col?` | Force nullable | `"email?"` → `Option<String>` |
| `col: Type` | Custom type | `"data: Json<Meta>"` → `Json<Meta>` |
| `col!: Type` | NOT NULL + custom | `"ts!: chrono::NaiveDateTime"` |
| `col?: Type` | Nullable + custom | `"ts?: chrono::NaiveDateTime"` |
| `col: _` | Infer from struct | lets `query_as!` figure it out |

---

## FromRow Derive

Maps row columns to struct fields by name.

```rust
#[derive(sqlx::FromRow)]
struct User {
    id: i64,
    name: String,
    email: Option<String>,
}
```

### Field Attributes

| Attribute | Description |
|---|---|
| `#[sqlx(rename = "col_name")]` | Map from different column name |
| `#[sqlx(rename_all = "camelCase")]` | Rename all fields (on struct) |
| `#[sqlx(default)]` | Use Default on null/error |
| `#[sqlx(flatten)]` | Flatten nested FromRow struct |
| `#[sqlx(skip)]` | Don't map from row |
| `#[sqlx(json)]` | Parse as JSON into type |
| `#[sqlx(json(nullable))]` | Parse as nullable JSON |
| `#[sqlx(try_from = "T")]` | TryFrom conversion |

### Examples

```rust
#[derive(sqlx::FromRow)]
struct User {
    id: i64,
    // Column is "email_address" in DB
    #[sqlx(rename = "email_address")]
    email: String,
    // If null, use Default instead of error
    #[sqlx(default)]
    avatar_url: Option<String>,
    // Parse JSON column into typed struct
    #[sqlx(json)]
    preferences: UserPrefs,
    // Flatten audit fields from same row
    #[sqlx(flatten)]
    audit: AuditInfo,
}
```

---

## Type Derive

For custom enum/struct types that map to database types.

### String Enum

```rust
#[derive(sqlx::Type)]
#[sqlx(type_name = "user_status")]
enum UserStatus {
    Active,
    Inactive,
    Suspended,
}
// Maps to PostgreSQL ENUM or SQLite TEXT
```

### Integer Enum

```rust
#[derive(sqlx::Type)]
#[sqlx(repr = "i16")]
enum Priority {
    Low = 0,
    Medium = 1,
    High = 2,
}
// Maps to SMALLINT
```

### Composite Type (PostgreSQL)

```rust
#[derive(sqlx::Type)]
#[sqlx(type_name = "address")]
struct Address {
    street: String,
    city: String,
    zip: String,
}
```

---

## Encode/Decode Derives

For custom types that need database encoding/decoding:

```rust
#[derive(sqlx::Encode, sqlx::Decode, sqlx::Type)]
#[sqlx(type_name = "varchar")]
struct NonEmptyString(String);
```

Usually `Type` derive is sufficient — it generates `Encode` and `Decode` automatically.

---

## Placeholder Syntax by Database

| Database | Positional | Named |
|---|---|---|
| PostgreSQL | `$1`, `$2`, `$3` | — |
| SQLite | `?1`, `?2`, `?3` or `?` | — |
| MySQL | `?` | — |

In `query!` macros, you pass bind values in order after the SQL string.
