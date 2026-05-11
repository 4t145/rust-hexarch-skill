# HTTP Handlers

Every handler follows the same three-step shape. If a handler deviates, it's wrong.

## File organization

One operation per file: `src/lib/inbound/http/handlers/create_author.rs`. The file contains:

- The handler function
- The HTTP request body type (with `try_into_domain()`)
- The HTTP response data type (with `From<&Entity>`)
- Operation-specific parse error (`Parse{Action}{Entity}HttpRequestError`) aggregating newtype errors
- Tests

Shared types (`ApiSuccess`, `ApiError`, `ApiResponseBody`) typically live alongside one handler or in a `responses.rs` module — the canonical repo puts them in `create_author.rs` since that's the only handler.

## The three-step shape

```
(transport input) → try_into_domain() → (domain input)
                                            ↓
                                    service.method().await
                                            ↓
(transport output) ← ApiSuccess::new ← From<&Entity> ← (domain output)
                     ApiError::from  ← From<DomainError>
```

## Full handler

```rust
// src/lib/inbound/http/handlers/create_author.rs
use axum::extract::State;
use axum::http::StatusCode;
use axum::Json;
use axum::response::{IntoResponse, Response};
use serde::{Deserialize, Serialize};
use thiserror::Error;

use crate::domain::blog::models::author::{
    Author, AuthorName, AuthorNameEmptyError, CreateAuthorRequest, EmailAddress, EmailAddressError,
};
use crate::domain::blog::models::author::CreateAuthorError;
use crate::domain::blog::ports::BlogService;
use crate::inbound::http::AppState;

/// HTTP request body.
#[derive(Debug, Clone, PartialEq, Eq, Deserialize)]
pub struct CreateAuthorHttpRequestBody {
    name: String,
    email_address: String,
}

/// Validation error aggregating newtype construction failures.
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

#[derive(Debug, Clone, PartialEq, Eq, Serialize)]
pub struct CreateAuthorResponseData {
    id: String,
}

impl From<&Author> for CreateAuthorResponseData {
    fn from(author: &Author) -> Self {
        Self { id: author.id().to_string() }
    }
}

/// Create a new [Author].
///
/// # Responses
/// - 201 Created
/// - 422 Unprocessable Entity: duplicate or invalid input
pub async fn create_author<BS: BlogService>(
    State(state): State<AppState<BS>>,
    Json(body): Json<CreateAuthorHttpRequestBody>,
) -> Result<ApiSuccess<CreateAuthorResponseData>, ApiError> {
    let domain_req = body.try_into_domain()?;
    state
        .author_service
        .create_author(&domain_req)
        .await
        .map_err(ApiError::from)
        .map(|ref author| ApiSuccess::new(StatusCode::CREATED, author.into()))
}
```

## `ApiSuccess` and `ApiResponseBody`

```rust
#[derive(Debug, Clone)]
pub struct ApiSuccess<T: Serialize + PartialEq>(StatusCode, Json<ApiResponseBody<T>>);

impl<T: Serialize + PartialEq> PartialEq for ApiSuccess<T> {
    fn eq(&self, other: &Self) -> bool {
        self.0 == other.0 && self.1.0 == other.1.0
    }
}

impl<T: Serialize + PartialEq> ApiSuccess<T> {
    pub fn new(status: StatusCode, data: T) -> Self {
        Self(status, Json(ApiResponseBody::new(status, data)))
    }
}

impl<T: Serialize + PartialEq> IntoResponse for ApiSuccess<T> {
    fn into_response(self) -> Response {
        (self.0, self.1).into_response()
    }
}

#[derive(Debug, Clone, PartialEq, Eq, Serialize)]
pub struct ApiResponseBody<T: Serialize + PartialEq> {
    status_code: u16,
    data: T,
}

impl<T: Serialize + PartialEq> ApiResponseBody<T> {
    pub fn new(status_code: StatusCode, data: T) -> Self {
        Self { status_code: status_code.as_u16(), data }
    }
}

impl ApiResponseBody<ApiErrorData> {
    pub fn new_error(status_code: StatusCode, message: String) -> Self {
        Self { status_code: status_code.as_u16(), data: ApiErrorData { message } }
    }
}

#[derive(Debug, Clone, PartialEq, Eq, Serialize)]
pub struct ApiErrorData {
    pub message: String,
}
```

