---
name: sqlx
description: Build type-safe database applications in Rust with SQLx. Use this skill whenever the user mentions SQLx, database queries in Rust, SQL migrations, connection pooling, PostgreSQL, SQLite, compile-time query checking, or the sqlx CLI. Also use when the user asks about Rust database drivers, async SQL, query macros (query!, query_as!), FromRow derive, offline mode, cargo sqlx prepare, or embedding migrations. Covers PostgreSQL and SQLite drivers, the full macro system, migration management, connection pooling, transactions, and the sqlx CLI tool.
---

# Building Database Applications with SQLx

SQLx is an async, pure-Rust SQL crate with compile-time query verification. It connects to your database at compile time to verify queries and infer types — not an ORM, but a type-safe SQL toolkit.

## When to read reference files

- `references/macros-and-types.md` — Full reference for query macros (query!, query_as!, query_scalar!, etc.), type overrides, FromRow derive attributes, and all macro variants. Read when you need exact macro syntax.
- `references/migrations-and-cli.md` — sqlx CLI commands, migration file format, embedding migrations, offline mode setup, and sqlx.toml configuration. Read when setting up migrations or CI.
- `references/pg-and-sqlite.md` — PostgreSQL and SQLite specific types, connection options, PgListener, advisory locks, SQLite extensions, and database-specific patterns. Read when you need driver-specific features.

---

## Project Setup

### Cargo.toml

```toml
[dependencies]
sqlx = { version = "0.9", features = ["runtime-tokio", "tls-native-tls", "postgres", "macros", "migrate"] }
# For SQLite instead:
# sqlx = { version = "0.9", features = ["runtime-tokio", "sqlite", "macros", "migrate"] }
tokio = { version = "1", features = ["macros", "rt-multi-thread"] }
serde = { version = "1", features = ["derive"] }
anyhow = "1"
```

### Feature Combinations

| Need | Features |
|---|---|
| PostgreSQL + Tokio + native TLS | `runtime-tokio`, `tls-native-tls`, `postgres` |
| PostgreSQL + Tokio + RustTLS | `runtime-tokio`, `tls-rustls-ring`, `postgres` |
| SQLite + Tokio | `runtime-tokio`, `sqlite` |
| SQLite bundled (no system lib) | `runtime-tokio`, `sqlite`, `sqlite-bundled` |
| Compile-time query checking | `macros` |
| FromRow/Type derives | `derive` |
| Embedded migrations | `migrate` |
| JSON column support | `json` |
| chrono date/time | `chrono` |
| time crate date/time | `time` |
| UUID support | `uuid` |

### Environment Variable

```bash
export DATABASE_URL="postgres://user:pass@localhost/mydb"
# or
export DATABASE_URL="sqlite:./mydb.sqlite"
# or for in-memory SQLite:
export DATABASE_URL="sqlite::memory:"
```

---

## Core Architecture

```
Your code
  ↓
query!() / query_as!() macros ← compile-time: connect to DB, verify SQL, infer types
  ↓
Pool<DB> ← manages connections, shared via Arc (Clone is cheap)
  ↓
Driver (PgConnection / SqliteConnection) ← async wire protocol
  ↓
Database
```

---

## Connection Pool

### PostgreSQL

```rust
use sqlx::postgres::PgPoolOptions;

let pool = PgPoolOptions::new()
    .max_connections(20)
    .min_connections(5)
    .acquire_timeout(Duration::from_secs(5))
    .idle_timeout(Duration::from_secs(600))
    .max_lifetime(Duration::from_secs(1800))
    .connect(&database_url)
    .await?;
```

### SQLite

```rust
use sqlx::sqlite::SqlitePoolOptions;

let pool = SqlitePoolOptions::new()
    .max_connections(5)
    .connect("sqlite:./app.db").await?;

// In-memory with file backing:
let pool = SqlitePoolOptions::new()
    .max_connections(1)  // must be 1 for in-memory
    .connect("sqlite::memory:").await?;
```

### SQLite Connection Options

