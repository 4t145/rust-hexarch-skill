# Testing

## Testing philosophy

Mock at the port boundary that the code under test depends on:

| Code under test            | Mocked port                                     |
|----------------------------|-------------------------------------------------|
| HTTP handler               | `{Context}Service` (e.g. `BlogService`)         |
| Domain `Service` impl      | `{Context}Repository`, `{Context}Metrics`, `{Entity}Notifier` |
| Adapter (e.g. `Sqlite`)    | Real infrastructure (in-memory sqlite, testcontainers) |

Do NOT mock third-party libraries (`sqlx::SqlitePool`, `reqwest::Client`). Mocks live at port traits you own.

## Mocking the service port (for handler tests)

```rust
// inside create_author.rs #[cfg(test)] mod tests
use std::mem;
use std::sync::{Arc, Mutex};

use anyhow::anyhow;
use uuid::Uuid;

use crate::domain::blog::models::author::{
    Author, AuthorName, CreateAuthorError, CreateAuthorRequest, EmailAddress,
};
use crate::domain::blog::ports::BlogService;

use super::*;

#[derive(Clone)]
struct MockBlogService {
    create_author_result: Arc<Mutex<Result<Author, CreateAuthorError>>>,
}

impl BlogService for MockBlogService {
    async fn create_author(&self, _: &CreateAuthorRequest) -> Result<Author, CreateAuthorError> {
        let mut guard = self.create_author_result.lock();
        let mut result = Err(CreateAuthorError::Unknown(anyhow!("substitute error")));
        mem::swap(guard.as_deref_mut().unwrap(), &mut result);
        result
    }
}

#[tokio::test(flavor = "multi_thread")]
async fn test_create_author_success() {
    let author_name = AuthorName::new("Angus").unwrap();
    let author_email = EmailAddress::new("angus@howtocodeit.com").unwrap();
    let author_id = Uuid::new_v4();

    let service = MockBlogService {
        create_author_result: Arc::new(Mutex::new(Ok(Author::new(
            author_id,
            author_name.clone(),
            author_email.clone(),
        )))),
    };

    let state = axum::extract::State(AppState {
        author_service: Arc::new(service),
    });
    let body = axum::extract::Json(CreateAuthorHttpRequestBody {
        name: author_name.to_string(),
        email_address: author_email.to_string(),
    });

    let expected = ApiSuccess::new(
        StatusCode::CREATED,
        CreateAuthorResponseData { id: author_id.to_string() },
    );

    let actual = create_author(state, body).await;
    assert!(actual.is_ok(), "expected success, got {actual:?}");
    assert_eq!(actual.unwrap(), expected);
}
```

### Why `Arc<Mutex<Result>>` + `mem::swap`

- `Arc<Mutex<_>>` — the port requires `Clone`, but each test wants one configured response.
- `mem::swap` — `Result<_, CreateAuthorError>` isn't `Clone` when the error variant holds `anyhow::Error`. Swap moves the configured value out and leaves a placeholder.
- For ports called multiple times, replace the `Result` with `VecDeque<Result<...>>` or use `mockall`.

Note: the reference repo uses `std::sync::Mutex`. For async-heavy test setup `tokio::sync::Mutex` also works — pick one and stay consistent.

## Mocking driven ports (for service tests)

When testing `Service<R, M, N>`, mock `BlogRepository`, `BlogMetrics`, `AuthorNotifier` to verify orchestration (metrics on failure, notifier on success, etc.):

```rust
#[derive(Clone, Default)]
struct RecordingMetrics {
    success: Arc<AtomicUsize>,
    failure: Arc<AtomicUsize>,
}

impl BlogMetrics for RecordingMetrics {
    async fn record_author_creation_success(&self) {
        self.success.fetch_add(1, Ordering::SeqCst);
    }
    async fn record_author_creation_failure(&self) {
        self.failure.fetch_add(1, Ordering::SeqCst);
    }
}

#[tokio::test]
async fn service_records_success_and_notifies_on_ok() {
    let repo = MockRepo::with_ok(/* ... */);
    let metrics = RecordingMetrics::default();
    let notifier = RecordingNotifier::default();

    let service = Service::new(repo, metrics.clone(), notifier.clone());
    service.create_author(&req).await.unwrap();

    assert_eq!(metrics.success.load(Ordering::SeqCst), 1);
    assert_eq!(metrics.failure.load(Ordering::SeqCst), 0);
}
```

## Integration test for adapters

```rust
// tests/integration/sqlite_blog_repo.rs
#[tokio::test]
async fn sqlite_persists_and_rejects_duplicate() {
    let repo = Sqlite::new("sqlite::memory:").await.unwrap();
    sqlx::migrate!("./migrations").run(&repo.pool).await.unwrap();

    let req = CreateAuthorRequest::new(
        AuthorName::new("Angus").unwrap(),
        EmailAddress::new("a@b.com").unwrap(),
    );

    let first = repo.create_author(&req).await.unwrap();
    assert_eq!(first.name().to_string(), "Angus");

    let second = repo.create_author(&req).await;
    assert!(matches!(second, Err(CreateAuthorError::Duplicate { .. })));
}
```

## What NOT to mock

- Your database library (`sqlx`) — mock your own `BlogRepository` port instead.
- HTTP clients (`reqwest`) — wrap them behind a port trait and mock that.
- Anything with a type you didn't declare.
