---
name: architecture
description: Clean/hexagonal architecture rules, folder structure, interface ownership, and dependency direction. Use when creating new packages, services, or understanding codebase structure.
---

# Clean Architecture

**References:** [Examples](examples.md)

## Dependency Rule

Dependencies point inward. Outer layers depend on inner, never reverse. Avoid circular dependencies.

| Layer | Contains | Must NOT |
|-------|----------|----------|
| Usecase | Entities, business logic, interfaces, repository | Import transport or apps |
| Driving Adapters (transport) | HTTP handlers, consumers | Contain business logic |
| Driven Adapters (clients) | External API implementations | Import transport or usecase |

## Interface Ownership

> [Example](examples.md#interface-ownership)

- **Interfaces defined where used, not where implemented**
- Each usecase defines only methods it needs (Interface Segregation)
- Interfaces are unexported and private to package
- No `ports/`, `interfaces/`, or `contracts/` packages

## Folder Structure

> [Example](examples.md#folder-structure)

| Location | Contains |
|----------|----------|
| `cmd/` | Lambda/CLI entrypoints |
| `config/appconfig/` | Environment configurations |
| `infra/` | CDK stack + constructs |
| `internal/<domain>/` | Self-contained bounded contexts (usecase/, repository/, model/) |
| `internal/clients/` | External API clients + models |
| `internal/transport/` | HTTP handlers, Kinesis consumers |
| `internal/apps/` | Dependency wiring |

## Package Boundaries

- `internal/` - Private to service
- `pkg/` - Shared utilities for other repos

## Domain Structure

> [Example](examples.md#domain-structure)

Each domain in `internal/<domain>/` follows:

```
<domain>/
├── input.go           # Input DTOs
├── output.go          # Output DTOs
├── usecase/
│   ├── service.go     # Business logic + interfaces
│   └── service_mock.go
├── repository/
│   └── repository.go
└── model/
    └── entity.go      # Domain entities
```
