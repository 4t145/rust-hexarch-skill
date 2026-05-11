# Ports: Trait Signatures

Ports are traits. There are two kinds:

- **Driving ports** (called by inbound adapters): `BlogService`
- **Driven ports** (called by the service, implemented by outbound adapters): `BlogRepository`, `BlogMetrics`, `AuthorNotifier`

## Naming: after the domain, whatever its scope

The port represents the whole domain's API. Its name should match the domain module it lives in.

- **Article teaching example** (Parts II–III): the domain is simply `author` (one entity). Ports are `AuthorService` / `AuthorRepository`. Perfectly fine.
- **Reference repo `3-simple-service` branch**: the same code refactored to treat the author as one entity in a larger `blog` domain. Ports become `BlogService` / `BlogRepository`.

Both are valid. The decision is made by **where you draw the domain boundary** — not by a naming rule. The article's guidance: *"Start with a single, large domain"* and split only when friction demands it. If your domain contains one entity today, `AuthorService` is correct; rename it to `BlogService` (or whatever) when the domain actually grows.

**Anti-pattern to avoid:** creating `AuthorService` AND `PostService` AND `CommentService` when all three entities must change atomically together. That means you have *one* domain with three entities, and you need *one* `BlogService` — see `anti-patterns.md` #1.

**Exception:** operation-specific notifier/listener ports (like `AuthorNotifier`) are fine to name per entity when they're genuinely entity-scoped — the reference repo does this.

## Canonical trait pattern

```rust
// src/lib/domain/blog/ports.rs
use std::future::Future;

use crate::domain::blog::models::author::{Author, CreateAuthorRequest};
#[allow(unused_imports)]  // used in doc comments
use crate::domain::blog::models::author::AuthorName;
use crate::domain::blog::models::author::CreateAuthorError;

/// `BlogService` is the public API for the blog domain.
pub trait BlogService: Clone + Send + Sync + 'static {
    /// Asynchronously create a new [Author].
    ///
    /// # Errors
    /// - [CreateAuthorError::Duplicate] if an [Author] with the same [AuthorName] already exists.
    fn create_author(
        &self,
        req: &CreateAuthorRequest,
    ) -> impl Future<Output = Result<Author, CreateAuthorError>> + Send;
}

pub trait BlogRepository: Send + Sync + Clone + 'static {
    fn create_author(
        &self,
        req: &CreateAuthorRequest,
    ) -> impl Future<Output = Result<Author, CreateAuthorError>> + Send;
}

pub trait BlogMetrics: Send + Sync + Clone + 'static {
    fn record_author_creation_success(&self) -> impl Future<Output = ()> + Send;
    fn record_author_creation_failure(&self) -> impl Future<Output = ()> + Send;
}

pub trait AuthorNotifier: Send + Sync + Clone + 'static {
    fn author_created(&self, author: &Author) -> impl Future<Output = ()> + Send;
}
```

## Why each bound matters (and when you could drop one)

The article walks through these bounds one at a time, adding each only when the concrete need appears. The full set `Clone + Send + Sync + 'static` is what you end up with for an axum + multi-threaded tokio server — the most common case. But understand *why* each is there:

- `Send` — the future returned by the trait method moves across `.await` points on multi-threaded runtimes. Not needed on single-threaded runtimes (rare).
- `Sync` — `&self` is shared across tasks. Not needed if every call has exclusive access.
- `'static` — axum holds state for the lifetime of the server. Not needed if your port isn't shared as long-lived framework state.
- `Clone` — axum's `State<T>` extractor requires `T: Clone`. Impls are cheap to clone because they hold `Arc<Pool>` or similar. Not needed if you aren't using axum-style state injection.

In practice: write all four bounds by default. Deviate only if you know why you're deviating.

## Why `impl Future<...> + Send` instead of `async fn`

As of Rust 1.75 `async fn` is allowed in traits, BUT the returned future doesn't automatically get `+ Send` bounds, breaking multi-threaded axum. The explicit `impl Future<Output = _> + Send` form is the robust choice until [return-type notation](https://github.com/rust-lang/rust/issues/109417) is stable.

Note: you CAN use `async fn` inside `impl Trait for Type` blocks even when the trait declares `impl Future` — the compiler desugars them compatibly. The reference impl does this:

```rust
impl<R, M, N> BlogService for Service<R, M, N> /* ... */ {
    async fn create_author(&self, req: &CreateAuthorRequest) -> Result<Author, CreateAuthorError> {
        // ...
    }
}
```

## Service impl wiring multiple ports

```rust
// src/lib/domain/blog/service.rs
use crate::domain::blog::models::author::{Author, CreateAuthorError, CreateAuthorRequest};
use crate::domain::blog::ports::{AuthorNotifier, BlogMetrics, BlogRepository, BlogService};

#[derive(Debug, Clone)]
pub struct Service<R, M, N>
where
    R: BlogRepository,
    M: BlogMetrics,
    N: AuthorNotifier,
{
    repo: R,
    metrics: M,
    author_notifier: N,
}

impl<R, M, N> Service<R, M, N>
where
    R: BlogRepository,
    M: BlogMetrics,
    N: AuthorNotifier,
{
    pub fn new(repo: R, metrics: M, author_notifier: N) -> Self {
        Self { repo, metrics, author_notifier }
    }
}

impl<R, M, N> BlogService for Service<R, M, N>
where
    R: BlogRepository,
    M: BlogMetrics,
    N: AuthorNotifier,
{
    async fn create_author(&self, req: &CreateAuthorRequest) -> Result<Author, CreateAuthorError> {
        let result = self.repo.create_author(req).await;
        if result.is_err() {
            self.metrics.record_author_creation_failure().await;
        } else {
            self.metrics.record_author_creation_success().await;
            self.author_notifier
                .author_created(result.as_ref().unwrap())
                .await;
        }
        result
    }
}
```

## Why separate ports for metrics/notifier instead of one fat trait

Because a no-op metrics impl should not require implementing notification logic, and vice versa. Small traits compose — the `Service` struct takes three generics, one per concern.
