---
name: design-patterns
description: Common design patterns with idiomatic Go examples (Functional Options, Factory, Strategy, Observer, Decorator, etc.). Use when user asks "implement pattern", "use functional options", "strategy pattern", or when designing extensible Go packages.
---

# Go Design Patterns Skill

When transitioning from Java/Spring to Go, the biggest shift is moving away from heavy class hierarchies and towards Go's core philosophies: **simplicity, composition, and implicit interfaces**. Go often handles these patterns more concisely using first-class functions and structural embedding.

## When to Use
- User asks to implement a specific pattern
- Designing extensible/flexible packages
- Refactoring rigid code or minimizing dependencies

## Quick Reference: When to Use What

| Problem | Go Pattern | Use When |
|---------|------------|----------|
| Complex object construction | **Functional Options** | Many parameters, most are optional or have defaults |
| Create objects without specifying concrete type | **Factory Function** | Struct initialization needs encapsulation or validation |
| Multiple algorithms, swap at runtime | **Strategy** | Behavior varies by context (using interfaces or first-class functions) |
| Add behavior without changing struct | **Decorator / Middleware** | Dynamic composition needed (e.g., HTTP middleware) |
| Notify multiple components of changes | **Observer** | One-to-many dependency (often using channels or callbacks) |
| Convert incompatible interfaces | **Adapter** | Integrate legacy/3rd party code via implicit interfaces |

---

## Creational Patterns

### Functional Options (Go's alternative to the Builder pattern)

**Problem:** Structs with many optional parameters. Go doesn't have constructors or method overloading.

```go
// ✅ Functional Options pattern (Idiomatic Go)
package server

import "time"

type Server struct {
	host    string
	port    int
	timeout time.Duration // optional
}

// ServerOption is a function that modifies a Server
type ServerOption func(*Server)

// WithTimeout is a functional option
func WithTimeout(t time.Duration) ServerOption {
	return func(s *Server) {
		s.timeout = t
	}
}

// NewServer acts as the constructor
func NewServer(host string, port int, opts ...ServerOption) *Server {
	// Set defaults
	s := &Server{
		host:    host,
		port:    port,
		timeout: 30 * time.Second,
	}

	// Apply options
	for _, opt := range opts {
		opt(s)
	}
	return s
}

// Usage
srv := NewServer("localhost", 8080, WithTimeout(60*time.Second))
```

### Factory

**Problem:** Create implementations of an interface without exposing the concrete structs.

```go
// ✅ Factory pattern
package notification

import "fmt"

type Sender interface {
	Send(message string) error
}

// Concrete implementations (unexported to force use of factory)
type emailSender struct{}
func (e *emailSender) Send(msg string) error { return nil }

type smsSender struct{}
func (s *smsSender) Send(msg string) error { return nil }

// NewSender is a simple factory function
func NewSender(senderType string) (Sender, error) {
	switch senderType {
	case "email":
		return &emailSender{}, nil
	case "sms":
		return &smsSender{}, nil
	default:
		return nil, fmt.Errorf("unknown sender type: %s", senderType)
	}
}

// Registry version (useful for dependency injection style)
type Registry struct {
	senders map[string]Sender
}

func NewRegistry() *Registry {
	return &Registry{senders: make(map[string]Sender)}
}

func (r *Registry) Register(name string, s Sender) {
	r.senders[name] = s
}

func (r *Registry) Get(name string) (Sender, error) {
	if sender, exists := r.senders[name]; exists {
		return sender, nil
	}
	return nil, fmt.Errorf("sender %s not found", name)
}
```

---

## Behavioral Patterns

### Strategy

**Problem:** Multiple algorithms for the same operation, choose at runtime.

```go
// ✅ Strategy pattern via Interfaces
package payment

import "fmt"

type Strategy interface {
	Pay(amount float64) error
}

type CreditCard struct {
	Number string
}

func (c *CreditCard) Pay(amount float64) error {
	fmt.Printf("Paid %.2f using Credit Card\n", amount)
	return nil
}

type ShoppingCart struct {
	strategy Strategy
}

func (s *ShoppingCart) SetStrategy(strategy Strategy) {
	s.strategy = strategy
}

func (s *ShoppingCart) Checkout(amount float64) error {
	return s.strategy.Pay(amount)
}

// ✅ Strategy pattern via First-Class Functions (Highly idiomatic)
type PayFunc func(amount float64) error

func CreditCardFunc(number string) PayFunc {
	return func(amount float64) error {
		fmt.Printf("Paid %.2f using card %s\n", amount, number)
		return nil
	}
}

func Checkout(amount float64, pay PayFunc) error {
	return pay(amount)
}

// Usage
Checkout(99.99, CreditCardFunc("4111..."))
```

