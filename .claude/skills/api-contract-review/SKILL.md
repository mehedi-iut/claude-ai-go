---
name: api-contract-review
description: Review REST API contracts for HTTP semantics, versioning, backward compatibility, and response consistency. Use when user asks "review API", "check endpoints", "REST review", or before releasing API changes.
---

# API Contract Review Skill (Go 1.22+ Edition)

Audit REST API design for correctness, consistency, backward compatibility, and idiomatic Go payload handling.

## When to Use
- User asks "review this API" / "check REST endpoints"
- Before releasing API changes or updating OpenAPI/Swagger specs
- Reviewing a PR with HTTP handler (`net/http`) changes
- Checking backward compatibility

---

## Quick Reference: Common Issues

| Issue | Symptom | Impact |
|-------|---------|--------|
| Wrong HTTP verb | POST for idempotent operation | Confusion, caching issues |
| Missing versioning | `/users` instead of `/v1/users` | Breaking changes affect all clients |
| Entity leak | Database struct (`sqlc` or `gorm`) in JSON response | Exposes internals, password hashes |
| 200 with error | `w.WriteHeader(200)` + `{"error": "..."}` | Breaks client-side error handling |
| Inconsistent naming | `GET /getUsers` vs `GET /users` | Hard to learn API, violates REST |

---

## HTTP Verb Semantics (Go 1.22+ Routing)

### Verb Selection Guide

| Verb | Use For | Idempotent | Safe | Request Body |
|------|---------|------------|------|--------------|
| GET | Retrieve resource | Yes | Yes | No |
| POST | Create new resource | No | No | Yes |
| PUT | Replace entire resource | Yes | No | Yes |
| PATCH | Partial update | No* | No | Yes |
| DELETE | Remove resource | Yes | No | Optional |

### Common Mistakes

```go
// ❌ POST for retrieval
mux.HandleFunc("POST /users/search", searchUsersHandler) 

// ✅ GET with query params (r.URL.Query())
mux.HandleFunc("GET /users", searchUsersHandler)

// ❌ GET for state change
mux.HandleFunc("GET /users/{id}/activate", activateUserHandler)

// ✅ POST or PATCH for state change
mux.HandleFunc("POST /users/{id}/activate", activateUserHandler)

// ❌ POST for idempotent update
mux.HandleFunc("POST /users/{id}", updateUserHandler)

// ✅ PUT for full replacement, PATCH for partial
mux.HandleFunc("PUT /users/{id}", replaceUserHandler)
mux.HandleFunc("PATCH /users/{id}", patchUserHandler)
```

---

## API Versioning

### Strategies

| Strategy | Example | Pros | Cons |
|----------|---------|------|------|
| URL path | `/v1/users` | Clear, easy routing | URL changes |
| Header | `Accept: application/vnd.api.v1+json` | Clean URLs | Hidden, harder to test |
| Query param | `/users?version=1` | Easy to add | Easy to forget |

### Recommended: URL Path

```go
// ✅ Versioned endpoints grouped via Go 1.22 mux
v1 := http.NewServeMux()
v1.HandleFunc("GET /users", v1GetUsers)
mux.Handle("/api/v1/", http.StripPrefix("/api/v1", v1))

// ❌ No versioning
mux.HandleFunc("GET /api/users", getUsers) // Breaking changes affect everyone
```

---

## Request/Response Design

### DTO (Payload) vs Entity (DB Model)

```go
// ❌ Entity in response (leaks internals)
type User struct { // DB Model
    ID           int    `json:"id"`
    PasswordHash string `json:"password_hash"` // 🚨 Leaks hash!
    InternalFlag bool   `json:"internal_flag"`
}

func getUser(w http.ResponseWriter, r *http.Request) {
    user := db.GetUser(r.PathValue("id"))
    json.NewEncoder(w).Encode(user) 
}

// ✅ DTO response mapping
type UserResponse struct {
    ID    int    `json:"id"`
    Email string `json:"email"`
}

func getUser(w http.ResponseWriter, r *http.Request) {
    user := db.GetUser(r.PathValue("id"))
    resp := UserResponse{ID: user.ID, Email: user.Email}
    json.NewEncoder(w).Encode(resp)
}
```

---

## HTTP Status Codes & Error Handling

### Anti-Pattern: 200 OK with Error Body

```go
// ❌ NEVER DO THIS: Returning 200 OK for an error
func getUser(w http.ResponseWriter, r *http.Request) {
    user, err := db.GetUser(id)
    if err != nil {
        w.WriteHeader(http.StatusOK) // 🚨 Still 200!
        json.NewEncoder(w).Encode(map[string]string{"status": "error", "msg": "not found"})
        return
    }
}

// ✅ Proper status codes and centralized error responses
func getUser(w http.ResponseWriter, r *http.Request) {
    user, err := db.GetUser(id)
    if err != nil {
        writeError(w, http.StatusNotFound, "USER_NOT_FOUND", "User does not exist")
        return
    }
    json.NewEncoder(w).Encode(user)
}
```

### Consistent Error Structure

```go
// ✅ Standard error response struct
type ErrorResponse struct {
    Code      string `json:"code"`       // Machine-readable: "USER_NOT_FOUND"
    Message   string `json:"message"`    // Human-readable: "User 123 not found"
    Timestamp string `json:"timestamp"`  // ISO 8601
    Path      string `json:"path"`
}

func writeError(w http.ResponseWriter, status int, code, msg string) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(ErrorResponse{
        Code:      code,
        Message:   msg,
        Timestamp: time.Now().UTC().Format(time.RFC3339),
    })
}
```

---

## Backward Compatibility

### Breaking Changes (Avoid in Same Version)
| Change | Breaking? | Migration |
|--------|-----------|-----------|
| Remove endpoint | Yes | Deprecate first, remove in next version |
| Remove field from JSON response | Yes | Keep field, return zero-value/empty |
| Add required field to request JSON | Yes | Make optional using Go pointers (`*string`) |
| Change JSON field type | Yes | Add new field, deprecate old |
| Rename JSON field (struct tag) | Yes | Support both temporarily |

---

## Token Optimization (For the AI Agent)

For large Go APIs, do not read every file. Use the shell to scan for anti-patterns:

1. List all handler files: `find internal/handler -name "*.go"`
2. Check `ServeMux` configuration once.
3. Grep for specific anti-patterns:
   ```bash
   # Find unversioned APIs (Looking for standard HandleFunc without v1/v2)
   grep -rnw . -e 'HandleFunc(".* /api/' | grep -v "/v[0-9]/"

   # Find potential 200 OK error leaks (Writing StatusOK inside an error block)
   grep -rnw . -B 2 -A 2 -e 'w.WriteHeader(http.StatusOK)' | grep 'err !='

   # Find direct JSON encoding of DB models (assuming db models are in a /models or /db pkg)
   grep -rnw . -e 'json.NewEncoder(w).Encode(' | grep -E 'db\.|models\.'
   ```
```