```rust
use sqlx::sqlite::SqliteConnectOptions;
use std::str::FromStr;

let options = SqliteConnectOptions::from_str("sqlite:./app.db")?
    .create_if_missing(true)
    .journal_mode(sqlx::sqlite::SqliteJournalMode::Wal)
    .synchronous(sqlx::sqlite::SqliteSynchronous::Normal)
    .busy_timeout(Duration::from_secs(5));

let pool = SqlitePoolOptions::new()
    .connect_with(options).await?;
```

---

## Querying

### Compile-Time Checked Queries (query! macro)

The `query!` macro connects to your database at compile time, verifies the SQL, and generates anonymous structs for results:

```rust
// Returns anonymous struct with inferred fields
let rec = sqlx::query!(
    r#"SELECT id, description, done FROM todos WHERE id = $1"#,
    todo_id
)
.fetch_one(&pool)
.await?;

println!("{}: {} (done: {})", rec.id, rec.description, rec.done);
```

**PostgreSQL placeholders:** `$1`, `$2`, `$3`, ...
**SQLite placeholders:** `?1`, `?2`, `?3`, ... (or just `?`)

### Typed Queries (query_as! macro)

Map results directly to your structs:

```rust
#[derive(Debug, sqlx::FromRow)]
struct Todo {
    id: i64,
    description: String,
    done: bool,
}

let todos = sqlx::query_as!(
    Todo,
    r#"SELECT id, description, done FROM todos ORDER BY id"#
)
.fetch_all(&pool)
.await?;
```

### Scalar Queries

```rust
let count: i64 = sqlx::query_scalar!(
    r#"SELECT COUNT(*) FROM todos WHERE done = FALSE"#
)
.fetch_one(&pool)
.await?;
```

### Dynamic Queries (no macro)

For queries built at runtime:

```rust
let todos = sqlx::query_as::<_, Todo>(
    "SELECT id, description, done FROM todos WHERE done = $1"
)
.bind(false)
.fetch_all(&pool)
.await?;
```

### Query Builder

For complex dynamic queries:

```rust
use sqlx::QueryBuilder;

let mut builder = QueryBuilder::new("SELECT * FROM users WHERE ");
let mut separated = builder.separated(" AND ");

if let Some(name) = &filter.name {
    separated.push("name = ").push_bind_unseparated(name);
}
if let Some(active) = filter.active {
    separated.push("active = ").push_bind_unseparated(active);
}

let users = builder.build_query_as::<User>().fetch_all(&pool).await?;
```

### Executor Methods

| Method | Returns | Use case |
|---|---|---|
| `.execute(pool)` | `QueryResult` (rows_affected) | INSERT, UPDATE, DELETE |
| `.fetch_one(pool)` | `T` (exactly one) | Get single required row |
| `.fetch_optional(pool)` | `Option<T>` | Get single optional row |
| `.fetch_all(pool)` | `Vec<T>` | Get all matching rows |
| `.fetch(pool)` | `Stream<T>` | Stream large results |

---

## Type Mappings

### PostgreSQL

| Rust Type | PostgreSQL Type |
|---|---|
| `bool` | BOOL |
| `i16`, `i32`, `i64` | SMALLINT, INT, BIGINT |
| `f32`, `f64` | REAL, DOUBLE PRECISION |
| `&str`, `String` | TEXT, VARCHAR, CHAR |
| `&[u8]`, `Vec<u8>` | BYTEA |
| `chrono::NaiveDateTime` | TIMESTAMP |
| `chrono::DateTime<Utc>` | TIMESTAMPTZ |
| `chrono::NaiveDate` | DATE |
| `chrono::NaiveTime` | TIME |
| `uuid::Uuid` | UUID |
| `serde_json::Value` | JSON, JSONB |
| `sqlx::types::Json<T>` | JSON, JSONB (typed) |
| `BigDecimal` / `Decimal` | NUMERIC |

### SQLite

