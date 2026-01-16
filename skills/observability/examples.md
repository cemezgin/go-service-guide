# Observability Examples

## Tracing

### Service Layer

```go
func (s *Service) GetOrder(ctx context.Context, id string) (*Order, error) {
    ctx, span := tracing.StartSpan(ctx, "Service.GetOrder")
    defer span.End()

    span.SetAttributes(attribute.String("order.id", id))

    order, err := s.repo.FindByID(ctx, id)
    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, err.Error())
        return nil, fmt.Errorf("get order: %w", err)
    }

    return order, nil
}
```

### Repository Layer

```go
func (r *Repository) FindByID(ctx context.Context, id string) (*Order, error) {
    ctx, span := tracing.StartSpan(ctx, "Repository.FindByID")
    defer span.End()

    span.SetAttributes(
        attribute.String("db.table", r.tableName),
        attribute.String("db.operation", "GetItem"),
    )

    result, err := r.client.GetItem(ctx, &dynamodb.GetItemInput{
        TableName: aws.String(r.tableName),
        Key:       map[string]types.AttributeValue{"PK": &types.AttributeValueMemberS{Value: id}},
    })
    if err != nil {
        span.RecordError(err)
        return nil, err
    }

    return unmarshal(result.Item)
}
```

### Client Layer

```go
func (c *Client) GetCustomer(ctx context.Context, id string) (*Customer, error) {
    ctx, span := tracing.StartSpan(ctx, "CustomerClient.GetCustomer")
    defer span.End()

    span.SetAttributes(
        attribute.String("http.method", "GET"),
        attribute.String("customer.id", id),
    )

    resp, err := c.http.R().
        SetContext(ctx).
        SetResult(&Customer{}).
        Get(fmt.Sprintf("/customers/%s", id))

    if err != nil {
        span.RecordError(err)
        return nil, fmt.Errorf("get customer %s: %w", id, err)
    }

    span.SetAttributes(attribute.Int("http.status_code", resp.StatusCode()))

    return resp.Result().(*Customer), nil
}
```

## Logging

### Good Patterns

```go
// Structured with context
slog.InfoContext(ctx, "order retrieved",
    "order_id", orderID,
    "reference_id", referenceID,
)

// Warning with recovery info
slog.WarnContext(ctx, "cache miss, fetching from source",
    "key", cacheKey,
    "fallback", "api",
)

// Error with details
slog.ErrorContext(ctx, "failed to save order",
    "order_id", orderID,
    "error", err,
)

// Debug for development
slog.DebugContext(ctx, "processing request",
    "payload", payload,
)
```

### Anti-patterns

```go
// ❌ Printf style logging
log.Printf("order retrieved: %s, %s", orderID, referenceID)

// ❌ String concatenation
slog.Info("order " + orderID + " retrieved")

// ❌ Missing context
slog.Info("order retrieved", "order_id", orderID)

// ❌ Logging sensitive data
slog.Info("user authenticated", "password", password)
```

## Lambda Lifecycle

```go
func main() {
    ctx := context.Background()

    // Setup OpenTelemetry
    otelShutdown, err := otel.Setup(ctx)
    if err != nil {
        fmt.Printf("Error setting up OpenTelemetry: %v\n", err)
        os.Exit(1)
    }

    // Ensure graceful shutdown
    defer func() {
        if err := otelShutdown(context.Background()); err != nil {
            fmt.Printf("Error shutting down: %v\n", err)
        }
    }()

    // Start Lambda with instrumentation
    lambda.Start(otellambda.InstrumentHandler(handler))
}

func handler(ctx context.Context, event events.APIGatewayProxyRequest) (events.APIGatewayProxyResponse, error) {
    ctx, span := tracing.StartSpan(ctx, "Handler")
    defer span.End()

    span.SetAttributes(
        attribute.String("http.method", event.HTTPMethod),
        attribute.String("http.path", event.Path),
    )

    // Handler logic...

    return events.APIGatewayProxyResponse{
        StatusCode: 200,
        Body:       responseBody,
    }, nil
}
```

## HTTP Client

```go
// Create traced HTTP client
client := tracing.NewRestyClient(baseURL,
    tracing.WithTimeout(30*time.Second),
    tracing.WithRetryCount(3),
    tracing.WithRetryWaitTime(100*time.Millisecond),
    tracing.WithRetryMaxWaitTime(2*time.Second),
    tracing.WithExponentialBackoff(true),
)

// Usage - tracing is automatic
resp, err := client.R().
    SetContext(ctx).
    SetResult(&Response{}).
    Get("/endpoint")
```

## Adding Span Attributes

```go
// Basic attributes
span.SetAttributes(
    attribute.String("user.id", userID),
    attribute.Int("items.count", len(items)),
    attribute.Bool("cache.hit", cacheHit),
)

// Error handling
if err != nil {
    span.RecordError(err)
    span.SetStatus(codes.Error, err.Error())
}

// Events
span.AddEvent("cache_miss", trace.WithAttributes(
    attribute.String("key", cacheKey),
))
```
