# Concurrency Patterns in Go

Go was designed from the ground up for high concurrency. Rather than relying on heavy OS threads and complex lock-based memory sharing, Go uses **Goroutines** (lightweight execution threads) and **Channels** (typed conduits for safely passing data). 

The guiding principle of Go concurrency is:
> *"Do not communicate by sharing memory; instead, share memory by communicating."*

Here are the standard patterns used in production Go backend services.

---

## 1. The Scatter-Gather Pattern (Fetching Data Concurrently)

When an API endpoint needs to fetch data from multiple independent sources (e.g., a database and two external APIs), doing it sequentially wastes time. You want to scatter the requests and gather the results.

While you can use `sync.WaitGroup`, the standard for backend services is `golang.org/x/sync/errgroup`. It handles synchronization, error propagation, and context cancellation automatically.

```go
package service

import (
	"context"
	"golang.org/x/sync/errgroup"
)

type DashboardData struct {
	User   *User
	Orders []Order
	Stats  *Stats
}

func (s *DashboardService) GetDashboard(ctx context.Context, userID string) (*DashboardData, error) {
	// Create an errgroup with a derived context that cancels if any goroutine fails
	g, gCtx := errgroup.WithContext(ctx)

	var data DashboardData

	// Goroutine 1: Fetch User
	g.Go(func() error {
		user, err := s.userRepo.FindByID(gCtx, userID)
		if err != nil {
			return err
		}
		data.User = user
		return nil
	})

	// Goroutine 2: Fetch Orders
	g.Go(func() error {
		orders, err := s.orderRepo.FindByUserID(gCtx, userID)
		if err != nil {
			return err
		}
		data.Orders = orders
		return nil
	})

	// Wait for all goroutines to finish. 
	// If any g.Go() returns an error, g.Wait() returns that error and cancels gCtx.
	if err := g.Wait(); err != nil {
		return nil, err
	}

	return &data, nil
}
```

---

## 2. Safe Background Tasks (Fire and Forget)

Sometimes you want to return an HTTP response immediately but continue processing in the background (e.g., sending a welcome email after user registration).

**The Trap:** If you simply use `go sendEmail(r.Context())`, the HTTP request context will be canceled as soon as the handler returns, killing your background task mid-flight.

**The Solution:** Create a new context decoupled from the HTTP request, but respect the application's overall lifecycle.

```go
func (h *UserHandler) RegisterUser(w http.ResponseWriter, r *http.Request) {
	// ... user creation logic ...

	// GOOD: Create a fresh context for the background task
	// (Optionally with its own timeout)
	bgCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	
	go func() {
		defer cancel() // Prevent context leak
		
		// This runs safely in the background
		err := h.emailService.SendWelcomeEmail(bgCtx, user.Email)
		if err != nil {
			log.Printf("failed to send welcome email: %v", err)
		}
	}()

	// Return response immediately
	w.WriteHeader(http.StatusCreated)
}
```

---

## 3. Worker Pools (Limiting Concurrency)

If you need to process 10,000 images, spawning 10,000 goroutines might overwhelm your database, network connections, or memory. A Worker Pool limits the number of active goroutines.

```go
package worker

import (
	"context"
	"fmt"
	"sync"
)

// ProcessJobs takes a slice of job IDs and processes them using a fixed number of workers.
func ProcessJobs(ctx context.Context, jobIDs []int, numWorkers int) error {
	jobsCh := make(chan int, len(jobIDs))
	resultsCh := make(chan error, len(jobIDs))

	// 1. Load the jobs into the channel
	for _, id := range jobIDs {
		jobsCh <- id
	}
	close(jobsCh) // Close the channel so workers know when to stop

	// 2. Start the workers
	var wg sync.WaitGroup
	for i := 0; i < numWorkers; i++ {
		wg.Add(1)
		go func(workerID int) {
			defer wg.Done()
			
			// Workers pull from the shared channel until it's closed
			for jobID := range jobsCh {
				// Check for context cancellation before processing
				if ctx.Err() != nil {
					resultsCh <- ctx.Err()
					return
				}
				
				err := processSingleJob(jobID) // Your business logic here
				resultsCh <- err
			}
		}(i)
	}

	// 3. Wait for workers to finish and close the results channel
	go func() {
		wg.Wait()
		close(resultsCh)
	}()

	// 4. Collect results/errors
	for err := range resultsCh {
		if err != nil {
			return fmt.Errorf("job failed: %w", err)
		}
	}

	return nil
}

func processSingleJob(id int) error {
	// simulate work
	return nil
}
```

---

## 4. Mutexes (When Not to Use Channels)

While channels are great for orchestrating flow and passing data, sometimes you just need to protect a shared map or struct variable from concurrent reads/writes. In these cases, a standard `sync.RWMutex` is perfectly idiomatic and often faster.

```go
type InMemCache struct {
	mu    sync.RWMutex
	store map[string]string
}

func (c *InMemCache) Set(key, value string) {
	c.mu.Lock()         // Exclusive lock for writing
	defer c.mu.Unlock()
	c.store[key] = value
}

func (c *InMemCache) Get(key string) (string, bool) {
	c.mu.RLock()        // Shared lock for reading
	defer c.mu.RUnlock()
	val, ok := c.store[key]
	return val, ok
}
```

---

## 5. Critical Guardrails

1. **Never Start a Goroutine Without Knowing How It Stops:** This is the #1 cause of memory leaks in Go. If a goroutine is listening on a channel, ensure that channel is eventually closed or the context is canceled.
2. **Race Detector:** Always run your tests with the race detector enabled: `go test -race ./...`. It will catch 99% of shared memory concurrency bugs.
3. **Don't Overuse Concurrency:** If a task is CPU-bound and fast, adding goroutines and channels might actually slow it down due to context-switching overhead. Use concurrency for I/O bound tasks (network, database, disk).