| Rust Type | SQLite Type |
|---|---|
| `bool` | INTEGER (0/1) |
| `i32`, `i64` | INTEGER |
| `f64` | REAL |
| `&str`, `String` | TEXT |
| `&[u8]`, `Vec<u8>` | BLOB |
| `chrono::NaiveDateTime` | TEXT (ISO 8601) |
| `uuid::Uuid` | BLOB or TEXT |
| `sqlx::types::Json<T>` | TEXT |

### Nullable Columns

Use `Option<T>` in your structs:

```rust
#[derive(FromRow)]
struct User {
    id: i64,
    name: String,
    email: Option<String>,  // nullable column
}
```

### Type Overrides in Macros

When the macro can't infer the right type:

```rust
let rec = sqlx::query!(
    r#"SELECT id, created_at as "created_at: chrono::NaiveDateTime" FROM events"#
)
.fetch_one(&pool)
.await?;
```

Override syntax in column aliases:
- `col!` — force not-null
- `col?` — force nullable
- `col: Type` — custom type
- `col!: Type` — not-null + custom type
- `col?: Type` — nullable + custom type
- `col: _` — infer from struct field

---

## FromRow Derive

```rust
#[derive(sqlx::FromRow)]
struct User {
    id: i64,
    username: String,
    // Rename column
    #[sqlx(rename = "email_address")]
    email: String,
    // Flatten nested struct
    #[sqlx(flatten)]
    audit: AuditInfo,
    // Default on null
    #[sqlx(default)]
    avatar_url: Option<String>,
    // JSON column
    #[sqlx(json)]
    metadata: UserMeta,
    // Skip this field
    #[sqlx(skip)]
    computed: String,
}
```

---

## Transactions

```rust
// Callback-based (auto-commit/rollback)
pool.begin().await?
    .execute("INSERT INTO users (name) VALUES ($1)")
    .bind("Alice")
    .await?
    .execute("INSERT INTO logs (action) VALUES ($1)")
    .bind("user_created")
    .await?
    .commit().await?;

// Manual transaction management
let mut tx = pool.begin().await?;

sqlx::query!("INSERT INTO posts (title) VALUES ($1)", title)
    .execute(&mut *tx)  // deref to inner connection
    .await?;

sqlx::query!("UPDATE counters SET post_count = post_count + 1")
    .execute(&mut *tx)
    .await?;

tx.commit().await?;  // or tx.rollback().await?
```

---

## Migrations

### Creating Migrations

```bash
# Install CLI
cargo install sqlx-cli --no-default-features --features postgres
# or for SQLite:
cargo install sqlx-cli --no-default-features --features sqlite

# Create database and run migrations
sqlx database create
sqlx database setup    # create + migrate
sqlx database reset    # drop + create + migrate

# Create a new migration
sqlx migrate add create_users_table
# Reversible migration (up + down):
sqlx migrate add --reversible create_users_table
```

### Migration Files

Simple migration: `migrations/20240101_000001_create_users.sql`
```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    username TEXT NOT NULL UNIQUE,
    email TEXT NOT NULL
);
```

Reversible migration: `migrations/20240101_000001_create_users.up.sql`
```sql
-- UP: apply
CREATE TABLE users (id BIGSERIAL PRIMARY KEY, name TEXT NOT NULL);
```
`migrations/20240101_000001_create_users.down.sql`
```sql
-- DOWN: revert
DROP TABLE users;
```

### Embedding Migrations in Binary

```rust
use sqlx::migrate::Migrator;

static MIGRATOR: Migrator = sqlx::migrate!(); // reads from ./migrations at compile time

async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let pool = PgPool::connect(&env::var("DATABASE_URL")?).await?;
    MIGRATOR.run(&pool).await?;
    Ok(())
}
```

### CLI Migration Commands

```bash
sqlx migrate run              # Run pending migrations
sqlx migrate revert           # Revert last migration
sqlx migrate info             # Show migration status
sqlx migrate build-script     # Generate build.rs for recompilation on change
```

---

## Offline Mode (CI without a database)

Compile-time query checking normally requires a live database. Offline mode stores query metadata in JSON files so CI builds work without DATABASE_URL.

### Setup

