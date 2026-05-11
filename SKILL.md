---
name: rust-hexagonal-architecture
description: Use when writing Rust backend/HTTP server code (axum, actix-web, etc.) — designing domain layers, repository or service traits, orchestrating I/O through ports and adapters, or organizing a new Rust service crate. Enforces the hexagonal architecture patterns from howtocodeit.com's "Master Hexagonal Architecture in Rust" guide. Do NOT use for CLI tools, scripts, embedded code, or services with almost no business logic — hexagonal adds overhead that only pays off when you have non-trivial domain logic or multiple swappable adapters.
---

# Rust Hexagonal Architecture

This skill makes Claude write Rust server code that cleanly separates domain logic from I/O, using the patterns from [Master Hexagonal Architecture in Rust](https://www.howtocodeit.com/guides/master-hexagonal-architecture-in-rust).

Canonical reference implementation: https://github.com/howtocodeit/hexarch/tree/3-simple-service — when in doubt, cross-check against this repo.

## When to apply

Apply when the task involves any of:

- Creating a new Rust backend/HTTP service
- Adding a new domain entity, repository, or service
- Adding an HTTP handler that calls into domain logic
- Refactoring a Rust service that has mixed concerns (e.g. sqlx calls inside handlers, domain types deriving `Deserialize`)
- Adding a new outbound integration (database, queue, external API) to an existing service

## When NOT to apply

Skip this skill (and tell the user) when:

- **Solo-dev project / prototype** — ceremony outweighs benefit
- **Minimal business logic** — a thin CRUD proxy doesn't need ports
- **Performance-critical hot path** — trait indirection and cloning may be unacceptable
- **CLI tools, scripts, build tools** — no adapters to swap
- **Library crates** — libraries shouldn't impose architecture on callers

If the user is clearly in one of these buckets, say so before writing code.

## Core rules (non-negotiable)

Check each one before declaring a file done.

1. **Domain does not import infrastructure.** No `sqlx`, `reqwest`, `axum`, `serde::Deserialize` inside `src/lib/domain/`. Domain types must compile without any framework dependency.
2. **Ports are traits with `Clone + Send + Sync + 'static` bounds** and return `impl Future<Output = ...> + Send`.
3. **Domain errors always carry a catch-all `Unknown(#[from] anyhow::Error)`** variant. Infrastructure errors convert into it via `?`.
4. **Separate request/response models from entities.** `CreateAuthorRequest` is not `Author`. Never reuse one for the other.
5. **Validated newtypes over raw strings/primitives.** `AuthorName(String)` with a `new() -> Result<Self, _>` constructor, no public field.
6. **Never derive `Deserialize`/`Serialize` on domain types.** Those derives live on transport types in the HTTP handler module.
7. **HTTP handlers do three things, in order:** (1) convert transport → domain via `try_into_domain()`, (2) call the service, (3) map domain result → transport response + `ApiError`.
8. **`main.rs` only bootstraps.** Config loading, adapter construction, service wiring, server start. No business logic.
9. **Public HTTP errors never echo domain error content verbatim.** `Unknown` variants map to generic `"Internal server error"`; details only in logs.
10. **Mocks implement the port trait at the boundary being tested.** Handler tests mock the `{Context}Service` port. Service tests mock `{Context}Repository` / `{Context}Metrics` / `{Entity}Notifier` ports.

## Required project layout

Every new service should look like this (crate has `[lib]` + `[[bin]]` split):

```
my-service/
├── Cargo.toml                     # [lib] path = "src/lib/lib.rs"; [[bin]] path = "src/bin/server/main.rs"
├── src/
│   ├── lib/
│   │   ├── lib.rs                 # pub mod config; pub mod domain; pub mod inbound; pub mod outbound;
│   │   ├── config.rs              # Config::from_env()
│   │   ├── domain.rs              # pub mod blog; (one per bounded context)
│   │   ├── domain/
│   │   │   └── blog.rs            # pub mod models; pub mod ports; pub mod service;
│   │   │   └── blog/
│   │   │       ├── models.rs      # pub mod author; (one per entity)
│   │   │       ├── models/
│   │   │       │   └── author.rs  # Author, AuthorName, EmailAddress, CreateAuthorRequest, CreateAuthorError
│   │   │       ├── ports.rs       # BlogService, BlogRepository, BlogMetrics, AuthorNotifier
│   │   │       └── service.rs     # Service<R, M, N>
│   │   ├── inbound.rs             # pub mod http;
│   │   ├── inbound/
│   │   │   └── http.rs            # HttpServer, HttpServerConfig, AppState
│   │   │   └── http/
│   │   │       ├── handlers.rs    # pub mod create_author; (one per operation)
│   │   │       └── handlers/
│   │   │           └── create_author.rs  # handler + ApiSuccess/ApiError + CreateAuthorHttpRequestBody
│   │   ├── outbound.rs            # pub mod sqlite; pub mod prometheus; pub mod email_client;
│   │   └── outbound/
│   │       ├── sqlite.rs
│   │       ├── prometheus.rs
│   │       └── email_client.rs
│   └── bin/
│       └── server/
│           └── main.rs            # bootstrap only
└── tests/
    └── integration/
```

See `reference/structure.md` for `Cargo.toml` details and variations.

## Naming conventions (memorize this table)

**Ports are named after the bounded context, not the entity.** `BlogService` contains authors; adding `Post` wouldn't create a new service.

| Concept              | Pattern                              | Example                             |
|----------------------|--------------------------------------|-------------------------------------|
| Bounded context      | Singular domain noun                 | `blog` (module name)                |
| Entity model         | Singular noun                        | `Author`                            |
| Validated newtype    | Entity + attribute                   | `AuthorName`, `EmailAddress`        |
| Request model        | `{Action}{Entity}Request`            | `CreateAuthorRequest`               |
| Response model       | `{Action}{Entity}ResponseData`       | `CreateAuthorResponseData`          |
| Repository port      | `{Context}Repository`                | `BlogRepository`                    |
| Service port         | `{Context}Service`                   | `BlogService`                       |
| Metrics port         | `{Context}Metrics`                   | `BlogMetrics`                       |
| Notifier port        | `{Entity}Notifier`                   | `AuthorNotifier` (entity-scoped OK) |
| Service impl         | `Service<R, M, N>`                   | `Service<Sqlite, Prometheus, …>`    |
| Adapter              | Technology name                      | `Sqlite`, `Prometheus`              |
| HTTP request body    | `{Action}{Entity}HttpRequestBody`    | `CreateAuthorHttpRequestBody`       |
| Domain error enum    | `{Action}{Entity}Error`              | `CreateAuthorError`                 |
| Transport error      | `ApiError`                           | `ApiError`                          |
| HTTP parse error     | `Parse{Action}{Entity}HttpRequestError` | aggregates newtype errors        |

## Workflow for common tasks

**Adding a new domain entity in an existing bounded context** → Add `src/lib/domain/<context>/models/<entity>.rs`, register it in `models.rs`. Extend the existing `{Context}Repository` and `{Context}Service` traits with new methods. Add a new error enum for the operation.

**Adding a new bounded context** → Copy `templates/domain_module.rs.tmpl`, adapt names. Register it in `src/lib/domain.rs`. Wire it into `main.rs`.

**Adding a new outbound integration** → Define a new port trait in `domain/<context>/ports.rs`, implement it in `outbound/<tech>.rs`, add it as a generic parameter to `Service<…>`.

**Adding an HTTP handler** → Create `src/lib/inbound/http/handlers/<operation>.rs`. Register in `handlers.rs`. Wire route in `http.rs`'s `api_routes()`.

**Refactoring a messy handler** → See `anti-patterns.md` for migration steps.

## Self-check before marking a task done

- [ ] No `use sqlx::`, `use axum::`, `use serde::` inside `src/lib/domain/`
- [ ] Every trait method returns `impl Future<...> + Send`
- [ ] Every domain error enum has `Unknown(#[from] anyhow::Error)`
- [ ] Every newtype has a fallible constructor and no public fields
- [ ] Ports are named after the bounded context (`BlogService`), not the entity
- [ ] HTTP handler body matches the three-step pattern (transport → domain → transport)
- [ ] `main.rs` has zero business logic
- [ ] Handler tests inject a mock of the `{Context}Service` port

## Reference files (read on demand)

- `reference/structure.md` — full directory layout, `Cargo.toml`, crate split
- `reference/ports.md` — trait signatures, why `Clone + Send + Sync + 'static`
- `reference/errors.md` — thiserror + anyhow pattern, domain↔transport error mapping
- `reference/handlers.md` — axum handler three-step pattern with full example
- `reference/testing.md` — MockService / MockRepository patterns, harness setup
- `anti-patterns.md` — what NOT to do, with Before/After examples
- `templates/domain_module.rs.tmpl` — copy-paste skeleton for a new bounded context
- `templates/adapter_sqlite.rs.tmpl` — copy-paste skeleton for sqlx adapters
- `templates/main.rs.tmpl` — bootstrap-only main.rs
