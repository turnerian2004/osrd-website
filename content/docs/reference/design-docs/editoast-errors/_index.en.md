---
title: "Editoast error management"
linkTitle: "Editoast error management"
weight: 80
---

# Issues of the old system

- Mix between internal errors and API errors
- Errors are converted into `InternalError` early, which means that a caller of a function returning an `editoast::Result` will have some trouble `match`ing on the error returned
  - That means that it's troublesome to add context to an existing error, or to wrap it into another higher-level one.
- We can't track at compile-time which errors are returned by each function: that means that we don't know for sure which errors an endpoint can return (without careful manual investigation at least...)
- Consequently, we hardly can declare in the OpenApi file what errors an endpoint precisely returns, degrading the editoast API quality
- The frontend still requires editoast to declare all its errors though, to ensure they are translated properly. To achieve that we dynamically collect each `EditoastError` using the crate `inventory`. All the error descriptions collected are then transformed into OpenAPI schemas procedurally. On top of being a Rust antipattern (collecting state in proc-macros), this is complex to maintain on both editoast and frontend sides.
  - Not having each endpoint linked to the list of errors it can raise, also prevents the frontend easily handling errors properly.
- It's still unclear how we should expose errors from Core.

# Goals

- Have a clear separation between logically distinct errors.
- Dispose of a way to actually match on errors when they occur deeper in the stack
- Separate the error definition and their serialization.
- Establish how we want to forward Core's errors.
- Tie the errors to the endpoint they originate from in the OpenAPI.

# Constraints

- Keep the same error format (for backward compatibility reasons to avoid involving the frontend too much).
  - We must keep the `editoast:` prefix in the error type.
- The error must live until it is handled, conversion to our standard error format only happens in the response serialization.
- Errors must implement `std::error::Error`.
- Errors must be composable. This will typically be handled by `thiserror`'s `#[from]` attribute.
- Error variants must be shareable to ensure the deduplication of error kinds. For example, let's say we have two functions `get_infra(id: usize)` and `rename_infra(id: usize, name: String)`. Both functions error types have to include a variant describing the error case of an infrastructure not being found by its ID. However, we can't duplicate something like `InfraNotFound { id: usize }` in both error types as this leads to two different error paths describing the same error case. This is especially problematic for error translation keys. We need to be able to define an error `InfraNotFound` and include it in both error types.
  - A unicity check may be performed in the post-processing of the OpenAPI file to ensure that each error has a unique error type.
