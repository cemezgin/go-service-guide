# Project Patterns Examples

## Functional Options

```go
type ClientOption func(*ClientOptions)

type ClientOptions struct {
    Timeout    time.Duration
    RetryCount int
    BaseURL    string
}

func WithTimeout(timeout time.Duration) ClientOption {
    return func(opts *ClientOptions) {
        opts.Timeout = timeout
    }
}

func WithRetryCount(count int) ClientOption {
    return func(opts *ClientOptions) {
        opts.RetryCount = count
    }
}

func NewClient(baseURL string, options ...ClientOption) *Client {
    opts := &ClientOptions{
        Timeout:    30 * time.Second,
        RetryCount: 3,
    }
    for _, option := range options {
        option(opts)
    }
    // create client...
}

// Usage
client := NewClient(baseURL,
    WithTimeout(30*time.Second),
    WithRetryCount(3),
)
```

## Must Pattern

**Allowed only in startup/wiring paths:** `cmd/`, `internal/apps/`

```go
func Must[T any](f func() (T, error)) T {
    v, err := f()
    if err != nil {
        panic(err)
    }
    return v
}

// ✅ CORRECT: must.Must in internal/apps/ (wiring)
// internal/apps/api/app.go
func (a *App) CustomerClient() *customer.Client {
    return must.Must(func() (*customer.Client, error) {
        return customer.New(a.cfg.CustomerService)
    })
}

// ✅ CORRECT: must.Must in cmd/ (entrypoint)
// cmd/api/main.go
func main() {
    cfg := must.Must(func() (*config.Config, error) {
        return config.Load()
    })
}
```

### Anti-pattern

```go
// ❌ WRONG: must.Must in usecase/client/repository
func (s *Service) GetOrder(ctx context.Context, id string) (*Order, error) {
    // Never panic here - return error instead
    result := must.Must(func() (*Order, error) {
        return s.repo.Find(ctx, id)
    })
}
```

## Caching Decorator

```go
type CachedClient struct {
    client externalClient
    cache  cacheAdapter
    config CacheConfig
}

func NewCachedClient(client externalClient, cache cacheAdapter, config CacheConfig) *CachedClient {
    return &CachedClient{client: client, cache: cache, config: config}
}

func (c *CachedClient) GetOrder(ctx context.Context, orderID, referenceID string) (*Order, error) {
    key := fmt.Sprintf("order:%s:%s", orderID, referenceID)

    // Check cache first
    if cached, ok := c.cache.Get(ctx, key); ok {
        return cached, nil
    }

    // Fetch from source
    result, err := c.client.GetOrder(ctx, orderID, referenceID)
    if err != nil {
        return nil, err
    }

    // Store in cache
    c.cache.Set(ctx, key, result, c.config.TTL)
    return result, nil
}
```

## Error Handling

```go
// Custom error types
type NotFoundError struct {
    message string
}

func NewNotFoundError(id string) *NotFoundError {
    return &NotFoundError{fmt.Sprintf("resource %s not found", id)}
}

func (e NotFoundError) Error() string { return e.message }

func (e NotFoundError) Is(err error) bool {
    _, matchType := err.(*NotFoundError)
    return matchType
}

// Naming convention
var ErrBrokenLink = errors.New("broken link")  // Err prefix for variables
type ValidationError struct { ... }             // Error suffix for types

// Wrap errors with context
if err != nil {
    return fmt.Errorf("get order %s: %w", orderID, err)
}

// Use errors.Is() for checking
if errors.Is(err, ErrOrderNotFound) {
    // handle not found
}

// Handle errors once - don't log AND return
if err != nil {
    return fmt.Errorf("get order: %w", err)  // Let caller decide
}

// HTTP error response
func handleError(err error) Response {
    switch {
    case errors.Is(err, ErrOrderNotFound):
        return Response{Status: 404, Message: "Order not found"}
    case errors.Is(err, ErrInvalidInput):
        return Response{Status: 400, Message: "Invalid input"}
    default:
        slog.Error("internal error", "error", err)
        return Response{Status: 500, Message: "Internal server error"}
    }
}
```

## Configuration

### Duration Fields

```go
type DynamoDBConfig struct {
    TableName string        `json:"table_name"`
    Timeout   time.Duration `json:"timeout"`
}

func (c *DynamoDBConfig) UnmarshalJSON(data []byte) error {
    type Alias DynamoDBConfig
    aux := &struct {
        Timeout string `json:"timeout"`
        *Alias
    }{Alias: (*Alias)(c)}

    if err := json.Unmarshal(data, &aux); err != nil {
        return err
    }
    if aux.Timeout != "" {
        duration, err := time.ParseDuration(aux.Timeout)
        if err != nil {
            return err
        }
        c.Timeout = duration
    }
    return nil
}
```

### JSON Format

```json
{
  "dynamodb": {
    "table_name": "myservice-local",
    "timeout": "30s"
  }
}
```

## HTTP Client

```go
client := tracing.NewRestyClient(baseURL,
    tracing.WithTimeout(30*time.Second),
    tracing.WithRetryCount(3),
    tracing.WithRetryWaitTime(100*time.Millisecond),
    tracing.WithRetryMaxWaitTime(2*time.Second),
    tracing.WithExponentialBackoff(true),
)
```

## DynamoDB

```go
func GeneratePK(orderID string) string {
    return fmt.Sprintf("ORDER#%s", orderID)
}

func GenerateSK(referenceID string) string {
    return fmt.Sprintf("REF#%s", referenceID)
}

// Query example
input := &dynamodb.QueryInput{
    TableName:              aws.String(tableName),
    KeyConditionExpression: aws.String("PK = :pk AND begins_with(SK, :sk)"),
    ExpressionAttributeValues: map[string]types.AttributeValue{
        ":pk": &types.AttributeValueMemberS{Value: GeneratePK(orderID)},
        ":sk": &types.AttributeValueMemberS{Value: "REF#"},
    },
}
```

## Feature Flags

```go
func (s *Service) IsFeatureEnabled(ctx context.Context, feature string) bool {
    enabled, err := s.splitClient.Treatment(ctx, feature)
    if err != nil {
        slog.WarnContext(ctx, "feature flag check failed, using default",
            "feature", feature,
            "error", err,
        )
        return false // Safe default
    }
    return enabled == "on"
}
```

## JSON Tags

```go
type Order struct {
    OrderID     string `json:"orderID"`
    ReferenceID string `json:"referenceID"`
    IsValid     bool   `json:"isValid"`
}

type ListOrdersRequest struct {
    PageSize   int `query:"pageSize"`
    PageNumber int `query:"pageNumber"`
}

type Response struct {
    Data    *Order `json:"data,omitempty"`
    Error   string `json:"error,omitempty"`
    TraceID string `json:"traceId,omitempty"`
}
```

## Validation

```go
func (r CreateOrderRequest) Validate() error {
    return validation.ValidateStruct(&r,
        validation.Field(&r.OrderID, validation.Required, validation.Length(1, 50)),
        validation.Field(&r.ReferenceID, validation.Required),
        validation.Field(&r.Email, validation.Required, is.Email),
    )
}

// Usage in handler
func (h *Handler) CreateOrder(w http.ResponseWriter, r *http.Request) {
    var req CreateOrderRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "invalid json", http.StatusBadRequest)
        return
    }

    if err := req.Validate(); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    // proceed with valid request...
}
```
