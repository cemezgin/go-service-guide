---
name: patterns
description: Project patterns including functional options, must pattern, caching decorator, error handling, configuration, and HTTP clients. Use when implementing common patterns in the codebase.
---

# Project Patterns

**References:** [Examples](examples.md)

## Functional Options

> [Example](examples.md#functional-options)

Use `With*` functions for configurable constructors:

```go
func NewClient(baseURL string, options ...ClientOption) *Client {
    opts := defaultOptions()
    for _, option := range options {
        option(opts)
    }
    // create client...
}

client := NewClient(url, WithTimeout(30*time.Second), WithRetryCount(3))
```

## Must Pattern

> [Example](examples.md#must-pattern)

**Allowed only in `cmd/`, `internal/apps/`** - to fail fast on misconfiguration.

```go
func Must[T any](f func() (T, error)) T {
    v, err := f()
    if err != nil { panic(err) }
    return v
}
```

## Caching Decorator

> [Example](examples.md#caching-decorator)

Wrap clients with cache layer for transparent caching.

## Error Handling

> [Example](examples.md#error-handling)

| Rule | Example |
|------|---------|
| Wrap with context | `fmt.Errorf("get order %s: %w", id, err)` |
| Check with Is/As | `errors.Is(err, ErrNotFound)` |
| Handle once | Don't log AND return |
| Custom errors | `var ErrFoo = errors.New()` or `type FooError struct{}` |

## Configuration

> [Example](examples.md#configuration)

| Type | Path |
|------|------|
| App config | `config/appconfig/{env}.json` |
| Infra config | `config/appconfig/infra-{env}.json` |
| SSM | `/myservice/{env}/app/config` |

**Rules:** No ARNs in app config, no secrets in files, use `time.Duration` with custom unmarshaler

## HTTP Client with Retry

> [Example](examples.md#http-client)

```go
client := tracing.NewRestyClient(baseURL,
    tracing.WithTimeout(30*time.Second),
    tracing.WithRetryCount(3),
    tracing.WithExponentialBackoff(true),
)
```

## DynamoDB Key Design

> [Example](examples.md#dynamodb)

Use composite keys (PK + SK), GSIs for query patterns.

## Feature Flags

> [Example](examples.md#feature-flags)

Fallback defaults, check at service layer, log evaluations.

## JSON Tags

> [Example](examples.md#json-tags)

Use camelCase for JSON tags and query params.

## Request Validation

> [Example](examples.md#validation)

Use `ozzo-validation` for struct validation.

## Libraries

| Library | Purpose |
|---------|---------|
| `chi` | HTTP router |
| `samber/lo` | Collections/utilities |
| `ozzo-validation` | Validation |
| `resty` | HTTP client |