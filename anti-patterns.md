# Anti-Patterns

Things that look reasonable but violate the architecture. With Before/After.

## 1. Naming ports after entities instead of bounded contexts

### Before (wrong)

```rust
pub trait AuthorService: Clone + Send + Sync + 'static { ... }
pub trait AuthorRepository: Clone + Send + Sync + 'static { ... }
pub trait PostService: Clone + Send + Sync + 'static { ... }
pub trait PostRepository: Clone + Send + Sync + 'static { ... }
```

Now adding `Post` doubled the ports. Cross-entity transactions (create a post and update its author's count atomically) need a third trait.

### After (right)

```rust
// One service and one repository for the bounded context.
pub trait BlogService: Clone + Send + Sync + 'static {
    fn create_author(&self, req: &CreateAuthorRequest) -> ...;
    fn create_post(&self, req: &CreatePostRequest) -> ...;
}

pub trait BlogRepository: Clone + Send + Sync + 'static {
    fn create_author(&self, req: &CreateAuthorRequest) -> ...;
    fn create_post(&self, req: &CreatePostRequest) -> ...;
}
```

Exception: entity-specific notifiers (`AuthorNotifier`) are fine — they're genuinely scoped to one entity.

---

## 2. Domain type deriving `Deserialize`

### Before (wrong)

```rust
// src/lib/domain/blog/models/author.rs
use serde::Deserialize;

#[derive(Debug, Deserialize)]
pub struct Author {
    pub id: Uuid,
    pub name: String,   // unvalidated! could be ""
}
```

Domain depends on `serde`, validation is bypassed at deserialization, `name` is a raw `String`.

### After (right)

```rust
// src/lib/domain/blog/models/author.rs
// NO serde import
pub struct Author {
    id: Uuid,
    name: AuthorName,   // validated newtype, private field
    email: EmailAddress,
}

// src/lib/inbound/http/handlers/create_author.rs
#[derive(Deserialize)]
pub struct CreateAuthorHttpRequestBody {
    name: String,          // raw, validated in try_into_domain
    email_address: String,
}
```

---

## 3. Handler calling `sqlx` directly

### Before (wrong)

```rust
pub async fn create_author(
    State(pool): State<SqlitePool>,
    Json(body): Json<CreateAuthorHttpRequestBody>,
) -> Result<Json<Author>, StatusCode> {
    let id = Uuid::new_v4();
    sqlx::query!("INSERT INTO authors VALUES (?, ?)", id, body.name)
        .execute(&pool)
        .await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;
    Ok(Json(Author { id, name: body.name }))
}
```

Impossible to test without a real DB, errors flattened to bare status codes, no validation, domain logic in the HTTP layer.

### After (right)

Handler depends on `BlogService` trait; see `reference/handlers.md`.

---

## 4. Single `AppError` covering every layer

### Before (wrong)

```rust
#[derive(Error, Debug)]
pub enum AppError {
    #[error(transparent)]
    Sqlx(#[from] sqlx::Error),
    #[error(transparent)]
    Reqwest(#[from] reqwest::Error),
    #[error("not found")]
    NotFound,
    // ... 30 more variants
}
```

Leaks `sqlx::Error::Database(...)` contents straight to HTTP responses.

### After (right)

- `CreateAuthorError` in domain — business-level variants only
- `anyhow::Error` inside adapters, converted via `#[from]`
- `ApiError` at transport boundary — generic `InternalServerError` for unknowns

Three types, three layers, explicit conversions.

---

## 5. Fat trait bundling unrelated capabilities

### Before (wrong)

```rust
pub trait BlogRepository: Send + Sync + 'static {
    async fn create_author(&self, ...) -> ...;
    async fn record_metric(&self, ...) -> ...;    // not persistence
    async fn send_email(&self, ...) -> ...;       // not persistence
}
```

Sqlite adapter has to stub metrics/email. Metrics adapter has to stub SQL. One change affects every impl.

### After (right)

Three separate traits — `BlogRepository`, `BlogMetrics`, `AuthorNotifier` — wired as three generics on `Service<R, M, N>`.

---

## 6. Using `#[non_exhaustive]` on domain errors

### Before (wrong)

```rust
#[derive(Error, Debug)]
#[non_exhaustive]
pub enum CreateAuthorError {
    Duplicate { name: AuthorName },
    Unknown(#[from] anyhow::Error),
}
```

Every match site now needs a wildcard arm, which silently catches future variants — including ones needing specific HTTP status codes.

### After (right)

Plain enum, no `#[non_exhaustive]`. Let the compiler flag un-updated match sites.

---

## 7. `main.rs` containing business logic

### Before (wrong)

```rust
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let pool = SqlitePool::connect(...).await?;
    let app = Router::new()
        .route("/authors", post(|Json(body): Json<CreateAuthorHttpRequestBody>| async move {
            let id = Uuid::new_v4();
            sqlx::query!(...).execute(&pool).await.unwrap();
            Json(AuthorResponse { id })
        }));
    axum::serve(listener, app).await?;
    Ok(())
}
```

Un-testable dead zone: closure logic can't be unit-tested.

### After (right)

See `templates/main.rs.tmpl`. `main` only constructs `Config`, adapters, service, `HttpServer`, then calls `run()`.

---

## 8. Reusing the entity as the request model

### Before (wrong)

```rust
pub struct Author {
    pub id: Option<Uuid>,    // Option because new authors don't have one
    pub name: AuthorName,
    pub created_at: Option<DateTime<Utc>>,
}
```

Every field is `Option`. Every access needs unwrapping. Invariants become runtime asserts.

### After (right)

```rust
pub struct CreateAuthorRequest {
    name: AuthorName,
    email: EmailAddress,
}

pub struct Author {
    id: Uuid,
    name: AuthorName,
    email: EmailAddress,
}
```

Two types, zero `Option` for always-present fields.

---

## 9. Public fields on newtypes

### Before (wrong)

```rust
pub struct AuthorName(pub String);
```

Anyone can write `AuthorName("".to_string())`, bypassing `AuthorName::new`. Validation is cosmetic.

### After (right)

```rust
pub struct AuthorName(String);

impl AuthorName {
    pub fn new(raw: &str) -> Result<Self, AuthorNameEmptyError> { /* ... */ }
}

impl Display for AuthorName {
    fn fmt(&self, f: &mut Formatter<'_>) -> std::fmt::Result { f.write_str(&self.0) }
}
```

Private field, fallible constructor, `Display` for read access.

---

## 10. Mocking `sqlx` instead of the port trait

### Before (wrong)

Using `mockall` to mock `SqlitePool` — library-specific, brittle.

### After (right)

```rust
impl BlogService for MockBlogService { /* ... */ }     // for handler tests
impl BlogRepository for MockBlogRepo { /* ... */ }      // for service tests
```

Mocks belong at port boundaries you own.

---

## 11. Mocking at the wrong boundary for handler tests

### Before (wrong)

Handler tests mock `BlogRepository`, then construct a real `Service` around it:

```rust
let repo = MockBlogRepository::new(...);
let service = Service::new(repo, RealMetrics, RealNotifier);
// test handler
```

Now a bug in `Service::create_author` orchestration affects handler tests. Conflates two layers.

### After (right)

Handler tests mock `BlogService` directly — the port the handler actually depends on:

```rust
let service = MockBlogService { /* canned response */ };
// test handler
```

Service orchestration gets tested in `service.rs`'s own tests with mocks of `BlogRepository`/`BlogMetrics`/`AuthorNotifier`.

---

## 12. Inbound adapter importing outbound adapter

### Before (wrong)

```rust
// src/lib/inbound/http/handlers/create_author.rs
use crate::outbound::sqlite::Sqlite;

pub async fn create_author(State(sqlite): State<Sqlite>, ...) { ... }
```

Handler depends on concrete adapter. Can't swap to Postgres without touching it.

### After (right)

Handler depends on a trait from `domain/`. Both inbound and outbound depend on the domain, never on each other.
