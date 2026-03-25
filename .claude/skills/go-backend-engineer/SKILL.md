---
name: go-backend-engineer
description: Core architectural guidelines and idiomatic practices for building Go backend services. Use when user asks to "architect a new service," "set up routing," "handle database connections," or when migrating concepts from Spring Boot to Go.
---

# Go Backend Engineer Skill

This guide outlines the core philosophies and architectural patterns for building robust, scalable, and idiomatic Go backend services. 

## The Paradigm Shift: Spring Boot vs. Go
When building backends in Go, the overarching rule is **explicitness over magic**. 

| Concept | Spring Boot Approach | Idiomatic Go Approach |
|---------|----------------------|-----------------------|
| **Architecture** | Heavy layering (Controller -> Service -> Repository), heavy abstraction | Pragmatic layers, flat structure for small apps, domain-driven for large ones |
| **Dependency Injection** | Magic `@Autowired`, IoC Container | Explicit constructor injection (`NewService(repo)`) |
| **Web / Routing** | Heavy framework (`@RestController`), complex filter chains | Standard library `net/http` + lightweight router (e.g., `chi` or Go 1.22+ mux) |
| **Request Scope** | ThreadLocal magic, `@RequestScope` | Explicitly passing `context.Context` as the first parameter |
| **Database** | JPA / Hibernate (complex ORM magic) | `database/sql`, `sqlx`, or structural ORMs like `gorm` / `ent` |
| **Concurrency** | Thread pools, `@Async`, WebFlux | Native Goroutines, Channels, and `sync` package |

---

## Core Architectural Pillars

### 1. Project Layout
Go does not mandate a specific directory structure, but throwing everything into a single package or creating deep `src/main/java/com/...` trees are both anti-patterns.

* **Small/Microservices:** Use a flat structure (all files in the root or a single `internal` folder).
* **Standard Layout:** Follow the community standard for larger projects:
    * `cmd/api/main.go`: The entry point. Wires up dependencies and starts the server.
    * `internal/`: Private application code. Prevents other projects from importing your business logic.
    * `pkg/`: Exported library code (use sparingly).
* **Domain-Driven:** Group by feature/domain (e.g., `internal/users`, `internal/orders`) rather than technical concern (e.g., avoiding global `models/`, `controllers/`, `services/` folders).

### 2. Web and Middleware (`net/http`)
Rely on the standard library as much as possible. Go's `net/http` is production-ready.

* **Routing:** Use the standard library (Go 1.22+) or lightweight routers like `go-chi/chi`. Avoid heavy web frameworks unless strictly necessary.
* **Middleware:** Implement middleware as functions that take an `http.Handler` and return an `http.Handler`. This replaces Spring's Interceptors and Security Filter Chains.
* **Context:** **Always** pass `context.Context`. Extract it from `r.Context()` in the HTTP handler and pass it down to database calls and external API requests for timeout and cancellation management.

### 3. Data Access (Repositories)
Avoid heavy ORMs that abstract away SQL. Go developers prefer to see the queries.

* **Interfaces:** Define the repository interface where it is *used* (the Service layer), not where it is implemented.
* **Implementation:** Use `sqlx` or `database/sql` for raw, fast queries. Map results to structs using tags. 
* **Connection Pooling:** Go's `sql.DB` handles connection pooling natively. Do not open/close connections per request; pass the `*sql.DB` pool around.

### 4. Concurrency
Do not try to implement Reactive WebFlux patterns. Go is natively concurrent.

* **Goroutines:** Spin up a goroutine (`go doWork()`) for background tasks.
* **WaitGroups & ErrGroups:** Use `sync.WaitGroup` or `golang.org/x/sync/errgroup` to fan-out tasks and wait for them to complete.
* **Channels:** Use channels to safely pass data between running goroutines without sharing memory (Mutexes).

### 5. Error Handling
Go does not have `try/catch` or global Exception Handlers (`@ControllerAdvice`).

* **Return Errors:** Errors are values. Always return them: `return nil, err`.
* **Wrap Errors:** Add context when returning errors up the stack using `fmt.Errorf("failed to fetch user: %w", err)`.
* **HTTP Responses:** Centralize HTTP error responses via a helper function (e.g., `RespondWithError(w, code, message)`).

---

## Anti-Patterns to Avoid

* **Global State:** Avoid global variables for database connections or loggers. Pass them explicitly or attach them to a receiver struct (e.g., `type Handler struct { db *sql.DB }`).
* `init()` **Functions:** Avoid using `init()` to set up dependencies. It makes the code unpredictable and hard to test. Wire dependencies explicitly in `main.go`.
* **Pointer Abuse:** Don't pass pointers everywhere just to save memory. Pass small structs by value to reduce garbage collection overhead on the heap.
* **Panics:** Never use `panic` for normal error handling. `panic` is only for truly unrecoverable states (like a missing config file on startup).

---

## Reference Documents
*(See the `references/` folder for deep dives into specific topics)*
1.  `project-layout.md`
2.  `web-and-middleware.md`
3.  `data-access.md`
4.  `concurrency.md`
5.  `testing-patterns.md`