- Each endpoint must provide all its error cases in the OpenAPI. (How the frontend will consume them is another problem that we'll have to deal with.)
- As for OSRD errors, the `context` field of the error is populated in the views.
- Errors can be handled in a generic manner (for situations where it makes some sense to do so).
  - I.e.: some form of downcasting is available.

# New system

- Rely on `thiserror` everywhere.
- Keep the `trait EditoastError` but only implement it for errors defined in `views`.
  - Since it is only used in `views` now, let's rename it to `trait ViewError`.
- Create a proc-macro `derive(ViewError)` which interfaces with `derive(thiserror::Error)`.
- The `context` is empty by default but can be provided by the `impl ViewError`. The macro is also able to take context providers.
- `ViewError`'s `#[source]`, `#[from]`, `source` and `backtrace` fields are never serialized, unless explicitly provided. This shouldn't be the case as it exposes editoast internals at the API level.
- `impl<T: ViewError> utoipa::IntoResponses for T` (may be generated or inferred)
- Errors **should not** implement `Serialize` except for view errors (`derive(ViewError)`) which generates an `impl ViewError` used to serialize in the HTTP response.
- Error cases that will be used repeatedly are defined as a `struct` but still `derive(thiserror::Error)`.
- The `error_type` of each variant is generated by the macro at the format `ErrorTypeName::VariantName`, but can be provided explicitly if conflicts arise.
  - `editoast:` will be prepended systematically to indicate the service that raised the error.
  - Since this type is not guaranteed to be unique, we may implement a post-processing step to ensure errors with the same `error_type` have the same OpenAPI schema.
  - To ease the debugging process, an optional `source_location` will be provided in `ViewError`s containing a link to the GitHub file and line where the error is defined.

## Nominal case

```rust
// in mod views;

fn get_resource(key: Key) -> Result<Resource, GetError> {
    todo!()
}

#[derive(Debug, thiserror::Error)]
pub enum GetError {
    #[error("ID not found")]
    IdNotFound { id: u64 },
    #[error("name not found")]
    NameNotFound { name: String }
}

fn process_resource(resource: Resource) -> Result<Computation, ProcessingError> {
    todo!()
}

#[derive(Debug, thiserror::Error)]
pub enum ProcessingError {
    #[error("Resource is invalid")]
    InvalidResource { resource: Resource },
    #[error("Resource is too old")]
    OutdatedResource { resource: Resource }
}

#[utoipa::path(
    ...,
    responses(
        (status = 200, body = Computation),
        EndpointError, // impl utoipa::IntoResponse
    )
)]
async fn endpoint(Path(key): Path<Key>) -> Result<Json<Computation>, EndpointError> {
    let resource = get_resource(key)?;
    let value = process_resource(resource)?;
    Ok(Json(value))
}

#[derive(Debug, thiserror::Error, ViewError)]
pub enum EndpointError {
    #[error("Resource not found")]
    #[view_error(user)] // <=> status = 400
    ResourceNotFound(#[from] GetError),

    #[view_error(internal)] // <=> status = 500, default
    ProcessingFailed(#[from] ProcessingError)
}
```

## Share errors between crates

Since we require no other constraint that `impl std::error::Error` for composition, it's easy to nest errors using `thiserror`.

````rust
// in editoast_models

#[derive(Debug, thiserror::Error)]
#[error("postgres error: {0}")]
struct DbError(#[from] diesel::Error);

// in editoast_valkey (if we actually had that crate 👀)

#[derive(Debug, thiserror::Error)]
#[error("valkey error: {0}")]
struct ValkeyError(#[from] redis::RedisError);

// in editoast_views, where diesel isn't available

#[derive(Debug, thiserror::Error)]
#[error("invalid resource form: {0}")]
struct FormError(ResourceForm);

#[derive(Debug, thiserror::Error, ViewError)]
#[view_error(name = "CreateResourceError")] // schema name & error_type
enum CreateError {
    #[error("will be overridden, but still useful for development")]
    #[view_error(internal, name = "Database")]
    Db(#[from] DbError),

    #[error(transparent)] // shown to the user
    #[view_error(user)]
    InvalidForm(#[from] FormError)
}

async fn create(...) -> Result<Json<Resource>, CreateError> {
    todo!()
}

#[derive(Debug, thiserror::Error, ViewError)]
enum UpdateError {
    #[view_error(internal)]
    Db(#[from] DbError),

    #[view_error(internal)]
    Valkey(#[from] ValkeyError),

    #[error(transparent)]
    #[view_error(user)]
    InvalidForm(#[from] FormError)
}

async fn update(...) -> Result<(), UpdateError> {
    todo!()
}
````

## Ease composability

### Nesting `ViewError`s

We composed errors by nesting them thanks to `thiserror`. However, to compose and reuse `EditoastErrors`, we need a special flag so that when we attempt to serialize the error, we return the serialization of the source error directly.

```rust
#[derive(Debug, thiserror::Error, ViewError)]
#[error("no such infra: {id}")]
#[view_error(context, status = NOT_FOUND)] // accepts http::StatusCode associated constants
struct InfraNotFound { id: u64 }

#[derive(Debug, thiserror::Error, ViewError)]
#[error("unauthorized")]
#[view_error(status = 401)]
struct Unauthorized;

#[derive(Debug, thiserror::Error, ViewError)]
enum EndpointError {
    NotFound(#[from] #[view_error] InfraNotFound),

    Unauthorized(#[from] #[view_error] Unauthorized)

    #[error("oh no")]
    #[view_error(user)]
    Error1,
}
```

## Full `derive(ViewError)` spec

The macro supports enums, named structs and tuple structs (fixme correct naming).

````rust
#[derive(thiserror::Error)] // not an EditoastError
#[error("wrong string: {0}")]
struct WrongString(String);

// error_type = "InvalidInt"
// context = { value: number }
#[derive(ViewError, thiserror::Error)]
#[error("wrong int: {value}")]
#[view_error(user, name = "InvalidInt")]
struct WrongInt { value: i64 };

#[derive(ViewError, thiserror::Error)]
#[error("my error type")]
#[view_error(context)]
enum MyError {
    // error_type = "MyError::InvalidString"
    // context = { expected_format: string }
    // #[view_error(internal)] by default
    InvalidString { #[from] source: WrongString, expected_format: String },

    // error_type = "MyError::InvalidInt"
    // context = { value: number } (xyz is skipped as we forward the ViewError)
    WrongInt { #[from] #[view_error] source: WrongInt, xyz: String }

    // error_type = "MyError::Bad"
    // context = { "0": string, "1": number }
    #[error("user did a bad with {0} and {1}")]
    #[view_error(user, name = "Bad")]
    Oops(String, i64)
}
````

### Providing `context`

Context is computed just before the error is serialized in `axum`'s error handler.

Note: it shouldn't be used in `editoast` as we now have enums variants we can `match` on. The `context` response field is meant to provide data potentially useful to the frontend so that it may perform some kind of error recovery.

`derive(ViewError)` provides a few ways to set it.

````rust
#[derive(Debug, thiserror::Error, ViewError)]
enum Error {
    #[view_error(user)]
    NoContext { because: String }, // context = { }

    #[view_error(user, context)]
    AllFieldsIntoContext { reasons: Vec<String> }, // context = { "reasons": [string] }

    // select (and maybe rename) some fields to include to the context
    #[view_error(user, context(reason, recovery_id = "recovery"))]
    SomeFieldsIntoContext {
        reason: String,
        recovery_id: String,
        not_serializable: mpsc::Sender<()>,
        not_wanted: u64,
    }, // context = { "reason": string, "recovery": string }
}

// with a provider function
#[derive(Debug, thiserror::Error, ViewError)]
#[view_error(context_with = context_provider)]
enum Error {
    Variant1(String),
    Variant2(String, u64)
}

fn context_provider(error: Error) -> HashMap<String, serde_json::Value> {
    todo!()
}
````

### About Core errors

The Core service is a bit special as it already returns errors with the common OSRD format. Since editoast doesn't really need to parse and recover from Core errors[^2], we don't need an exhaustive list of them. We still need to differentiate them from other editoast errors (let's not start tossing `InternalError` around again…) and to provide a key for the frontend to translate them.

[^2]: Core errors will likely never be recoverable from either editoast or the frontend. For the latter, such errors are likely to be displayed as a generic "Internal error" message. So no translation is needed. For these reasons, we don't need to pass them in the OpenAPI. However, if in the future, we want editoast to actually parse Core errors, ensuring a proper mapping will still be possible.

Core errors are then "lightly" wrapped: we keep the error as a generic `serde_json::Value` that we include into a `struct CoreError` that we can augment with additional information about the request. This way, the original is preserved, forwarded to the frontend, but fits our new error paradigm.

`CoreError` draft:

```rust
// in editoast_core

#[derive(Debug, thiserror::Error)]
struct CoreError {
    /// RabbitMQ "endpoint"
    rpc: String,
    /// Request metadata
    metadata: HashMap<String, String>,
    /// The original error
    error: serde_json::Value,
}
```

Note: the `error` field is kept as a `serde_json::Value` and not parsed (even though its format is standard) as we're not supposed to perform any kind of analysis or recovery on it. If we end up parsing it in the future, that means we need a stronger mapping between Core errors and what editoast expects. The red flag will be more obvious if we end up manipulating a JSON dict instead of a proper structure.

### Why do we need a derive macro?

The main issue with our error system is that the types we manipulate do not serialize to the error format we want. For example, an error defined like so:

```rust
#[derive(Debug, thiserror::Error)]
#[error("{cause}")]
struct MyError { cause: String, fix: String }
```

shouldn't be serialized as:

```json
{
  "cause": "Emperor Zurg",
  "fix": "Buzz Lightyear"
}
```

like `serde::Serialize` would do, but as:

```json
{
  "error_type": "editoast:MyError",
  "status": 500,
  "message": "Emperor Zurg",
  "context": {
    "cause": "Emperor Zurg",
    "fix": "Buzz Lightyear",
  }
}
```

making `derive(serde::Serialize)` basically useless for our errors. On top of that, since by design the derive macros of `utoipa` (`ToSchema`, `IntoResponse` especially) interpret the type structure like `derive(serde::Serialize)` would do, we can't rely on them either. Therefore we need a custom derive macro to convey the structural information of the type at runtime, while still allowing a custom `Serialize` and `IntoResponse` implementations.

Another solution would be to shift our error definition paradigm and orient ourselves to a system without code generation (probably using a combination of traits and builders). This would imply to rewrite all our errors and their handling, which is costly 🤑🫰. We'd also have to get rid of the convenience of `thiserror`, a huge loss in terms of ergonomics. And that would break the consistency with the other sub-crates of editoast.

The macro doesn't even have to be overly complex. The `trait ViewError` could be responsible of translating the static type definition into an associated constant, which would be used to compute data produced at runtime. (Ie. `impl axum::IntoResponse for T: ViewError` and `impl utoipa::IntoResponses for T: ViewError`.) This would reduce the amount of generated code, at the expense of more complex data manipulation at runtime.

Going this deep into the implementation is not the goal of this document: the best way to do things will be decided when the migration work will start.

## Implementation plan

We'll need a progressive migration as this implies too much change to fit in a single PR. `EditoastError` and `ViewError` will have to cohabit for some time.

### Setup

1. Create the `trait views::ViewError`
2. Implement an `axum::IntoResponse` for `ViewError` to generate a standard OSRD error response payload
3. Add a post-processing step to the OpenAPI generation to ensure the consistency of error status codes. More details below.
4. Create a derive macro `ViewError` that interfaces with `thiserror::Error` API and generates *at least* `impl ViewError`
5. The macro may generate an `impl utoipa::IntoResponses` that tells `utoipa` what to expect in the response payloads. This trait may be auto-implemented for each `ViewError` type (we'll see how things go in the implementation).
6. We'll have to change the frontend error keys collection script almost entirely by the end of this migration. We could update it to also look for errors in the OpenAPI routes response section but that's extra work which brings little benefits. We accept a temporary desync of the error keys while this migration is ongoing.


### Migration

The easier way to proceed here would be, to start by converting simple errors that occur deep in the stack (such as Postgres errors, Valkey errors, Core errors, etc.). This way, we can rely on the Rust compiler to guide us through the process and ensure we don't forget any error. We'll need some kind of adapters to incorporate these errors into `EditoastError`s. We may find a generic way to do that, but that's more an implementation detail, especially since that would be temporary.

A good starting place would be `editoast_search`[^1] because its internal errors do not implement `EditoastError` already. Valkey errors may also be a decent candidate.

One large change that will have to be atomic will be the adaptation of `Model`'s errors[^3].

[^1]: Provided we start this migration before the rewrite of the search engine.
[^3]: This work has already started at the time of writing.

### Wrapping up things

Eventually, when all errors are converted and views errors are attached to their endpoint(s) in the OpenAPI, we'll have to:

1. Remove `trait EditoastError`, `derive(EditoastError)` and `struct InternalError` (at least its former version as the name may be reused in a different scope)
2. Adapt the frontend error keys collection script to look for errors in the OpenAPI routes response section instead of `components/schemas`
3. (Out of scope) Discuss with the frontend the level of visibility about internal errors we want to give the user

## Rejected ideas

### Anonymous enums

{{% pageinfo color="warning" %}}
Rejected because it wouldn't be trivial to implement the multiple `From<T, U, ..> for EnumX<T, U, ..>` without negative type parameters or the fallback type (unstable). That's necessary to make the `EnumX` usage transparent and avoid using `T1`, ..., `Tn` variants explicitly. The crate `anon_enum` deals with this issue by not providing the feature, so it won't help either.
{{% /pageinfo %}}

Since we now have to really manage errors happening in every function **as precisely as possible**, there will likely be a lot of error enums going around. This may be a hassle and wrongfully encourage returning `Option` as an error. To circumvent that, an easy (albeit opinionated) QoL feature would be to use anonymous enums.

````rust
fn get_resource(id: u64) -> Result<Resource, Enum3<DbError, ValkeyError, MissingResourceError>> {
    todo!()
}

fn process_resource(resource: Resource) -> Result<Computation, Enum2<DbError, ProcessingError> {
    todo!()
}

#[utoipa::path(
    ...,
    responses(
        (status = 200, body = Computation),
        (status = 400, body = EndpointError)
    )
)]
async fn endpoint(Path(key): Path<Key>) -> Result<Json<Computation>, EndpointError> {
    let resource = get_resource(key)?;
    let value = process_resource(resource)?;
    Ok(Json(value))
}

#[derive(Debug, thiserror::Error, ViewError)]
pub enum EndpointError {
    Db(#[from] DbError),

    Valkey(#[from] ValkeyError),

    #[error(transparent)]
    #[view_error(user)]
    Missing(#[from] MissingResourceError)

    #[error(transparent)]
    #[view_error(user)]
    Processing(#[from] ProcessingError)
}
````

The implementation of the `EnumX` type would be rather easy to generate for many tuple sizes. The crate [anon_enum](https://docs.rs/anon_enum/1.1.0/anon_enum/) exists but if we choose to use this pattern, it's probably better to have our own type for greater flexibility and avoid another dependency.


### Incident reports

{{% pageinfo color="warning" %}}
Rejected because it would be another mechanism to maintain with little benefits: errors are already persisted using opentelemetry, and for "internal server errors", it's up to the frontend to choose how much details is shown to the user.
{{% /pageinfo %}}

For `internal` errors that won't contain meaningful information for the end user, we substitute the error by:

```json
{
  "error_type": "InternalError",
  "message": "<a meaningful message>",
  "status": 500,
  "context": {},
  "incident": "<uuid>"
}
```

In order to be able to find and investigate the error later on, we associate to each `5xx` error a unique `incident` identifier. At first we'll just log the incident with:

- the error message
- the error `Debug` representation
- the backtrace(s), if any

The log entry will be persisted in datadog/jaeger so it's probably good enough at first.

It's useful in development to have the real error shown in the interface instead of just "Internal error". We can set an environment variable `OSRD_DEV=1` to avoid replacing the error in the `axum` handler.
