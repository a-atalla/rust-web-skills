# Rust Web Skills

A collection of skills for building web applications with Rust frameworks. Each skill is installed independently — load only what you need.

## Available Skills

| Skill | Description |
|-------|-------------|
| **leptos** | Build full-stack reactive web apps with Leptos. Components, signals, SSR/hydration, server functions, routing, and Axum/Actix integration. |
| **poem-openapi** | Build type-safe REST APIs with Poem and poem-openapi. Automatic OpenAPI v3 spec generation, Swagger UI, auth, file uploads, and more. |
| **sqlx** | Build type-safe database applications with SQLx. Compile-time checked queries, connection pooling (PostgreSQL/SQLite), migrations, transactions, and the sqlx CLI. |

## Install

```bash
# Install a specific skill
npx skills add a-atalla/rust-web-skills --skill leptos
npx skills add a-atalla/rust-web-skills --skill poem-openapi
npx skills add a-atalla/rust-web-skills --skill sqlx

# List all available skills
npx skills add a-atalla/rust-web-skills --list

# Install all skills
npx skills add a-atalla/rust-web-skills --all
```

## License

MIT
