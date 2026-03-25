# Testing Patterns in Go

Tests live directly alongside the code they test in `_test.go` files and use the standard library's `testing` package.

## Core Principles

- **No Magic:** Tests are regular Go functions starting with `Test...` that take `*testing.T`.
- **No Annotations:** Setup and teardown use plain Go code and `defer`.
- **Interfaces over Reflection:** Mocking is achieved through interface substitution, not byte-code manipulation.

---

## 1. Table-Driven Tests (The Standard)

Instead of writing a separate test function for every edge case, Go developers use "Table-Driven Tests." You define a slice of anonymous structs containing the inputs and expected outputs, then loop over them using `t.Run()`.

```go
package user

import (
	"errors"
	"testing"
)

// The function we are testing
func ValidateAge(age int) error {
	if age < 0 {
		return errors.New("age cannot be negative")
	}
	if age < 18 {
		return errors.New("user must be at least 18")
	}
	return nil
}

// The Test
func TestValidateAge(t *testing.T) {
	// 1. Define the table
	tests := []struct {
		name        string
		inputAge    int
		wantErr     bool
		expectedErr string
	}{
		{name: "Valid adult", inputAge: 25, wantErr: false},
		{name: "Exactly 18", inputAge: 18, wantErr: false},
		{name: "Underage", inputAge: 17, wantErr: true, expectedErr: "user must be at least 18"},
		{name: "Negative age", inputAge: -5, wantErr: true, expectedErr: "age cannot be negative"},
	}

	// 2. Loop through the table
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			err := ValidateAge(tt.inputAge)

			// 3. Assertions
			if (err != nil) != tt.wantErr {
				t.Fatalf("ValidateAge() error = %v, wantErr %v", err, tt.wantErr)
			}
			if err != nil && err.Error() != tt.expectedErr {
				t.Errorf("ValidateAge() expected err = %v, got %v", tt.expectedErr, err.Error())
			}
		})
	}
}
```

---

## 2. Mocking via Interfaces

Go does not use reflection-based mocking frameworks. If you need to mock a database or an external API, you should define a small interface in the *consumer* package and pass a mock implementation during testing.

### A. Manual Mocks
For small interfaces, writing a manual mock is often the cleanest approach.

```go
package service

import "testing"

// 1. Define the dependency as an interface
type EmailSender interface {
	Send(to string, body string) error
}

// 2. The struct being tested
type NotificationService struct {
	sender EmailSender
}

// 3. The Manual Mock
type MockEmailSender struct {
	SendFunc func(to string, body string) error
	Sent     int // track how many times it was called
}

func (m *MockEmailSender) Send(to string, body string) error {
	m.Sent++
	if m.SendFunc != nil {
		return m.SendFunc(to, body)
	}
	return nil
}

// 4. The Test
func TestNotificationService(t *testing.T) {
	mockSender := &MockEmailSender{
		SendFunc: func(to string, body string) error {
			// Simulate a successful send
			return nil
		},
	}

	svc := NotificationService{sender: mockSender}
	err := svc.NotifyUser("test@example.com", "Hello")

	if err != nil {
		t.Errorf("Expected no error, got %v", err)
	}
	if mockSender.Sent != 1 {
		t.Errorf("Expected email to be sent exactly once")
	}
}
```

### B. Generated Mocks
For larger interfaces, use a code generator like `go.uber.org/mock/mockgen` or `github.com/matryer/moq`.
- **Moq** generates simple, predictable structs similar to the manual mock above.
- **Mockgen** creates robust mocks with strict call ordering and argument matching.

---

## 3. Integration Testing with Testcontainers

For testing data-access layers (repositories), mocking the database often provides false confidence. Go natively supports spinning up Docker containers for true integration tests using [Testcontainers for Go](https://golang.testcontainers.org/).

Use Build Tags (`//go:build integration`) to separate slow integration tests from fast unit tests.

```go
//go:build integration

package postgres_test

import (
	"context"
	"testing"
	"github.com/testcontainers/testcontainers-go"
	"github.com/testcontainers/testcontainers-go/modules/postgres"
)

func TestUserRepositoryIntegration(t *testing.T) {
	ctx := context.Background()

	// 1. Spin up a real Postgres container
	pgContainer, err := postgres.RunContainer(ctx,
		testcontainers.WithImage("postgres:15-alpine"),
		postgres.WithDatabase("testdb"),
		postgres.WithUsername("testuser"),
		postgres.WithPassword("testpass"),
	)
	if err != nil {
		t.Fatalf("failed to start container: %s", err)
	}

	// 2. Ensure container is cleaned up after the test
	t.Cleanup(func() {
		if err := pgContainer.Terminate(ctx); err != nil {
			t.Fatalf("failed to terminate container: %s", err)
		}
	})

	// 3. Get connection string and run tests
	connStr, _ := pgContainer.ConnectionString(ctx, "sslmode=disable")
	
	// -> Setup your database connection using connStr
	// -> Run your migrations (e.g., using golang-migrate)
	// -> Run your repository tests
}
```

---

## 4. Test Execution & Best Practices

- **Run all tests:** `go test ./...`
- **Run with verbosity:** `go test -v ./...`
- **Race detector (always enable in CI):** `go test -race ./...`
- **Run specific tests:** `go test -run TestValidateAge ./...`
- **Run integration tests:** `go test -tags=integration ./...`
- **Coverage:** `go test -coverprofile=coverage.out ./...` then `go tool cover -html=coverage.out`
- **Use `t.Helper()`:** Call `t.Helper()` inside helper functions so failures report the caller's line number, not the helper's.
- **Use `t.Parallel()`:** Call `t.Parallel()` in subtests to run table-driven test cases concurrently.
