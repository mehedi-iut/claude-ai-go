---
name: logging-and-observability
description: Guidelines for structured logging, metrics, and distributed tracing in Go. Use when a user asks about "how to log errors," "setting up slog," "passing trace IDs," or "instrumenting a Go application."
---

# Go Logging and Observability Skill

In Go, observability is explicit. You must pass `context.Context` down the call stack to achieve distributed tracing and contextual logging.

## 1. Structured Logging (`log/slog`)

As of Go 1.21, the standard library includes `log/slog` for high-performance, structured logging. Use `slog` over the legacy `log` package. (`zap` is acceptable for extreme high-performance edge cases, but `slog` is the community standard.)

### Setting Up the Logger (Once in `main.go`)

```go
package main

import (
	"log/slog"
	"os"
)

func main() {
	// Use JSON format for production (parseable by Datadog/ELK/CloudWatch)
	logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
		Level: slog.LevelInfo,
	}))

	slog.SetDefault(logger)

	slog.Info("application started", slog.String("env", "production"), slog.Int("port", 8080))
}
```

### Logging in Business Logic

Avoid string formatting (`fmt.Sprintf`) inside log messages. Pass attributes as key-value pairs so log aggregators can index them.

```go
// Anti-pattern: unstructured string concatenation
slog.Info(fmt.Sprintf("User %s failed to login due to %v", userID, err))

// Idiomatic: structured attributes
slog.Warn("user login failed",
    slog.String("user_id", userID),
    slog.String("reason", err.Error()),
)
```

---

## 2. Context Propagation for Trace IDs

Go uses goroutines, which have no thread-local storage. Trace IDs must be passed explicitly via `context.Context`.

### Extracting and Logging Trace IDs

Create a helper to extract trace IDs from context and attach them to the logger.

```go
package obs

import (
	"context"
	"log/slog"
)

type contextKey string

const traceIDKey contextKey = "trace_id"

// InjectTraceID adds a trace ID to the context (typically called in HTTP middleware).
func InjectTraceID(ctx context.Context, traceID string) context.Context {
	return context.WithValue(ctx, traceIDKey, traceID)
}

// Logger returns a logger with the trace ID attached, if present in the context.
func Logger(ctx context.Context) *slog.Logger {
	logger := slog.Default()
	if traceID, ok := ctx.Value(traceIDKey).(string); ok {
		return logger.With(slog.String("trace_id", traceID))
	}
	return logger
}
```

**Usage in a service:**

```go
func (s *UserService) CreateUser(ctx context.Context, email string) error {
	log := obs.Logger(ctx)

	log.Info("attempting to create user", slog.String("email", email))

	err := s.repo.Insert(ctx, email)
	if err != nil {
		log.Error("failed to insert user", slog.String("error", err.Error()))
		return err
	}

	return nil
}
```

---

## 3. Distributed Tracing and Metrics (OpenTelemetry)

Use **OpenTelemetry (OTel)** for distributed tracing and metrics. Pass `context.Context` to outbound HTTP calls and database queries so OTel can attach `span_id` and `trace_id`.

```go
import (
	"context"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/attribute"
)

func (s *UserService) FetchProfile(ctx context.Context, userID string) (*Profile, error) {
	tracer := otel.Tracer("user-service")
	ctx, span := tracer.Start(ctx, "FetchProfile")
	defer span.End()

	span.SetAttributes(attribute.String("user.id", userID))

	profile, err := s.repo.GetProfile(ctx, userID)
	if err != nil {
		span.RecordError(err)
		return nil, err
	}

	return profile, nil
}
```

---

## 4. Anti-Patterns to Avoid

| Anti-Pattern | Problem | Better Approach |
|--------------|---------|-----------------|
| `log.Fatal()` deep in code | Kills the app abruptly, bypasses `defer` and graceful shutdown | Return an `error` up the stack. Only call `log.Fatal` in `main.go` |
| Logging errors without returning | `if err != nil { log.Error(err) }` then continuing | Log AND return the error, or handle the fallback properly |
| Global loggers with state | `slog.With("user", id)` on a shared logger leaks data across concurrent requests | Create a per-request logger instance, or extract state from `context.Context` |
| Heavy logging in tight loops | Causes I/O bottlenecks and GC pressure from allocations | Log aggregations, use debug levels, or sample logs |