## `ApiError` with `IntoResponse` and `From` impls

```rust
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum ApiError {
    InternalServerError(String),
    UnprocessableEntity(String),
}

impl From<anyhow::Error> for ApiError {
    fn from(e: anyhow::Error) -> Self {
        Self::InternalServerError(e.to_string())
    }
}

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

impl From<ParseCreateAuthorHttpRequestError> for ApiError {
    fn from(e: ParseCreateAuthorHttpRequestError) -> Self {
        let message = match e {
            ParseCreateAuthorHttpRequestError::Name(_) => "name cannot be empty".into(),
            ParseCreateAuthorHttpRequestError::EmailAddress(cause) => {
                format!("email address {} is invalid", cause.invalid_email)
            }
        };
        Self::UnprocessableEntity(message)
    }
}

impl IntoResponse for ApiError {
    fn into_response(self) -> Response {
        match self {
            ApiError::InternalServerError(e) => {
                tracing::error!("{}", e);
                (
                    StatusCode::INTERNAL_SERVER_ERROR,
                    Json(ApiResponseBody::new_error(
                        StatusCode::INTERNAL_SERVER_ERROR,
                        "Internal server error".to_string(),
                    )),
                ).into_response()
            }
            ApiError::UnprocessableEntity(message) => (
                StatusCode::UNPROCESSABLE_ENTITY,
                Json(ApiResponseBody::new_error(StatusCode::UNPROCESSABLE_ENTITY, message)),
            ).into_response(),
        }
    }
}
```

## `AppState` and `HttpServer`

```rust
// src/lib/inbound/http.rs
use std::sync::Arc;

use anyhow::Context;
use axum::Router;
use axum::routing::post;
use tokio::net;

use crate::domain::blog::ports::BlogService;
use crate::inbound::http::handlers::create_author::create_author;

mod handlers;

#[derive(Debug, Clone, PartialEq, Eq)]
pub struct HttpServerConfig<'a> {
    pub port: &'a str,
}

#[derive(Debug, Clone)]
pub(crate) struct AppState<BS: BlogService> {
    pub author_service: Arc<BS>,
}

pub struct HttpServer {
    router: Router,
    listener: net::TcpListener,
}

impl HttpServer {
    pub async fn new(
        blog_service: impl BlogService,
        config: HttpServerConfig<'_>,
    ) -> anyhow::Result<Self> {
        let trace_layer = tower_http::trace::TraceLayer::new_for_http().make_span_with(
            |request: &axum::extract::Request<_>| {
                let uri = request.uri().to_string();
                tracing::info_span!("http_request", method = ?request.method(), uri)
            },
        );

        let state = AppState { author_service: Arc::new(blog_service) };

        let router = Router::new()
            .nest("/api", api_routes())
            .layer(trace_layer)
            .with_state(state);

        let listener = net::TcpListener::bind(format!("0.0.0.0:{}", config.port))
            .await
            .with_context(|| format!("failed to listen on {}", config.port))?;

        Ok(Self { router, listener })
    }

    pub async fn run(self) -> anyhow::Result<()> {
        tracing::debug!("listening on {}", self.listener.local_addr().unwrap());
        axum::serve(self.listener, self.router)
            .await
            .context("server error")?;
        Ok(())
    }
}

fn api_routes<BS: BlogService>() -> Router<AppState<BS>> {
    Router::new().route("/authors", post(create_author::<BS>))
}
```

## What handlers must NOT do

- Call `sqlx` or any database crate directly
- Build domain types by hand without going through a validated constructor
- Return raw `CreateAuthorError` (leaks domain variant shapes via `Debug`)
- Contain more than ~10 lines of logic — logic belongs in the service
