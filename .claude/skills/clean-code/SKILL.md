---
name: clean-code
description: Clean Code principles (DRY, KISS, YAGNI), idiomatic Go naming conventions, function design, and refactoring. Use when user says "clean this code", "refactor", "improve readability", or when reviewing Go code quality.
---

# Clean Code Skill (Go Edition)

Write readable, maintainable, and idiomatic Go code following Clean Code principles and the "Go Way".

## When to Use
- User says "clean this code" / "refactor" / "make it idiomatic"
- Code review focusing on maintainability and simplicity
- Reducing complexity and eliminating "magic"
- Improving naming (Go has strict, unique naming philosophies)

---

## Core Principles

| Principle | Meaning | Violation Sign |
|-----------|---------|----------------|
| **DRY** | Don't Repeat Yourself | Copy-pasted code blocks, repetitive error handling |
| **KISS** | Keep It Simple, Stupid | Over-engineered interfaces, unnecessary generics |
| **YAGNI** | You Aren't Gonna Need It | Massive interfaces, features "just in case" |

---

## DRY - Don't Repeat Yourself

> "Every piece of knowledge must have a single, unambiguous representation in the system."

### Violation

```go
// ❌ BAD: Same validation and error formatting repeated
func CreateUser(req UserRequest) error {
    if req.Email == "" {
        return fmt.Errorf("validation failed: email is required")
    }
    if !strings.Contains(req.Email, "@") {
        return fmt.Errorf("validation failed: invalid email format")
    }
    // ... create user
    return nil
}

func UpdateUser(req UserRequest) error {
    if req.Email == "" {
        return fmt.Errorf("validation failed: email is required")
    }
    if !strings.Contains(req.Email, "@") {
        return fmt.Errorf("validation failed: invalid email format")
    }
    // ... update user
    return nil
}
```

### Refactored

```go
// ✅ GOOD: Single source of truth with idiomatic custom errors
var ErrInvalidEmail = errors.New("invalid email format")
var ErrEmailRequired = errors.New("email is required")

func validateEmail(email string) error {
    if email == "" {
        return ErrEmailRequired
    }
    if !strings.Contains(email, "@") {
        return ErrInvalidEmail
    }
    return nil
}

func CreateUser(req UserRequest) error {
    if err := validateEmail(req.Email); err != nil {
        return fmt.Errorf("create user validation: %w", err)
    }
    // ... create user
    return nil
}
```

---

## KISS - Keep It Simple

> "Clear is better than clever. Reflection and interfaces are not always the answer."

### Violation

```go
// ❌ BAD: Over-abstracted and generic for a simple task
func IsEmpty[T any](val T) bool {
    v := reflect.ValueOf(val)
    switch v.Kind() {
    case reflect.String:
        return strings.TrimSpace(v.String()) == ""
    case reflect.Slice, reflect.Map:
        return v.Len() == 0
    }
    return false
}
```

### Refactored

```go
// ✅ GOOD: Simple, type-safe, and clear
func IsStringEmpty(s string) bool {
    return strings.TrimSpace(s) == ""
}
// For slices, just use len(slice) == 0 directly in the caller!
```

### KISS Checklist
- Is this code hiding control flow? (Avoid "magic")
- Am I using `reflect` or `any` where static types would work?
- Am I using channels/goroutines when a simple synchronous function is faster and safer?

---

## YAGNI - You Aren't Gonna Need It

> "The bigger the interface, the weaker the abstraction." - Rob Pike

### Violation

```go
// ❌ BAD: Building a massive Java-style interface "just in case"
type UserRepository interface {
    FindByID(ctx context.Context, id uuid.UUID) (*User, error)
    FindAll(ctx context.Context) ([]*User, error)
    FindAllPaginated(ctx context.Context, limit, offset int) ([]*User, error)
    Save(ctx context.Context, u *User) error
    Delete(ctx context.Context, id uuid.UUID) error
    Count(ctx context.Context) (int64, error)
    // ... 15 more methods
}
```

### Refactored

```go
// ✅ GOOD: Small, consumer-defined interfaces
// Define the interface exactly where it is used, requiring only what is needed.
type UserCreator interface {
    Save(ctx context.Context, u *User) error
}

func RegisterUser(ctx context.Context, db UserCreator, u *User) error {
    return db.Save(ctx, u)
}
```

---

## Idiomatic Go Naming Conventions

### Variables

Go prefers short, concise variable names for small scopes, and descriptive names for package-level variables.

```go
// ❌ BAD
var userList []User       // Stuttering (type in name)
var theError error        // Unidiomatic
var d int                 // Too short if used globally

// ✅ GOOD
var users []User          // Plural
var err error             // Standard idiom
var activeConnections int // Descriptive for large scope

// ✅ GOOD (Short in small scopes)
for i, u := range users { 
    fmt.Println(u.Name)
}
```

### Getters and Setters

Go does NOT use "Get" for getters.

```go
// ❌ BAD
func (u *User) GetEmail() string { return u.email }

// ✅ GOOD
func (u *User) Email() string { return u.email }
func (u *User) SetEmail(e string) { u.email = e } // Setters keep "Set"
```

### Interfaces

