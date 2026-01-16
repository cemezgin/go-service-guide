---
name: concurrency
description: Goroutine ownership, errgroup patterns, worker pools, and graceful shutdown. Use when writing concurrent code or parallel request handling.
---

# Concurrency

**References:** [Examples](examples.md)

## Core Rule

**No fire-and-forget goroutines.** Every goroutine must have managed lifecycle.

| Requirement | Description |
|-------------|-------------|
| Context cancellation | Pass `ctx` and respect `ctx.Done()` for graceful shutdown |
| Synchronization | Use `sync.WaitGroup` or `errgroup.Group` to wait for completion |
| Bounded concurrency | Limit parallel goroutines with semaphore or worker pool |
| Error propagation | Use `errgroup` when errors need to bubble up |
| Parallel requests | **Always use `errgroup`** for parallel HTTP/DB/API calls |

## Pattern Selection

| Pattern | Use When |
|---------|----------|
| `sync.WaitGroup` | Simple fan-out, no error collection needed |
| `errgroup.Group` | Need first error, automatic cancellation |
| `errgroup.SetLimit(n)` | Bounded concurrency with error handling |
| Worker pool | High-volume processing, backpressure needed |

## Do / Don't

| Do | Don't |
|----|-------|
| `errgroup.Group` with context | `go func() { process() }()` |
| Worker pool with fixed size | Unbounded `for { go handle() }` |
| `select` on `ctx.Done()` | Ignore cancellation signals |
| Explicit `wg.Wait()` | Hope goroutines finish |

## errgroup for Parallel Requests

> [Example](examples.md#errgroup-parallel)

```go
g, ctx := errgroup.WithContext(ctx)

g.Go(func() error { return fetchA(ctx) })
g.Go(func() error { return fetchB(ctx) })
g.Go(func() error { return fetchC(ctx) })

if err := g.Wait(); err != nil {
    return fmt.Errorf("parallel fetch: %w", err)
}
```

## Bounded Concurrency

> [Example](examples.md#bounded-concurrency)

```go
g, ctx := errgroup.WithContext(ctx)
g.SetLimit(10)  // Max 10 concurrent goroutines

for _, item := range items {
    item := item
    g.Go(func() error { return process(ctx, item) })
}

return g.Wait()
```

## Worker Pool

> [Example](examples.md#worker-pool)

For high-volume processing with backpressure control.

## Graceful Shutdown

> [Example](examples.md#graceful-shutdown)

Always handle context cancellation in long-running goroutines.
