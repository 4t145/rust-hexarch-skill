# Error Handling

Three categories of error types live in the codebase:

1. **Newtype construction errors** — `AuthorNameEmptyError`, `EmailAddressError`
2. **Domain operation errors** — `CreateAuthorError` (what a domain method can fail with)
3. **Transport errors** — `ApiError` (what HTTP returns to clients)

They are never the same type. They are connected by `From` impls.

## Domain operation error pattern

```rust
// src/lib/domain/blog/models/author.rs
use thiserror::Error;

#[derive(Debug, Error)]
pub enum CreateAuthorError {
    #[error("author with name {name} already exists")]
    Duplicate { name: AuthorName },

    #[error(transparent)]
    Unknown(#[from] anyhow::Error),
    // to be extended as new error scenarios are introduced
}
```

### Why `Unknown(#[from] anyhow::Error)` is mandatory

Every real operation has failure modes you didn't model: connection drops, OS errors, upstream bugs. Without a catch-all, adapters would either enumerate every possible cause (impossible) or silently swallow errors (unsafe).

With `Unknown(#[from] anyhow::Error)`, adapters propagate any `anyhow::Error` via `?`:

```rust
let tx = self.pool.begin().await.context("failed to start tx")?;
// if this fails, it becomes CreateAuthorError::Unknown automatically
```

### Why NOT `#[non_exhaustive]` — in application code

The service, handler, and tests all need to match on every variant. `#[non_exhaustive]` forces a wildcard arm that silently absorbs new variants and skips required handling. For **application code** where you control every match site, keep the enum exhaustive so the compiler flags un-updated matches.

**Library exception:** if you're publishing the domain as a crate others consume, the reverse is true — the article: *"If you were writing a library, this wouldn't be true. You'd have to use `non_exhaustive`, forcing library users to include a catch-all case… otherwise, any change to the number or structure of enum variants would be breaking, and require a major version bump."*

### Derives

- `Debug` — always
- `Error` (thiserror) — always
- `Clone`, `PartialEq`, `Eq` — often omitted when a variant holds `anyhow::Error` (which is none of those). In tests, use `matches!` for variant-shape assertions.

## Transport error pattern

```rust
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum ApiError {
    InternalServerError(String),
    UnprocessableEntity(String),
}
```

Plus `IntoResponse` and `From<_>` impls — see `reference/handlers.md`.

## Newtype error pattern

Small, single-reason errors:

```rust
#[derive(Clone, Debug, Error)]
#[error("author name cannot be empty")]
pub struct AuthorNameEmptyError;

#[derive(Clone, Debug, Error)]
#[error("{invalid_email} is not a valid email address")]
pub struct EmailAddressError {
    pub invalid_email: String,
}
```

## Aggregating newtype errors at the HTTP boundary

When a transport type calls multiple newtype constructors, wrap their errors in a handler-local enum:

```rust
#[derive(Debug, Clone, Error)]
enum ParseCreateAuthorHttpRequestError {
    #[error(transparent)]
    Name(#[from] AuthorNameEmptyError),
    #[error(transparent)]
    EmailAddress(#[from] EmailAddressError),
}

impl CreateAuthorHttpRequestBody {
    fn try_into_domain(self) -> Result<CreateAuthorRequest, ParseCreateAuthorHttpRequestError> {
        let name = AuthorName::new(&self.name)?;
        let email = EmailAddress::new(&self.email_address)?;
        Ok(CreateAuthorRequest::new(name, email))
    }
}
```

`#[from]` makes each newtype error coerce via `?` automatically.

## Domain → Transport mapping rules

```rust
impl From<CreateAuthorError> for ApiError {
    fn from(e: CreateAuthorError) -> Self {
        match e {
            CreateAuthorError::Duplicate { name } => {
                Self::UnprocessableEntity(format!("author with name {name} already exists"))
            }
            CreateAuthorError::Unknown(cause) => {
                tracing::error!("{:?}\n{}", cause, cause.backtrace());
                Self::InternalServerError("Internal server error".to_string())
            }
        }
    }
}
```

1. **Every `Unknown` becomes a generic 500.** No matter how tempting, don't include `cause.to_string()` in the response.
2. **Log `cause` with its full chain.** Use `?cause` or `cause.backtrace()` — debugging data lives here.
3. **Map business errors to specific 4xx.** `Duplicate` → 422. `NotFound` → 404. Validation → 422.
4. **Parse errors also become 422.**

## `From<anyhow::Error> for ApiError`

The reference repo also implements this:

```rust
impl From<anyhow::Error> for ApiError {
    fn from(e: anyhow::Error) -> Self {
        Self::InternalServerError(e.to_string())
    }
}
```

This is for the rare case where an anyhow error reaches the handler directly (not wrapped in a domain error). Still maps to 500, still logs via `IntoResponse`.