Interfaces with one method usually end in `-er`.

```go
// ❌ BAD
type WriteInterface interface { Write(p []byte) (n int, err error) }
type UserActions interface { Save() error }

// ✅ GOOD
type Writer interface { Write(p []byte) (n int, err error) }
type Saver interface { Save() error }
```

### Naming Conventions Table

| Element | Convention | Example |
|---------|------------|---------|
| Exported Type/Func| PascalCase | `type Order struct{}`, `func NewOrder()` |
| Unexported Type/Func| camelCase | `type order struct{}`, `func calculate()`|
| Interface | PascalCase, `-er` suffix | `Reader`, `Formatter` |
| Package | short, single word, lowercase| `http`, `postgres` (not `http_utils`)|
| Receiver | 1-2 letters of type | `func (s *Server) Start()` |

---

## Functions / Methods

### Limit Parameters (Functional Options Pattern)

```go
// ❌ BAD: Too many parameters
func NewServer(host string, port int, maxConn int, timeout time.Duration, tls bool) *Server {
    // ...
}

// ✅ GOOD: Config struct
func NewServer(cfg ServerConfig) *Server {
    // ...
}

// ✅ BEST: Functional Options (Idiomatic Go for libraries)
func NewServer(host string, port int, opts ...ServerOption) *Server {
    s := &Server{host: host, port: port, timeout: defaultTimeout}
    for _, opt := range opts {
        opt(s)
    }
    return s
}
```

### Avoid Boolean Flag Arguments

```go
// ❌ BAD: Boolean flag changes behavior
func ProcessPayment(amount float64, useCredit bool) error {
    if useCredit {
        // ...
    }
}

// ✅ GOOD: Separate functions
func ProcessCreditPayment(amount float64) error { /* ... */ }
func ProcessCashPayment(amount float64) error { /* ... */ }
```

---

## Comments

### Let Code Speak

```go
// ❌ BAD: Noise comments
// Check if err is not nil
if err != nil {
    return err
}

// ❌ BAD: Explaining bad code
// Check if state is 1 (active) or 2 (pending)
if state == 1 || state == 2 { }

// ✅ GOOD: Self-documenting code
const (
    StateActive  = 1
    StatePending = 2
)
if state == StateActive || state == StatePending { }
```

### Idiomatic GoDoc Comments
Comments on exported identifiers must start with the name of the identifier.

```go
// ✅ GOOD
// ValidateEmail checks if the provided string is a valid RFC 5322 email address.
func ValidateEmail(email string) error { ... }
```

---

## Common Go Code Smells

| Smell | Description | Refactoring |
|-------|-------------|-------------|
| **Nesting** | "Arrow Code" (deeply nested ifs) | Guard Clauses / Early Returns |
| **Ignored Errors** | `_, _ = doSomething()` | Handle every `err` explicitly |
| **Package `util`** | Dumping ground package | Name packages by what they *provide* |
| **Pointer Abuse** | `*[]*User` | Pass slices by value, structs by pointer |
| **Goroutine Leaks**| Starting a goroutine with no exit | Pass Context, use WaitGroups |
| **Primitive Obsession**| `string` for everything | Custom Types |

### Primitive Obsession

```go
// ❌ BAD: Primitives everywhere, easily swapped
func SendMoney(fromID string, toID string, amount float64) error { }
SendMoney(toID, fromID, 100) // Compiles, but catastrophic logic bug!

// ✅ GOOD: Custom types ensure type safety at compile time
type AccountID string
type Money float64

func SendMoney(from AccountID, to AccountID, amount Money) error { }
// SendMoney(to, from, amount) -> Compiler error!
```

---

## Refactoring Quick Reference: Guard Clauses (Line of Sight)

Keep the "happy path" aligned to the left edge of the screen. Return early on errors.

```go
// ❌ BAD: Deeply nested happy path
func ProcessFile(filename string) error {
    file, err := os.Open(filename)
    if err == nil {
        defer file.Close()
        data, err := io.ReadAll(file)
        if err == nil {
            if len(data) > 0 {
                return parse(data)
            } else {
                return fmt.Errorf("file empty")
            }
        } else {
            return err
        }
    }
    return err
}

// ✅ GOOD: Guard clauses (Idiomatic Go)
func ProcessFile(filename string) error {
    file, err := os.Open(filename)
    if err != nil {
        return err
    }
    defer file.Close()

    data, err := io.ReadAll(file)
    if err != nil {
        return err
    }

    if len(data) == 0 {
        return fmt.Errorf("file empty")
    }

    return parse(data) // Happy path at the bottom, un-indented
}
```

---

## Go Clean Code Checklist

When reviewing code, check:
- [ ] Is error handling explicit? (`if err != nil` checked immediately)
- [ ] Are errors wrapped with context? (`fmt.Errorf("failed to open: %w", err)`)
- [ ] Is the happy path left-aligned? (Guard clauses used)
- [ ] Are variable names appropriately sized for their scope?
- [ ] Are interfaces small and defined where they are used?
- [ ] Is concurrency (goroutines/channels) actually necessary, or just clever?
- [ ] Is the context (`ctx context.Context`) passed as the first parameter to blocking functions?
```