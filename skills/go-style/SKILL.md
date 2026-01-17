---
name: go-style
description: Uber Go style guide rules for guidelines, performance, and code style. Use when writing or reviewing Go code for style compliance.
userInvokable: true
---

# Go Style (Uber Guide)

**References:** [Uber Go Style Guide](https://github.com/uber-go/guide/blob/master/style.md) | [Examples](examples.md)

## Guidelines

> [Examples](examples.md#guidelines)

| Rule | Do | Don't |
|------|-----|-------|
| Interface pointers | `func(w io.Writer)` | `func(w *io.Writer)` |
| Interface compliance | `var _ Interface = (*T)(nil)` | Runtime check only |
| Receivers | Consistent pointer `(s *Service)` | Mix value/pointer on same type |
| Zero-value mutex | `var mu sync.Mutex` (ready to use) | `mu := new(sync.Mutex)` |
| Slice/map boundaries | Copy at API boundaries | Return/store internal slices/maps |
| Defer | For cleanup, outside loops | `defer` inside loops |
| Channel size | 0 (sync) or 1 (async) | Arbitrary buffer sizes |
| Enums | Start at 1 if zero invalid | Start at 0 when zero is invalid |
| Time | `time.Time`, `time.Duration` | `int64` for timestamps/durations |
| Error types | `var ErrFoo = errors.New()` | String comparison |
| Error wrapping | `fmt.Errorf("...: %w", err)` | `fmt.Errorf("...: %s", err)` |
| Error naming | `ErrFoo`, `FooError` type | `FooErr`, `ErrorFoo` |
| Error handling | Handle once, immediately with `:=` | Log and return, `var err` at top |
| Type assertion | `v, ok := x.(T)` check ok | `v := x.(T)` panics |
| Panic | `must.Must` in `cmd/`, `internal/apps/` only | Panic in usecases/clients/repos |
| Atomic | `go.uber.org/atomic` | `sync/atomic` directly |
| Global state | Dependency injection | Mutable globals |
| Embedding | Private structs only | Embed types in public structs |
| Built-in names | Custom names | Shadow `error`, `int`, `string`, etc |
| init() | Avoid, or deterministic only | Complex logic in `init()` |
| Exit | `os.Exit` only in `main()` | `os.Exit` in library |
| Field tags | Always on marshaled structs | Missing `json:` tags |
| Goroutines | Managed lifecycle, wait groups | Fire-and-forget, goroutines in `init()` |

## Performance

> [Examples](examples.md#performance)

| Rule | Do | Don't |
|------|-----|-------|
| Int conversion | `strconv.Itoa(n)` | `fmt.Sprintf("%d", n)` |
| String/byte | Reuse `[]byte`, minimize conversion | Repeated `[]byte(s)` in loops |
| Map capacity | `make(map[K]V, hint)` | Grow map incrementally |
| Slice capacity | `make([]T, 0, cap)` | Append without capacity |

## Style

> [Examples](examples.md#style)

| Rule | Do | Don't |
|------|-----|-------|
| Line length | ~99 chars, break logically | Overly long lines |
| Consistency | Match surrounding code style | Mix styles in same file |
| Declarations | Group similar `var`, `const`, `type` | Scatter related declarations |
| Imports | stdlib → external → internal | Mixed import groups |
| Package names | Short, lowercase, singular | `_`, `util`, `common`, `base` |
| Function names | `MarshalJSON`, no stuttering | `json.JSONMarshal` |
| Import alias | Only on conflict | Unnecessary aliases |
| Function order | Exported first, by receiver, call order | Random ordering |
| Nesting | Early return, max 2 levels | Deep nesting |
| Else | Omit after `return`/`break`/`continue` | `if x { return } else { ... }` |
| Top-level var | `var x = ...` not `:=` | `:=` at package level |
| Unexported globals | `var _cache = ...` | `var cache` (no underscore) |
| Struct embedding | Embed interfaces, mutex private | Embed concrete types publicly |
| Local var | `:=` for most, `var` for zero | `var x int = 0` |
| Nil slice | Return `nil` not `[]T{}` | Empty literal when nil works |
| Variable scope | Declare near use, reduce scope | All vars at function top |
| Naked parameters | Struct for 3+ params | `func(a, b, c, d, e string)` |
| Raw strings | `` `json:"x"` `` | `"json:\"x\""` |
| Struct init | Field names `{Name: "x"}` | Positional `{"x", 1}` |
| Zero fields | Omit zero value fields | `{Name: "", Count: 0}` |
| Zero struct | `var s S` | `s := S{}` |
| Struct refs | `&S{Name: "x"}` | `s := S{}; return &s` |
| Map init | `make(map[K]V)` or literal | `nil` map then assign |
| Printf format | Define outside call | Format strings inline |
| Printf naming | `Foof(format, ...)` | `FooFormat`, `FooPrintf` |
| Getters | `Name()` | `GetName()` |
| Constants | Named constants | Magic numbers |