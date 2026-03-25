# Go Project Layout

Go does not mandate a specific directory structure. The compiler only cares about `main` packages and Go Modules. However, the community has converged on structural conventions that make codebases readable, maintainable, and predictable.

---

## 1. The Standard Go Project Layout

For anything larger than a single-file script or a tiny Lambda function, the community widely adopts a structure based on the [Standard Go Project Layout](https://github.com/golang-standards/project-layout).

A production-ready Go microservice:

```text
my-go-service/
├── cmd/
│   └── api/
│       └── main.go       # Entry point. Wires dependencies and starts the server.
├── internal/             # Code that CANNOT be imported by other projects.
│   ├── config/           # Environment variables and configuration parsing.
│   ├── user/             # Domain package: User feature (handler, service, repo).
│   ├── order/            # Domain package: Order feature.
│   └── platform/         # Shared infrastructure (database connections, logger setup).
├── pkg/                  # Code that CAN be imported by other projects (use sparingly).
│   └── httputil/         # Generic HTTP helpers, error formatters.
├── api/                  # OpenAPI specs, Protocol Buffer (.proto) files.
├── migrations/           # Raw SQL files for database schema changes.
├── go.mod                # Go module dependencies.
├── go.sum                # Dependency checksums.
└── Makefile              # Build, test, and run commands.
```

### Key Directories

- **`cmd/`**: Main applications for this project. The directory name should match the executable name (e.g., `cmd/api/main.go` builds an `api` binary). No business logic here — only dependency wiring and server startup.
- **`internal/`**: A special directory enforced by the Go compiler. Packages inside `internal/` can only be imported by code within the same parent directory tree. This is where your application code lives.
- **`pkg/`**: Library code that external applications may import. If you aren't building a shared library, you likely don't need `pkg/`. When in doubt, put it in `internal/`.

---

## 2. Package by Feature (Domain-Driven) vs. Package by Layer

When organizing code inside `internal/`, **avoid packaging by technical layer**.

### Anti-Pattern: Package by Layer

This mimics older MVC frameworks. Files that change together are spread far apart, often leading to circular dependency issues.

```text
internal/
├── controllers/  # user_controller.go, order_controller.go
├── services/     # user_service.go, order_service.go
├── repositories/ # user_repo.go, order_repo.go
└── models/       # user.go, order.go (the dreaded global "models" package)
```

### Idiomatic Go: Package by Feature (Domain)

Group code by what it does, not what it is. Everything related to a "User" lives in the `user` package.

```text
internal/
└── user/
    ├── handler.go    # HTTP handlers for users
    ├── service.go    # Business logic for users
    ├── repository.go # Database access for users
    └── user.go       # The User struct and domain errors
```

**Benefits:**
1. **High Cohesion:** Adding a new field to User means changes in exactly one folder.
2. **Encapsulation:** Repository implementation details can remain unexported (lowercase), exposing only the interface.
3. **No Circular Dependencies:** Domain packages (e.g., `user`, `order`) interact cleanly without import cycles.

---

## 3. The Flat Structure (For Micro-Apps)

For small CLI tools or single-purpose Lambda functions, a flat structure is perfectly idiomatic.

```text
tiny-worker/
├── main.go
├── processor.go
├── client.go
├── go.mod
└── go.sum
```

*Rule of thumb: Start flat. Introduce `cmd/` and `internal/` only when the project grows enough to warrant the overhead.*

---

## 4. Common Anti-Patterns

1. **The `util` or `common` Package:** Never create packages named `util`, `common`, `shared`, or `helpers`. These become dumping grounds. Name packages after what they provide (e.g., `stringset`, `jsonparser`, `errs`).
2. **Global Variables:** Avoid global `var DB *sql.DB` or global loggers. Initialize them in `cmd/api/main.go` and pass them explicitly via constructors (dependency injection).
3. **Deep Nesting:** Prefer shallow folder structures. Avoid `internal/app/core/domain/user/...`. Keep it simple.
