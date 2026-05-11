# Project Structure Reference

Mirrors the canonical repo: https://github.com/howtocodeit/hexarch/tree/3-simple-service

## `Cargo.toml`

```toml
[package]
name = "my_service"
version = "0.1.0"
edition = "2021"

[lib]
name = "my_service"
path = "src/lib/lib.rs"

[[bin]]
name = "my_service_server"
path = "src/bin/server/main.rs"

[dependencies]
anyhow = "1"
axum = "0.7"
derive_more = "0.99"
serde = { version = "1", features = ["std", "derive"] }
sqlx = { version = "0.7", features = ["runtime-tokio", "sqlite", "macros"] }
thiserror = "1"
tokio = { version = "1", features = ["full"] }
tower-http = { version = "0.5", features = ["trace"] }
tower-layer = "0.3"
tracing = "0.1"
tracing-subscriber = "0.3"
uuid = { version = "1", features = ["v4", "fast-rng"] }
```

### Why `[lib]` + `[[bin]]`

- `lib.rs` holds all code, `bin/server/main.rs` only wires and calls `run()`.
- Integration tests in `tests/` can `use my_service::...` — impossible if everything lives in `main.rs`.
- Adding a second bin target (e.g. CLI admin tool, migration runner) is then trivial.

## Full layout

```
my-service/
├── Cargo.toml
├── migrations/                      # sqlx migrations
├── src/
│   ├── bin/
│   │   └── server/
│   │       └── main.rs              # bootstrap entry point
│   └── lib/
│       ├── lib.rs                   # pub mod config; pub mod domain; pub mod inbound; pub mod outbound;
│       ├── config.rs                # Config + from_env()
│       ├── domain.rs                # pub mod blog;
│       ├── domain/
│       │   └── blog.rs              # pub mod models; pub mod ports; pub mod service;
│       │   └── blog/
│       │       ├── models.rs        # pub mod author;
│       │       ├── models/
│       │       │   └── author.rs    # entity + newtypes + request + error
│       │       ├── ports.rs         # BlogService, BlogRepository, BlogMetrics, AuthorNotifier
│       │       └── service.rs       # Service<R, M, N>
│       ├── inbound.rs               # pub mod http;
│       ├── inbound/
│       │   └── http.rs              # HttpServer, HttpServerConfig, AppState, api_routes()
│       │   └── http/
│       │       ├── handlers.rs      # pub mod create_author;
│       │       ├── handlers/
│       │       │   └── create_author.rs
│       │       └── responses.rs     # shared generic response envelopes (optional)
│       ├── outbound.rs              # pub mod sqlite; pub mod prometheus; pub mod email_client;
│       └── outbound/
│           ├── sqlite.rs
│           ├── prometheus.rs
│           └── email_client.rs
└── tests/
    └── integration/
```

## Why this layout

**`domain/` is the center of the hexagon.** It owns business rules and must compile without framework deps. If you `cargo check` the domain module alone and it builds with only `std`, `thiserror`, `anyhow`, `uuid`, `derive_more` — you're doing it right.

**`inbound/` drives the hexagon.** HTTP handlers, CLI commands, message consumers. Translates external inputs into domain inputs.

**`outbound/` is driven by the hexagon.** Database clients, HTTP clients, mailers. Translates domain requests into external system calls.

## One module per domain

Entities that must change together live in the same domain, in the same module. `Author`, `Post`, `Comment` that need atomic deletion all live under `src/lib/domain/blog/models/`. A *separate* domain (e.g. auth/`User` management, which the article's "Authentication and authorization" section treats specifically) would be `src/lib/domain/auth/`.

**Start with one domain.** Per the article: *"Start with a single, large domain."* Don't predict future boundaries — you'll guess wrong and pay to undo it. Split only when you experience real friction (independent rates of change, cross-team ownership, genuine need for different deployment cadences).

**Port naming tracks the domain module name.** Ports (`{Domain}Service`, `{Domain}Repository`) are named after the domain. If the module is `author` with only an `Author` entity, the port is `AuthorService`. If the module is `blog` with `Author`, `Post`, `Comment`, the port is `BlogService`, and `BlogService::create_author` + `BlogService::create_post` live on the same trait.

## One handler file per operation

`handlers.rs` is just `pub mod create_author; pub mod get_author; …`. Each `handlers/<op>.rs` file contains the handler function plus its request body type, response data type, and any operation-specific error mappings. This keeps each file under ~200 lines.

## When to split into multiple crates

The article doesn't mandate a crate split — the single-crate `[lib]` + `[[bin]]` layout works for the full teaching example. Consider splitting into a workspace when you see concrete pressure:

- Domain code is large enough that compile times suffer when you touch unrelated adapter code
- You want the compiler (not just convention) to enforce "domain cannot import adapters"
- Multiple binaries (server, CLI admin tool, migration runner) all consume the domain

A workspace layout:

```
Cargo.toml                  # workspace
crates/
├── domain/                 # depends only on std + thiserror + anyhow + uuid + derive_more
├── adapters/               # depends on domain
└── app/                    # depends on domain + adapters, contains main.rs
```

Crate boundaries make the dependency rule compiler-enforced: domain *cannot* import adapters because it's a separate crate. But this is purely an ergonomic/safety upgrade — the article's single-crate layout is architecturally equivalent.
