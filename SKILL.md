---
name: rust-hexagonal-architecture
description: Use when writing Rust backend/HTTP server code (axum, actix-web, etc.) — designing domain layers, repository or service traits, orchestrating I/O through ports and adapters, or organizing a new Rust service crate. Enforces the hexagonal architecture patterns from howtocodeit.com's "Master Hexagonal Architecture in Rust" guide. Do NOT use for CLI tools, scripts, embedded code, or services with almost no business logic — hexagonal adds overhead that only pays off when you have non-trivial domain logic or multiple swappable adapters.
---

# Rust Hexagonal Architecture

This skill makes Claude write Rust server code that cleanly separates domain logic from I/O, using the patterns from [Master Hexagonal Architecture in Rust](https://www.howtocodeit.com/guides/master-hexagonal-architecture-in-rust).

Canonical sources:
- Article: https://github.com/howtocodeit/article-md/blob/main/guides/master-hexagonal-architecture-in-rust.md (authoritative text)
- Reference implementation: https://github.com/howtocodeit/hexarch/tree/3-simple-service (authoritative code)

The article uses `AuthorService` / `AuthorRepository` throughout (single-entity teaching example). The `3-simple-service` branch refactors to `BlogService` / `BlogRepository` because it models the author as one entity inside a `blog` domain. Both naming styles are valid — which one to use is a **domain-boundary decision** (see rule 11 below).

## When to apply

Apply when the task involves any of:

- Creating a new Rust backend/HTTP service
- Adding a new domain entity, repository, or service
- Adding an HTTP handler that calls into domain logic
- Refactoring a Rust service that has mixed concerns (e.g. sqlx calls inside handlers, domain types deriving `Deserialize`)
- Adding a new outbound integration (database, queue, external API) to an existing service

## When NOT to apply

Skip this skill (and tell the user) when the article's Part IV trade-offs clearly apply:

- **Apps that don't have any business logic** — "there's nothing to encapsulate" (article Part IV). A CRUD proxy just piping requests into a DB gains nothing from ports.
- **Performance-critical hot paths** — "if you ever find yourself wondering if rustc is outputting the optimal assembly… you don't have the nanoseconds to spare for trait dispatch."
- **Solo-dev prototypes that will be thrown away** — the cost of ceremony outweighs long-term benefits you'll never realize.
- **CLI tools, scripts, build tools** — no adapters to swap.
- **Library crates** — libraries shouldn't impose architecture on callers.

If the user is clearly in one of these buckets, say so before writing code. The article is emphatic that hexagonal is a *trade-off*, not a default.

## Core rules

Check each one before declaring a file done. Rules 1–10 are hard — follow them unless you can articulate why the article's trade-off analysis doesn't apply. Rule 11 is a judgement call.

1. **Domain does not import framework-specific infrastructure.** No `sqlx`, `reqwest`, `axum` inside `src/lib/domain/`. Pervasive, stable deps (`tokio`, `anyhow`, `uuid`, `thiserror`, `derive_more`) are acceptable — the article explicitly permits them because "their utility is so general, and community adoption so extensive, that the odds of ever needing to replace them are slim."
2. **Ports are traits with `Clone + Send + Sync + 'static` bounds** and return `impl Future<Output = ...> + Send`. These bounds exist because axum holds the state behind an `Arc` across multi-threaded tokio tasks for the lifetime of the server; if your runtime is single-threaded or your port isn't shared as axum state, some bounds may be unnecessary (rare).
3. **Domain errors always carry a catch-all `Unknown(#[from] anyhow::Error)`** variant. Infrastructure errors convert into it via `?`. Keep enums exhaustive (no `#[non_exhaustive]`) for application code — let the compiler flag un-updated match sites. `#[non_exhaustive]` is only appropriate if you're publishing the domain as a library.
4. **Separate request/response models from entities.** `CreateAuthorRequest` is not `Author`. The article calls this "duplicative boilerplate that isn't" — the two models diverge as the system grows.
5. **Validated newtypes over raw strings/primitives.** `AuthorName(String)` with a `new() -> Result<Self, _>` constructor, no public field.
6. **Do not derive `Deserialize`/`Serialize` on domain types that perform validation.** The article ("Don't be abSerde") permits serde on domain types ONLY if both: (a) you want adapters to serialize domain models directly, AND (b) the model does no validation. Since rule 5 mandates validated newtypes, in practice this means: serde derives live on transport types.
7. **HTTP handlers do three things, in order:** (1) convert transport → domain via `try_into_domain()`, (2) call the service, (3) map domain result → transport response + `ApiError`.
8. **`main.rs` only bootstraps.** Config loading, adapter construction, service wiring, server start. No business logic.
9. **Public HTTP errors never echo domain error content verbatim.** `Unknown` variants map to generic `"Internal server error"`; details only in logs.
10. **Mocks implement the port trait at the boundary being tested.** Handler tests mock the service port. Service tests mock the repository / metrics / notifier ports.
11. **Start with one large domain.** Per the article: "Start with a single, large domain." Do NOT split into multiple bounded contexts upfront. Add a second context only when you experience real friction (different rates of change, independent teams, legitimate cross-domain boundaries). Naming follows from boundaries: if your domain is `author` (single entity), call the ports `AuthorService` / `AuthorRepository`. If it grows to include posts, comments, etc., rename to `BlogService` / `BlogRepository`. See `reference/structure.md` for migration guidance.

### Key quotes from the article

- *"Start with a single, large domain."*
- *"A domain should include all entities that must change together as part of a single, atomic operation."*
- *"If you leak transactions into your business logic to perform cross-domain operations atomically, your domain boundaries are wrong."*
- *"`CreateAuthorError` is a complete description of everything that can go wrong when creating an author."*

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

## Naming conventions

**Ports are named after the domain, not any single entity within it.** A domain containing only `Author` is legitimately called the `author` domain (article's teaching example → `AuthorService`). A domain containing `Author`, `Post`, `Comment` is the `blog` domain (reference repo → `BlogService`). Pick one based on the actual scope of your domain — rule 11 says start small, expand the name only when the domain genuinely grows.

| Concept              | Pattern                                   | Example(s)                                      |
|----------------------|-------------------------------------------|-------------------------------------------------|
| Domain module name   | Singular domain noun                      | `author` (single-entity), `blog` (multi-entity) |
| Entity model         | Singular noun                             | `Author`                                        |
| Validated newtype    | Entity + attribute                        | `AuthorName`, `EmailAddress`                    |
| Request model        | `{Action}{Entity}Request`                 | `CreateAuthorRequest`                           |
| Response model       | `{Action}{Entity}ResponseData`            | `CreateAuthorResponseData`                      |
| Repository port      | `{Domain}Repository`                      | `AuthorRepository` or `BlogRepository`          |
| Service port         | `{Domain}Service`                         | `AuthorService` or `BlogService`                |
| Metrics port         | `{Domain}Metrics`                         | `AuthorMetrics` or `BlogMetrics`                |
| Notifier port        | `{Entity}Notifier` (entity-scoped OK)     | `AuthorNotifier`                                |
| Service impl         | `Service<R, M, N>`                        | `Service<Sqlite, Prometheus, …>`                |
| Adapter              | Technology name                           | `Sqlite`, `Prometheus`                          |
| HTTP request body    | `{Action}{Entity}HttpRequestBody`         | `CreateAuthorHttpRequestBody`                   |
| Domain error enum    | `{Action}{Entity}Error`                   | `CreateAuthorError`                             |
| Transport error      | `ApiError`                                | `ApiError`                                      |
| HTTP parse error     | `Parse{Action}{Entity}HttpRequestError`   | aggregates newtype errors                       |

## Workflow for common tasks

**Adding a new domain entity in the existing domain** → Add `src/lib/domain/<domain>/models/<entity>.rs`, register it in `models.rs`. Extend the existing `{Domain}Repository` and `{Domain}Service` traits with new methods. Add a new error enum for the operation. If adding the entity means the domain's name is now misleading (`author` domain now holds posts), rename the domain module to match (`blog`).

**Adding a second domain** → Only do this when you have concrete friction — independent rates of change, different teams, or a legitimate cross-domain boundary (e.g. auth/user management in a large app, per article's "Authentication and authorization" section). Otherwise, per article rule 11, keep adding to the existing domain. When you do split: copy `templates/domain_module.rs.tmpl`, adapt names, register in `src/lib/domain.rs`, wire into `main.rs`.

**Adding a new outbound integration** → Define a new port trait in `domain/<context>/ports.rs`, implement it in `outbound/<tech>.rs`, add it as a generic parameter to `Service<…>`.

**Adding an HTTP handler** → Create `src/lib/inbound/http/handlers/<operation>.rs`. Register in `handlers.rs`. Wire route in `http.rs`'s `api_routes()`.

**Refactoring a messy handler** → See `anti-patterns.md` for migration steps.

## Self-check before marking a task done

- [ ] No `use sqlx::`, `use axum::`, `use reqwest::`, `use serde::` inside `src/lib/domain/`
- [ ] Every trait method returns `impl Future<...> + Send`
- [ ] Every domain error enum has `Unknown(#[from] anyhow::Error)`
- [ ] Every newtype has a fallible constructor and no public fields
- [ ] Ports are named after the domain module (`AuthorService` for `author` domain, `BlogService` for `blog` domain)
- [ ] HTTP handler body matches the three-step pattern (transport → domain → transport)
- [ ] `main.rs` has zero business logic
- [ ] Handler tests inject a mock of the domain's `Service` port

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
