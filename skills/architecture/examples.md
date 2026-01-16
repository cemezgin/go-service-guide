# Architecture Examples

## Folder Structure

```
cmd/                                 # Entrypoints
├── api/main.go
├── cli/main.go
└── consumer/main.go

config/appconfig/                    # Configuration
├── local.json
├── development.json
├── staging.json
└── production.json

infra/                               # AWS CDK (Go)
├── stack.go
└── constructs/*.go

internal/
├── domainname/              # Bounded context
│   ├── input.go                     # Input DTOs
│   ├── output.go                    # Output DTOs
│   ├── usecase/
│   │   ├── service.go               # Business logic + interfaces
│   │   ├── service_test.go
│   │   └── service_mock.go
│   ├── repository/
│   │   ├── repository.go
│   │   └── repository_mock.go
│   └── model/entity.go
├── clients/                         # External APIs
│   ├── external1/
│   │   ├── client.go
│   │   └── model.go
│   └── external2/
│       ├── client.go
│       └── model.go
├── transport/                       # Driving adapters
│   ├── http/
│   └── kinesis/
└── apps/api/app.go                  # Dependency wiring
```

## Interface Ownership

```go
// internal/order/usecase/service.go
// Interface defined in consumer, only methods needed

//go:generate mockgen -source=service.go -destination=service_mock.go -package=order

type customerFetcher interface {
    GetCustomer(ctx context.Context, id string) (*Customer, error)
}

type orderRepository interface {
    Save(ctx context.Context, o *Order) error
    FindByID(ctx context.Context, id string) (*Order, error)
}

type Service struct {
    customers customerFetcher
    repo      orderRepository
}

func NewService(customers customerFetcher, repo orderRepository) *Service {
    return &Service{customers: customers, repo: repo}
}
```

```go
// internal/clients/customer/client.go
// Implementation only, no interface defined here

type Client struct {
    http *resty.Client
}

func (c *Client) GetCustomer(ctx context.Context, id string) (*Customer, error) {
    // Satisfies order.customerFetcher interface
}

func (c *Client) ListCustomers(ctx context.Context) ([]*Customer, error) {
    // Additional method - other usecases can define interfaces that include this
}
```

### Anti-pattern: Shared Interface Package

```go
// ❌ WRONG: Generic shared interface package
// internal/ports/customer.go
type CustomerClient interface {
    GetCustomer(ctx context.Context, id string) (*Customer, error)
    ListCustomers(ctx context.Context) ([]*Customer, error)
    UpdateCustomer(ctx context.Context, c *Customer) error
    DeleteCustomer(ctx context.Context, id string) error
}
// Forces consumers to depend on methods they don't need
```

## Domain Structure

```go
// internal/order/usecase/service.go
package usecase

//go:generate mockgen -source=service.go -destination=service_mock.go -package=usecase

// Interfaces defined here - only methods this service needs
type externalClient interface {
    GetOrder(ctx context.Context, orderID, referenceID string) (*model.Order, error)
}

type orderRepository interface {
    Save(ctx context.Context, o *model.Order) error
    FindByOrderID(ctx context.Context, orderID string) (*model.Order, error)
}

type Service struct {
    external externalClient
    repo     orderRepository
}

func NewService(external externalClient, repo orderRepository) *Service {
    return &Service{external: external, repo: repo}
}

func (s *Service) VerifyOrder(ctx context.Context, orderID, referenceID string) (*output.OrderResult, error) {
    // Business logic here
}
```

```go
// internal/order/input.go
package order

type VerifyOrderRequest struct {
    OrderID     string `json:"orderID"`
    ReferenceID string `json:"referenceID"`
}
```

```go
// internal/order/output.go
package order

type OrderResult struct {
    IsValid bool   `json:"isValid"`
    Status  string `json:"status"`
}
```
