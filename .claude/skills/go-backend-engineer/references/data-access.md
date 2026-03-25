# Data Access Patterns in Go

Go's approach to data access favors explicitness, raw SQL (or lightweight query builders), and manual transaction management over heavy, reflection-based ORMs.

## Core Principles

| Concept | Idiomatic Go Approach |
|---------|----------------------|
| **Mapping** | Simple row-to-struct mapping (`db:"column_name"` tags) |
| **Querying** | Explicit SQL queries (no derived/magic query methods) |
| **Transactions** | Explicit (`tx.Begin()`, `tx.Commit()`, `tx.Rollback()`) |
| **Connection Pool** | Native `sql.DB` connection pool (built-in) |
| **Schema Mgmt** | `golang-migrate` / `goose` (raw SQL up/down scripts) |

---

## 1. Connection Pooling Setup

Go's standard `database/sql` package has connection pooling built-in. Configure the pool limits explicitly on startup. We recommend using `sqlx` as it extends standard `database/sql` with struct mapping capabilities.

```go
package database

import (
	"context"
	"fmt"
	"time"

	"github.com/jmoiron/sqlx"
	_ "github.com/lib/pq" // Postgres driver (blank import to register driver)
)

func NewPostgresDB(dsn string) (*sqlx.DB, error) {
	db, err := sqlx.Connect("postgres", dsn)
	if err != nil {
		return nil, fmt.Errorf("failed to connect to db: %w", err)
	}

	// Configure the connection pool
	db.SetMaxOpenConns(25)
	db.SetMaxIdleConns(25)
	db.SetConnMaxLifetime(5 * time.Minute)
	db.SetConnMaxIdleTime(5 * time.Minute)

	// Verify connection
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	if err := db.PingContext(ctx); err != nil {
		return nil, fmt.Errorf("failed to ping db: %w", err)
	}

	return db, nil
}
```

---

## 2. The Repository Pattern

Define the interface where it is consumed (in the service layer), and return a concrete struct from the data layer. This follows Go's "accept interfaces, return structs" principle.

**A. The Domain Model (Plain Structs)**

```go
package domain

import "time"

type User struct {
	ID        int       `db:"id"`
	Email     string    `db:"email"`
	Password  string    `db:"password_hash"`
	CreatedAt time.Time `db:"created_at"`
}
```

**B. The Implementation (Data Layer)**

```go
package postgres

import (
	"context"

	"github.com/jmoiron/sqlx"
	"myapp/internal/domain"
)

type UserRepository struct {
	db *sqlx.DB
}

func NewUserRepository(db *sqlx.DB) *UserRepository {
	return &UserRepository{db: db}
}

// Always pass context.Context as the first argument.
func (r *UserRepository) FindByID(ctx context.Context, id int) (*domain.User, error) {
	var user domain.User
	query := `SELECT id, email, password_hash, created_at FROM users WHERE id = $1`

	// sqlx GetContext maps a single row to a struct
	err := r.db.GetContext(ctx, &user, query, id)
	if err != nil {
		return nil, err // Let the caller handle sql.ErrNoRows
	}
	return &user, nil
}
```

**C. The Consumer (Service Layer)**

```go
package service

import (
	"context"

	"myapp/internal/domain"
)

// The interface is defined by the consumer (accept interfaces, return structs).
type UserReader interface {
	FindByID(ctx context.Context, id int) (*domain.User, error)
}

type UserService struct {
	repo UserReader // Inject via interface
}

func NewUserService(repo UserReader) *UserService {
	return &UserService{repo: repo}
}
```

---

## 3. Explicit Transaction Management

Go does not have AOP to inject transaction boundaries. You must manage them explicitly.

```go
func (r *UserRepository) CreateUserWithProfile(ctx context.Context, user domain.User, profile domain.Profile) error {
	tx, err := r.db.BeginTxx(ctx, nil)
	if err != nil {
		return err
	}

	// Defer rollback — safe to call even after a successful commit
	defer tx.Rollback()

	var userID int
	err = tx.QueryRowxContext(ctx, `INSERT INTO users (email) VALUES ($1) RETURNING id`, user.Email).Scan(&userID)
	if err != nil {
		return err
	}

	_, err = tx.ExecContext(ctx, `INSERT INTO profiles (user_id, bio) VALUES ($1, $2)`, userID, profile.Bio)
	if err != nil {
		return err
	}

	return tx.Commit()
}
```

---

## 4. Best Practices

1. **Context is King:** Always use the `*Context` methods (e.g., `QueryContext`, `ExecContext`, `GetContext`). This ensures that if an HTTP request is canceled, the underlying database query is also canceled.
2. **Avoid Global DB Variables:** Pass the `*sqlx.DB` instance to your repository constructors. Never use global variables like `var DB *sqlx.DB`.
3. **Handle `sql.ErrNoRows`:** Go's `QueryRow` / `Get` returns `sql.ErrNoRows` if no record is found. Check for this explicitly to differentiate between "not found" and an actual database error.
4. **Use Migrations:** Use tools like `golang-migrate` or `goose` to write raw SQL up/down migration scripts. Do not rely on ORM auto-migration.
