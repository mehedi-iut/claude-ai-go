# Go Engineering Skills

A collection of architectural guidelines, design patterns, and coding standards for building idiomatic, production-ready Go backend services.

These skills provide context for AI assistants, code reviewers, and engineers. They focus on the Go philosophy: **simplicity, explicitness, and composition**.

---

## Core Principles

1. **Explicitness over Magic:** Favor readable, traceable code. Avoid hidden control flows, magical dependency injection, or reflection-heavy abstractions.
2. **Standard Library First:** The Go standard library (`net/http`, `database/sql`, `testing`) is the foundation. Use it before reaching for third-party libraries.
3. **Errors as Values:** Handle errors explicitly where they occur and add context before returning them up the stack.
4. **Concurrency via Communication:** Share memory by communicating (channels), not by sharing memory (mutexes) — though mutexes are fine for simple state protection.

---

## Directory Structure

```text
skills/
├── README.md                           # This file
├── api-contract-review/
│   └── SKILL.md                        # REST/gRPC API design standards
├── clean-code/
│   └── SKILL.md                        # Effective Go practices, naming conventions, early returns
├── design-patterns/
│   └── SKILL.md                        # Idiomatic Go design patterns (Functional Options, Interfaces, etc.)
├── go-code-review/
│   └── SKILL.md                        # Go code review checklist (idiomatic Go, concurrency, error handling)
├── logging-and-observability/
│   └── SKILL.md                        # Structured logging (log/slog), tracing, OpenTelemetry
└── go-backend-engineer/
    ├── SKILL.md                        # Core architectural guidelines for Go backend services
    └── references/
        ├── concurrency.md              # Goroutines, channels, errgroup
        ├── data-access.md              # database/sql, sqlx, explicit transactions
        ├── project-layout.md           # Domain-driven layout vs. flat layout
        ├── testing-patterns.md         # Table-driven tests, mocking, testcontainers
        └── web-and-middleware.md        # net/http, routing, function-wrapping middleware
```

---

## How to Use

These documents are injected into LLM prompts or referenced directly during development:

- **Starting a new project:** Reference `go-backend-engineer/references/project-layout.md` to set up directories correctly.
- **Writing endpoints:** Use `web-and-middleware.md` and `data-access.md` for routing requests and database interaction.
- **Refactoring:** Consult `design-patterns/SKILL.md` for idiomatic patterns (Functional Options, Strategy via first-class functions, etc.).
- **Code quality:** Use `clean-code/SKILL.md` for naming conventions, formatting, and refactoring guidance.
- **API design:** Reference `api-contract-review/SKILL.md` for HTTP semantics, versioning, and backward compatibility.
- **Code review:** Use `go-code-review/SKILL.md` for reviewing pull requests against Go idioms and concurrency safety.
- **Observability:** Reference `logging-and-observability/SKILL.md` for structured logging with `slog`, trace ID propagation, and OpenTelemetry.
