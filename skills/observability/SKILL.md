---
name: observability
description: Tracing with OpenTelemetry and structured logging with slog. Use when adding observability to services, handlers, or debugging production issues.
---

# Observability

**References:** [Examples](examples.md)

## Tracing (OpenTelemetry)

> [Example](examples.md#tracing)

**Mandatory for:** repositories, API clients, Lambdas, usecases

```go
func (s *Service) GetOrder(ctx context.Context, id string) (*Order, error) {
    ctx, span := tracing.StartSpan(ctx, "Service.GetOrder")
    defer span.End()

    span.SetAttributes(attribute.String("order.id", id))
    // business logic...
}
```

## Structured Logging (slog)

> [Example](examples.md#logging)

Use `log/slog` with structured JSON:

```go
slog.InfoContext(ctx, "order retrieved",
    "order_id", orderID,
    "reference_id", referenceID,
)
```

## Log Levels

| Level | Use When |
|-------|----------|
| INFO | Normal operations, successful requests |
| WARN | Recoverable errors, degraded state, fallbacks used |
| ERROR | Failures requiring attention, unrecoverable errors |

## Lambda Lifecycle

> [Example](examples.md#lambda-lifecycle)

```go
func main() {
    ctx := context.Background()

    otelShutdown, err := otel.Setup(ctx)
    if err != nil {
        fmt.Printf("Error setting up OpenTelemetry: %v\n", err)
        os.Exit(1)
    }

    defer func() {
        if err := otelShutdown(context.Background()); err != nil {
            fmt.Printf("Error shutting down: %v\n", err)
        }
    }()

    lambda.Start(otellambda.InstrumentHandler(handler))
}
```

## HTTP Client Tracing

> [Example](examples.md#http-client)

```go
client := tracing.NewRestyClient(baseURL,
    tracing.WithTimeout(30*time.Second),
    tracing.WithRetryCount(3),
)
```
