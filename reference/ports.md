# Ports: Trait Signatures

Ports are traits. There are two kinds:

- **Driving ports** (called by inbound adapters): `BlogService`
- **Driven ports** (called by the service, implemented by outbound adapters): `BlogRepository`, `BlogMetrics`, `AuthorNotifier`

## Naming: bounded context, not entity

The single biggest mistake: naming the service/repository after an entity (`AuthorService`, `AuthorRepository`). Don't do this. The canonical repo uses `BlogService` / `BlogRepository` — the bounded context is `blog`, and `Author` is merely one entity within it. Adding `Post` or `Comment` later does NOT create new services; it extends the existing `BlogService`.

Exception: operation-specific notifier/listener ports (like `AuthorNotifier`) are fine to name per entity when they're genuinely entity-scoped.

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

## Why each bound matters

- `Clone` — axum's `State<T>` extractor requires `T: Clone`. Cheap because impls hold `Arc<Pool>` or similar.
- `Send` — move across `.await` points in multi-threaded runtimes.
- `Sync` — `&self` shared across tasks.
- `'static` — axum holds state for the lifetime of the server.

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
