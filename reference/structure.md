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

## One module per bounded context

`Author`, `Post`, `Comment` all belong to the `blog` bounded context → they live under `src/lib/domain/blog/models/`. A *different* bounded context (e.g. `billing`) would be `src/lib/domain/billing/`.

**Ports (`BlogService`, `BlogRepository`) are named after the bounded context, not the entity.** `BlogService.create_author` and `BlogService.create_post` both live on the same trait. This is the single biggest naming point people get wrong.

## One handler file per operation

`handlers.rs` is just `pub mod create_author; pub mod get_author; …`. Each `handlers/<op>.rs` file contains the handler function plus its request body type, response data type, and any operation-specific error mappings. This keeps each file under ~200 lines.

## When to split into multiple crates

Once domain code passes ~5k lines or multiple bounded contexts emerge, split:

```
Cargo.toml                  # workspace
crates/
├── domain/                 # depends only on std + thiserror + anyhow + uuid + derive_more
├── adapters/               # depends on domain
└── app/                    # depends on domain + adapters, contains main.rs
```

Crate boundaries make the dependency rule compiler-enforced: domain *cannot* import adapters because it's a separate crate.
