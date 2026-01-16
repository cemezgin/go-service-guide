# Go Style Examples

## Guidelines

```go
// Interface directly, not pointer
func process(w io.Writer) {}  // not *io.Writer

// Compile-time interface check
var _ http.Handler = (*Server)(nil)

// Consistent receivers
func (s *Service) Get() {}
func (s *Service) Set() {}

// Zero-value mutex ready to use
type Cache struct {
    mu    sync.Mutex
    items map[string]string
}

// Copy at boundary
func (s *Store) Items() []Item {
    result := make([]Item, len(s.items))
    copy(result, s.items)
    return result
}

// Defer outside loop
for _, f := range files {
    func() {
        f, _ := os.Open(f)
        defer f.Close()
    }()
}

// Channel: unbuffered or 1
ch := make(chan int)    // or make(chan int, 1)

// Enum starts at 1
const (
    _ Status = iota
    StatusPending  // 1
    StatusActive   // 2
)

// Type assertion with ok
val, ok := x.(string)
if !ok { return errors.New("not a string") }

// Managed goroutine
var wg sync.WaitGroup
wg.Add(1)
go func() {
    defer wg.Done()
    process(data)
}()
wg.Wait()
```

### Anti-patterns

```go
// ❌ Pointer to interface
func process(w *io.Writer) {}

// ❌ Mixed receivers
func (s Service) Get() {}
func (s *Service) Set() {}

// ❌ Copy internal slice
func (s *Store) Items() []Item { return s.items }

// ❌ Defer in loop
for _, f := range files {
    f, _ := os.Open(f)
    defer f.Close()
}

// ❌ Arbitrary channel size
ch := make(chan int, 100)

// ❌ Enum starts at 0 (invalid zero)
type Status int
const StatusPending Status = iota  // 0

// ❌ Type assertion without ok
val := x.(string)

// ❌ Panic in library
func Parse(s string) *Config {
    if s == "" { panic("empty") }
}

// ❌ Mutable global
var cache = map[string]string{}

// ❌ Embed in public struct (exposes Lock/Unlock)
type Cache struct {
    sync.Mutex
    data map[string]int
}

// ❌ Fire-and-forget goroutine
go process(data)
```

## Performance

```go
// strconv over fmt
s := strconv.Itoa(n)  // not fmt.Sprintf("%d", n)

// Convert once
b := []byte(s)
for _, v := range data { process(b) }

// With capacity
m := make(map[string]int, 100)
s := make([]int, 0, 100)
```

### Anti-patterns

```go
// ❌ fmt for int conversion
s := fmt.Sprintf("%d", n)

// ❌ Repeated conversion
for _, v := range data {
    process([]byte(s))
}

// ❌ No capacity hint
m := make(map[string]int)
s := make([]int, 0)
```

## Style

```go
// Early return
func process(x int) error {
    if x <= 0 { return nil }
    if x >= 100 { return nil }
    return doWork(x)
}

// No else after return
if err != nil { return err }
return nil

// Inline :=
if err := step1(); err != nil { return err }

// Field names in struct
o := Order{ID: "123", Status: "active"}

// var for zero value
var s Service

// Named constants
const maxRetries = 3
if retries < maxRetries {}

// Struct for params
type CreateParams struct {
    Name, Email, Phone, Addr, City string
}
func Create(p CreateParams) {}

// Import organization
import (
    // Standard library
    "context"
    "fmt"

    // Third-party
    "github.com/go-resty/resty/v2"
    "github.com/samber/lo"

    // Internal
    "github.com/myorg/myservice/internal/config"
)
```

### Anti-patterns

```go
// ❌ Deep nesting
func process(x int) error {
    if x > 0 {
        if x < 100 {
            if x != 50 {
                return doWork(x)
            }
        }
    }
    return nil
}

// ❌ Unnecessary else
if err != nil {
    return err
} else {
    return nil
}

// ❌ var err at top
func process() error {
    var err error
    err = step1()
    if err != nil { return err }
    err = step2()
    return err
}

// ❌ Positional struct
o := Order{"123", "active", 30}

// ❌ Zero value struct literal
s := Service{}

// ❌ Magic numbers
if score > 85 && retries < 3 {}

// ❌ Many parameters
func Create(name, email, phone, addr, city string) {}

// ❌ Getter with Get prefix
func (u *User) GetName() string {}
```
