---
name: go-code-review
description: Comprehensive checklist and guidelines for reviewing Go code. Use when user asks to "review this Go code," "check my pull request," or "find bugs in this Go file." Focuses on idiomatic Go, concurrency traps, error handling, and performance.
---

# Go Code Review Skill

This guide outlines the critical areas to focus on when reviewing Go code. A good Go code review goes beyond business logic; it ensures the code is safe, concurrent, and respects Go's explicit nature.

## 0. Prerequisite: The Automated Baseline
Before spending human or AI time on a review, ensure the code passes standard Go tooling. **Do not nitpick formatting in code reviews.**
- [ ] Code is formatted with `gofmt` or `goimports`.
- [ ] Code passes `go vet`.
- [ ] Code passes a standard linter setup (e.g., `golangci-lint`).

---

## 1. Error Handling (The Go Way)
Go relies on explicit error checking. Ignoring errors or handling them poorly is the #1 cause of flaky Go applications.

- [ ] **No swallowed errors:** Are errors being ignored with the blank identifier (`_ = err`)? This is almost always a red flag.
- [ ] **Contextual wrapping:** Are errors wrapped with context before being returned? (e.g., `fmt.Errorf("failed to fetch user %s: %w", id, err)` instead of just `return err`).
- [ ] **No Panics:** Does the code use `panic`? Panics should *only* be used for unrecoverable initialization errors (like a missing config file on startup). Standard business logic must return `error`.
- [ ] **Error type checking:** If the caller needs to inspect the error, does the code use `errors.Is` or `errors.As` instead of string matching (`err.Error() == "not found"`)?

## 2. Concurrency and Goroutines
Concurrency is easy to write in Go but hard to get right. Look closely for leaks and race conditions.

- [ ] **Goroutine leaks:** Does every spawned `go func()` have a clear exit condition? If it listens on a channel, is that channel guaranteed to be closed? If it blocks on I/O, does it respect a `context.Context` timeout?
- [ ] **WaitGroups:** If `sync.WaitGroup` is used, is `wg.Add()` called *outside* the goroutine, and `wg.Done()` called via `defer` *inside* the goroutine?
- [ ] **Race Conditions:** Are variables being mutated by multiple goroutines without a `sync.Mutex` or without being passed through a channel? (Remind the author to run tests with `-race`).
- [ ] **Channel sizes:** Are buffered channels being used prematurely as a performance optimization? Unbuffered channels should be the default for synchronization.

## 3. Context Usage
`context.Context` is the standard way to handle timeouts, cancellation, and request-scoped values in Go.

- [ ] **First Argument:** Is `ctx context.Context` the first parameter of any function that does I/O (database, network, file system)?
- [ ] **Never in Structs:** Is `context.Context` stored inside a struct? Contexts should flow through function arguments, not be held in state.
- [ ] **Value usage:** Is `context.WithValue` being abused to pass optional parameters or dependencies? It should only be used for request-scoped data (like trace IDs or authenticated User IDs).

## 4. Pointers, Memory, and Slices
Migrating developers often use pointers everywhere, thinking it's faster. In Go, passing by value is often more efficient due to stack allocation and less Garbage Collection (GC) pressure.

- [ ] **Pointer abuse:** Are pointers used for small structs just to avoid copying? Prefer passing small structs by value.
- [ ] **Maps and Slices:** Is the author passing a pointer to a slice (`*[]string`) or a map (`*map[string]int`)? Slices and maps are already reference types; they almost never need to be passed as pointers.
- [ ] **Slice pre-allocation:** If the final size of a slice is known, is it pre-allocated? (e.g., `make([]int, 0, len(items))` instead of `var res []int`). This prevents unnecessary memory reallocations.

## 5. Interfaces and Design
Go interfaces are implicit, meaning you don't need to declare `implements`.

- [ ] **Accept Interfaces, Return Structs:** Is the code returning an interface when it could return a concrete struct? Functions should generally return concrete types and accept interfaces.
- [ ] **Define where used:** Are interfaces defined in the package that *implements* them (Java style)? In Go, interfaces should be defined in the package that *consumes* them.
- [ ] **Interface size:** Are interfaces small (1-3 methods)? `io.Reader` and `io.Writer` are the gold standards. Avoid monolithic interfaces.

---

## Code Review Anti-Patterns Quick Reference

| Smell / Anti-Pattern | Why it's bad in Go | Better Approach |
|----------------------|--------------------|-----------------|
| `init()` functions | Causes hidden state changes, makes testing hard, unpredictable execution order. | Explicit initialization in `main.go`. |
| Naked Returns | e.g., `func split(sum int) (x, y int) { ... return }`. Hard to read in long functions. | Explicitly return variables `return x, y`. |
| Deep nesting (Else) | Makes code hard to follow. Go prefers "line of sight" coding. | Handle errors and edge cases early, return early. Drop the `else`. |
| Global Variables | E.g., global database connections or loggers. Causes race conditions and breaks tests. | Inject dependencies via constructors (`NewService(db)`). |
| Magic Numbers | Hardcoded values scattered in logic. | Define as `const` at the top of the file/package. |