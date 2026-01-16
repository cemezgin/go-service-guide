# Concurrency Examples

## errgroup Parallel

### Parallel API Calls (Independent)

```go
func getOrderDetails(ctx context.Context, orderID, customerID, productID string) (*OrderDetails, error) {
    g, ctx := errgroup.WithContext(ctx)

    var order *Order
    var customer *Customer
    var product *Product

    g.Go(func() error {
        var err error
        order, err = s.orderClient.Get(ctx, orderID)
        return err
    })

    g.Go(func() error {
        var err error
        customer, err = s.customerClient.Get(ctx, customerID)
        return err
    })

    g.Go(func() error {
        var err error
        product, err = s.productClient.Get(ctx, productID)
        return err
    })

    if err := g.Wait(); err != nil {
        return nil, fmt.Errorf("get order details: %w", err)
    }

    return &OrderDetails{
        Order:    order,
        Customer: customer,
        Product:  product,
    }, nil
}
```

### Parallel API Calls (With Dependencies)

When calls depend on previous results, use two stages:

```go
func getOrderDetails(ctx context.Context, orderID string) (*OrderDetails, error) {
    // Stage 1: Get order first (has the IDs we need)
    order, err := s.orderClient.Get(ctx, orderID)
    if err != nil {
        return nil, fmt.Errorf("get order: %w", err)
    }

    // Stage 2: Fetch related data in parallel
    g, ctx := errgroup.WithContext(ctx)

    var customer *Customer
    var product *Product

    g.Go(func() error {
        var err error
        customer, err = s.customerClient.Get(ctx, order.CustomerID)
        return err
    })

    g.Go(func() error {
        var err error
        product, err = s.productClient.Get(ctx, order.ProductID)
        return err
    })

    if err := g.Wait(); err != nil {
        return nil, fmt.Errorf("get order details: %w", err)
    }

    return &OrderDetails{
        Order:    order,
        Customer: customer,
        Product:  product,
    }, nil
}
```

### Anti-patterns

```go
// ❌ WRONG: All sequential when some can be parallel
func getOrderDetails(ctx context.Context, orderID string) (*OrderDetails, error) {
    order, err := s.orderClient.Get(ctx, orderID)
    if err != nil { return nil, err }

    customer, err := s.customerClient.Get(ctx, order.CustomerID)
    if err != nil { return nil, err }

    product, err := s.productClient.Get(ctx, order.ProductID)
    if err != nil { return nil, err }

    return &OrderDetails{...}, nil
}

// ❌ WRONG: Fire-and-forget parallel requests
func getOrderDetails(ctx context.Context, orderID string) (*OrderDetails, error) {
    var order *Order
    var customer *Customer

    go func() { order, _ = s.orderClient.Get(ctx, orderID) }()
    go func() { customer, _ = s.customerClient.Get(ctx, "id") }()
    // No wait, no error handling!

    return &OrderDetails{...}, nil
}

// ❌ WRONG: Race condition - accessing order before it's fetched
func getOrderDetails(ctx context.Context, orderID string) (*OrderDetails, error) {
    g, ctx := errgroup.WithContext(ctx)
    var order *Order

    g.Go(func() error {
        var err error
        order, err = s.orderClient.Get(ctx, orderID)
        return err
    })

    g.Go(func() error {
        // BUG: order may be nil here!
        _, err := s.customerClient.Get(ctx, order.CustomerID)
        return err
    })

    return nil, g.Wait()
}
```

## Bounded Concurrency

```go
func processItems(ctx context.Context, items []Item, maxConcurrency int) error {
    g, ctx := errgroup.WithContext(ctx)
    g.SetLimit(maxConcurrency)  // Limit concurrent goroutines

    for _, item := range items {
        item := item  // Capture for goroutine
        g.Go(func() error {
            return process(ctx, item)
        })
    }

    return g.Wait()
}
```

### Usage

```go
// Process 100 items with max 10 concurrent goroutines
err := processItems(ctx, items, 10)
```

## Worker Pool

```go
func processItems(ctx context.Context, items []Item, numWorkers int) error {
    jobs := make(chan Item)
    g, ctx := errgroup.WithContext(ctx)

    // Start workers
    for i := 0; i < numWorkers; i++ {
        g.Go(func() error {
            for item := range jobs {
                if err := process(ctx, item); err != nil {
                    return err
                }
            }
            return nil
        })
    }

    // Send jobs
    g.Go(func() error {
        defer close(jobs)
        for _, item := range items {
            select {
            case <-ctx.Done():
                return ctx.Err()
            case jobs <- item:
            }
        }
        return nil
    })

    return g.Wait()
}
```

## Graceful Shutdown

```go
type Worker struct {
    ctx    context.Context
    cancel context.CancelFunc
    wg     sync.WaitGroup
}

func NewWorker(ctx context.Context) *Worker {
    ctx, cancel := context.WithCancel(ctx)
    return &Worker{ctx: ctx, cancel: cancel}
}

func (w *Worker) Start() {
    w.wg.Add(1)
    go func() {
        defer w.wg.Done()

        ticker := time.NewTicker(time.Second)
        defer ticker.Stop()

        for {
            select {
            case <-w.ctx.Done():
                slog.Info("worker shutting down")
                return
            case <-ticker.C:
                w.doWork()
            }
        }
    }()
}

func (w *Worker) Stop() {
    w.cancel()
    w.wg.Wait()
}
```

### Usage

```go
func main() {
    ctx := context.Background()
    worker := NewWorker(ctx)
    worker.Start()

    // Handle shutdown signal
    sigCh := make(chan os.Signal, 1)
    signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
    <-sigCh

    slog.Info("shutdown signal received")
    worker.Stop()
    slog.Info("shutdown complete")
}
```

## WaitGroup with Context

```go
func processItems(ctx context.Context, items []Item) error {
    var wg sync.WaitGroup

    for _, item := range items {
        wg.Add(1)
        go func(item Item) {
            defer wg.Done()

            select {
            case <-ctx.Done():
                return
            default:
                process(ctx, item)
            }
        }(item)
    }

    wg.Wait()
    return ctx.Err()
}
```

## Context Propagation

```go
// Always pass context to goroutines
g.Go(func() error {
    // Use the ctx from errgroup.WithContext
    return fetchData(ctx, id)
})

// Respect cancellation in loops
for {
    select {
    case <-ctx.Done():
        return ctx.Err()
    case item := <-ch:
        process(item)
    }
}
```