### Observer

**Problem:** Notify multiple objects when state changes. In Go, this is often done with channels or simple callbacks.

```go
// ✅ Observer pattern
package event

import "sync"

type Event struct {
	Data string
}

type Observer interface {
	OnNotify(Event)
}

type EventBus struct {
	mu        sync.RWMutex
	observers []Observer
}

func (b *EventBus) Subscribe(o Observer) {
	b.mu.Lock()
	defer b.mu.Unlock()
	b.observers = append(b.observers, o)
}

func (b *EventBus) Publish(e Event) {
	b.mu.RLock()
	defer b.mu.RUnlock()
	for _, o := range b.observers {
		// Can be wrapped in a goroutine for async: go o.OnNotify(e)
		o.OnNotify(e)
	}
}

// Usage
type Logger struct{}
func (l *Logger) OnNotify(e Event) { /* log event */ }

bus := &EventBus{}
bus.Subscribe(&Logger{})
bus.Publish(Event{Data: "OrderPlaced"})
```

---

## Structural Patterns

### Decorator (Middleware)

**Problem:** Add behavior dynamically. Go uses interface wrapping extensively for this (e.g., `io.Reader` or `http.Handler`).

```go
// ✅ Decorator pattern
package coffee

type Coffee interface {
	Cost() float64
	Description() string
}

type SimpleCoffee struct{}

func (s *SimpleCoffee) Cost() float64       { return 2.0 }
func (s *SimpleCoffee) Description() string { return "Simple Coffee" }

// Milk decorator
type Milk struct {
	coffee Coffee // Embeds the interface
}

func (m *Milk) Cost() float64 {
	return m.coffee.Cost() + 0.50
}

func (m *Milk) Description() string {
	return m.coffee.Description() + ", Milk"
}

// Usage
var c Coffee = &SimpleCoffee{}
c = &Milk{coffee: c}
```

### Adapter

**Problem:** Make incompatible interfaces work together. Go's implicit interfaces make this extremely seamless.

```go
// ✅ Adapter pattern
package player

// Target interface
type MediaPlayer interface {
	Play(filename string)
}

// Legacy struct (doesn't implement MediaPlayer)
type LegacyAudioPlayer struct{}

func (l *LegacyAudioPlayer) PlayMP3(filename string) { /* ... */ }

// Adapter
type MP3Adapter struct {
	legacy *LegacyAudioPlayer
}

// Implicitly implements MediaPlayer
func (a *MP3Adapter) Play(filename string) {
	a.legacy.PlayMP3(filename)
}

// Usage
var player MediaPlayer = &MP3Adapter{legacy: &LegacyAudioPlayer{}}
player.Play("song.mp3")
```

---

## Pattern Selection Guide

| Situation | Go Pattern |
|-----------|------------|
| Struct has >3 parameters or many defaults | Functional Options |
| Need to hide concrete implementation | Factory / Dependency Injection |
| Need to add features dynamically | Interface Wrapping (Decorator) |
| Multiple implementations of algorithm | Interface Strategy or Func Strategy |
| React to state changes asynchronously | Channel-based Observer / Event Bus |
| Integrate with legacy code | Struct Adapter |

## Anti-Patterns to Avoid in Go

| Anti-Pattern | Problem | Better Approach |
|--------------|---------|-----------------|
| `init()` func abuse | Global state, unpredictable execution order, hard to test | Explicit initialization, Dependency Injection |
| Deep Inheritance | Go isn't OOP; deep embedding causes confusion | Shallow Composition (embed only what you need) |
| Interface Pollution | Creating interfaces before they are needed | Define interfaces where they are *used*, not where they are implemented |
| Heavy Frameworks | Fights Go's standard library (e.g., net/http) | Standard library + lightweight routing/middleware |
