# Web and Middleware Patterns in Go

Go's `net/http` package provides a powerful foundation for building web services. Handlers are explicit functions, JSON encoding/decoding is manual, and middleware uses a simple function-wrapping pattern.

## Core Concepts

| Concept | Idiomatic Go (`net/http`) |
|---------|---------------------------|
| **Handlers** | `http.HandlerFunc` or structs with handler methods |
| **Path Variables** | `r.PathValue("id")` (Go 1.22+ `ServeMux`) |
| **Request Body** | `json.NewDecoder(r.Body).Decode(&dto)` |
| **Response Body** | `json.NewEncoder(w).Encode(obj)` |
| **Middleware** | `func(http.Handler) http.Handler` (decorator pattern) |
| **Request Context** | `r.Context()` (passed explicitly to downstream calls) |

---

## 1. Handlers and Routing

As of Go 1.22, the standard library `http.ServeMux` supports HTTP methods and path variables natively, reducing the need for third-party routers.

### A. The Handler Struct

Use a struct to hold dependencies and attach handler functions as methods.

```go
package web

import (
	"encoding/json"
	"net/http"

	"myapp/internal/domain"
)

type UserHandler struct {
	userService domain.UserService
}

func NewUserHandler(us domain.UserService) *UserHandler {
	return &UserHandler{userService: us}
}

func (h *UserHandler) HandleGetUser(w http.ResponseWriter, r *http.Request) {
	id := r.PathValue("id")
	if id == "" {
		http.Error(w, "missing user id", http.StatusBadRequest)
		return
	}

	user, err := h.userService.FindByID(r.Context(), id)
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	if err := json.NewEncoder(w).Encode(user); err != nil {
		http.Error(w, "failed to encode response", http.StatusInternalServerError)
	}
}
```

### B. Request Body Parsing

Go requires explicit JSON decoding into a struct.

```go
func (h *UserHandler) HandleCreateUser(w http.ResponseWriter, r *http.Request) {
	var req domain.CreateUserRequest

	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		http.Error(w, "invalid request payload", http.StatusBadRequest)
		return
	}
	defer r.Body.Close()

	// Validate manually or via go-playground/validator
	if req.Email == "" {
		http.Error(w, "email is required", http.StatusBadRequest)
		return
	}

	// Process...
	w.WriteHeader(http.StatusCreated)
}
```

---

## 2. Middleware

Go uses the **decorator pattern** for middleware. A middleware takes an `http.Handler` and returns a new `http.Handler` that wraps the original.

### A. Logging Middleware

```go
func LoggingMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()

		next.ServeHTTP(w, r)

		log.Printf("%s %s took %v", r.Method, r.URL.Path, time.Since(start))
	})
}
```

### B. Authentication Middleware

Pass data (like an authenticated user ID) from middleware to handlers using `context.WithValue`. Use a custom type for context keys to avoid collisions.

```go
type contextKey string

const userIDKey contextKey = "userID"

func AuthMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		token := r.Header.Get("Authorization")
		if token == "" {
			http.Error(w, "Unauthorized", http.StatusUnauthorized)
			return
		}

		userID, err := validateJWT(token)
		if err != nil {
			http.Error(w, "Forbidden", http.StatusForbidden)
			return
		}

		ctx := context.WithValue(r.Context(), userIDKey, userID)
		next.ServeHTTP(w, r.WithContext(ctx))
	})
}

// UserIDFromContext extracts the user ID in handlers.
func UserIDFromContext(ctx context.Context) (string, bool) {
	id, ok := ctx.Value(userIDKey).(string)
	return id, ok
}
```

---

## 3. Wiring It All Together (main.go)

Wire dependencies and routes explicitly in `main.go`.

```go
func main() {
	// Setup dependencies
	db := database.NewPostgresDB("postgres://...")
	userRepo := postgres.NewUserRepository(db)
	userService := service.NewUserService(userRepo)
	userHandler := web.NewUserHandler(userService)

	// Setup router (Go 1.22+)
	mux := http.NewServeMux()
	mux.HandleFunc("GET /users/{id}", userHandler.HandleGetUser)
	mux.HandleFunc("POST /users", userHandler.HandleCreateUser)

	// Wrap with middleware (order matters: outermost runs first)
	handler := LoggingMiddleware(AuthMiddleware(mux))

	// Start server with explicit timeouts
	server := &http.Server{
		Addr:         ":8080",
		Handler:      handler,
		ReadTimeout:  5 * time.Second,
		WriteTimeout: 10 * time.Second,
		IdleTimeout:  120 * time.Second,
	}

	log.Println("Server starting on :8080")
	if err := server.ListenAndServe(); err != nil {
		log.Fatalf("Server failed: %v", err)
	}
}
```

---

## 4. Best Practices

1. **Centralize Error Responses:** Create helpers like `RespondJSON(w, status, payload)` and `RespondError(w, status, message)` to avoid repeating JSON encoding boilerplate.
2. **Timeouts are Mandatory:** Always configure `ReadTimeout`, `WriteTimeout`, and `IdleTimeout` on `http.Server`. Default Go servers allow infinite timeouts, leading to resource exhaustion in production.
3. **Pass Context Everywhere:** Pass the request's context (`r.Context()`) to every database or external API call. This ensures client disconnects propagate cancellation to in-flight operations.
4. **Graceful Shutdown:** Start the server in a goroutine and listen for `os.Interrupt`/`SIGTERM` signals to drain connections cleanly before exiting.