```bash
# 1. Generate offline query data (run with a live DB)
cargo sqlx prepare

# 2. Commit the .sqlx/ directory
git add .sqlx/
git commit -m "Add sqlx offline query data"
```

### CI Configuration

```bash
# In CI, set the env var:
SQLX_OFFLINE=true cargo build
```

This tells the macros to read from `.sqlx/` instead of connecting to a database.

### Verify Freshness

```bash
# Fails if .sqlx/ is out of date
cargo sqlx prepare --check
```

### How It Works

When `SQLX_OFFLINE=true`, the `query!` macro:
1. Hashes the SQL string
2. Looks up `query-<hash>.json` in `.sqlx/`
3. Uses the stored metadata for type inference

Each JSON file contains: SQL, parameters, columns, and their types.

---

## sqlx.toml Configuration

```toml
[macros]
# Prefer chrono over time for datetime types
preferred_crates = { date_time = "chrono" }
# Override types globally
type_overrides = { "uuid" = "uuid::Uuid" }

[migrate]
# Custom migrations directory
migrations_dir = "db/migrations"
# Custom migration table name
table_name = "_sqlx_migrations"
# Auto-create schemas (Postgres)
create_schemas = true
```

---

## Testing with #[sqlx::test]

```rust
#[sqlx::test]
async fn test_basic(pool: SqlitePool) {
    // Pool connected to a temporary in-memory database
    sqlx::query("INSERT INTO users (name) VALUES ($1)")
        .bind("test")
        .execute(&pool)
        .await
        .unwrap();
}

#[sqlx::test(migrations = "./migrations")]
async fn test_with_migrations(pool: PgPool) {
    // Pool with migrations already applied
    let count = sqlx::query_scalar!("SELECT COUNT(*) FROM users")
        .fetch_one(&pool)
        .await
        .unwrap();
    assert_eq!(count, 0);
}

#[sqlx::test(migrations = "./migrations", fixtures("users", "posts"))]
async fn test_with_fixtures(pool: PgPool) {
    // Pool with migrations + fixture data loaded
    // fixtures are SQL files in fixtures/ directory
}
```

---

## Error Handling

```rust
use sqlx::Error;

match sqlx::query!("INSERT INTO users (username) VALUES ($1)", username)
    .execute(pool)
    .await
{
    Ok(result) => println!("Inserted {} row(s)", result.rows_affected()),
    Err(Error::Database(e)) => {
        if let Some(constraint) = e.constraint() {
            if constraint == "users_username_key" {
                return Err(AppError::Conflict("Username taken".into()));
            }
        }
        return Err(AppError::Internal(e.into()));
    }
    Err(e) => return Err(AppError::Internal(e.into())),
}
```

---

## Common Patterns

1. **query! for reads, query! with execute for writes** — the macro gives you type safety for free. Use it.

2. **Pool is cheap to clone** — `Pool<DB>` is `Arc`-based. Pass clones, not references. `&Pool` implements `Executor` directly.

3. **Use FromRow for manual queries** — When you can't use `query_as!` (dynamic SQL), use `query_as::<_, YourStruct>()` with `#[derive(FromRow)]`.

4. **Option<T> for nullable columns** — SQL NULL maps to `Option<T>` in Rust. Always.

5. **Transactions need deref** — In SQLx 0.7+, pass `&mut *tx` to query methods, not just `&mut tx`.

6. **SQLite uses ?1 not $1** — PostgreSQL uses positional `$N`, SQLite uses `?N` or `?`.

7. **Offline mode for CI** — Always `cargo sqlx prepare` and commit `.sqlx/` before pushing. Set `SQLX_OFFLINE=true` in CI.

8. **Use migrate!() to embed migrations** — Don't require the CLI in production. Embed migrations into your binary.

9. **PoolOptions before connect** — Configure pool size, timeouts, and callbacks before calling `.connect()`. Use `.connect_lazy()` if the DB might not be available immediately.

10. **json feature for JSON columns** — Use `sqlx::types::Json<T>` with the `json` feature to map JSON columns to typed Rust structs.
