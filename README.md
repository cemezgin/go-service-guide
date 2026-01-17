# Go Service Guide

A Claude Code plugin providing comprehensive Go service development standards covering architecture, style, patterns, concurrency, testing, and observability.

## Installation

### Via Claude Code Marketplace

```bash
# Add the marketplace source
claude /plugin marketplace add cemezgin/go-service-guide

# Install the plugin
claude /plugin install coding-standards
```

### Manual Installation

Clone the repository and add it as a local plugin:

```bash
git clone https://github.com/cemezgin/go-service-guide.git
claude /plugin add ./coding-standards
```

## Updating

To fetch the latest version of the plugin:

```bash
# Reinstall to get the latest version
claude /plugin update go-service-guide

# Or manually: uninstall and reinstall
claude /plugin uninstall go-service-guide
claude /plugin install go-service-guide
```

For manual installations, pull the latest changes:

```bash
cd go-service-guide
git pull origin main
```

## Skills

Once installed, the following skills become available in your projects:

| Skill | Description | Trigger |
|-------|-------------|---------|
| **architecture** | Clean/hexagonal architecture rules, folder structure, interface ownership, and dependency direction | Creating new packages or services |
| **patterns** | Functional options, must pattern, caching decorator, error handling, configuration, and HTTP clients | Implementing common patterns |
| **testing** | Table-driven tests, mock setup, coverage requirements (85% minimum) | Writing tests |
| **concurrency** | Goroutine ownership, errgroup patterns, worker pools, and graceful shutdown | Writing concurrent code |
| **observability** | Tracing with OpenTelemetry and structured logging with slog | Adding observability |
| **go-style** | Uber Go style guide rules for guidelines, performance, and code style | Writing or reviewing Go code |

## Usage

Skills are automatically invoked by Claude Code when relevant context is detected. You can also manually invoke them:

```bash
# Invoke a specific skill
claude /go-service-guide:architecture
claude /go-service-guide:patterns
claude /go-service-guide:testing
claude /go-service-guide:concurrency
claude /go-service-guide:observability
claude /go-service-guide:go-style
```

## Skill Highlights

### Architecture
- Dependency rule: outer layers depend on inner, never reverse
- Interface ownership: defined where used, not implemented
- Standard folder structure: `cmd/`, `internal/`, `config/`, `infra/`

### Patterns
- Functional options with `With*` functions
- Must pattern (allowed only in `cmd/`, `internal/apps/`)
- Error wrapping with context: `fmt.Errorf("get order %s: %w", id, err)`

### Testing
- Table-driven tests for multiple scenarios
- Mock generation with `gomock`
- 85% coverage requirement

### Concurrency
- No fire-and-forget goroutines
- Use `errgroup` for parallel requests
- Bounded concurrency with `errgroup.SetLimit(n)`

### Observability
- Tracing mandatory for repositories, clients, lambdas, usecases
- Structured logging with `log/slog`
- Proper Lambda lifecycle with OpenTelemetry setup

### Go Style
- Follows Uber Go Style Guide
- Performance optimizations (strconv over fmt, capacity hints)
- Consistent naming and formatting conventions

## Structure

```
go-service-guide/
├── .claude-plugin/
│   ├── plugin.json        # Plugin manifest
│   └── marketplace.json   # Marketplace configuration
└── skills/
    ├── architecture/
    │   ├── SKILL.md       # Skill definition
    │   └── examples.md    # Code examples
    ├── concurrency/
    ├── go-style/
    ├── observability/
    ├── patterns/
    └── testing/
```

## Contributing

1. Fork the repository
2. Create a feature branch
3. Add or update skill content in the appropriate `skills/` directory
4. Submit a pull request

## License

MIT
