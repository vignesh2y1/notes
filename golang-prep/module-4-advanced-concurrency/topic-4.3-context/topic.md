# Topic 4.3: The Context Package

The `context` package is Go's standard mechanism for managing request lifecycles, propagating cancellation signals, handling deadlines/timeouts, and sharing request-scoped values across API boundaries and goroutine trees.

This topic covers the `Context` interface, context hierarchy propagation, value injection best practices, and memory leak prevention.

---

## 1. Concepts & Theory

### 1.1 The `Context` Interface
The `context.Context` is an immutable interface containing four methods:

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key any) any
}
```

1. **`Deadline()`:** Returns the time when work on behalf of this context should be canceled (if configured).
2. **`Done()`:** Returns a read-only channel. This channel is closed when the context is canceled (either manually, via timeout, or by its parent context).
3. **`Err()`:** Returns the reason *why* the context was canceled. It returns `context.Canceled` (manual cancel) or `context.DeadlineExceeded` (timeout). If `Done` is not yet closed, it returns `nil`.
4. **`Value(key any)`:** Returns the value associated with this context for a given key.

---

### 1.2 The Context Tree & Signal Propagation
Context objects form a **directed acyclic tree** where each child context holds a reference to its parent.

```
          [context.Background]  (Root)
                   |
            [context.WithCancel]  (Child 1)
                   |
         [context.WithTimeout]  (Grandchild)
```

* **Downward Propagation:** When a parent context is canceled, the Go runtime automatically propagates the cancellation signal downward to all of its children, closing their `Done` channels.
* **No Upward Propagation:** Canceling a child context has no effect on its parent or siblings.

---

### 1.3 Context Varieties

#### 1. Empty Contexts (`Background` vs `TODO`)
* `context.Background()`: The root of all context trees. Used in main functions, tests, and initialization.
* `context.TODO()`: Used as a placeholder when you are unsure which context to use, or when the calling function has not yet been refactored to accept context.

#### 2. Cancellation Contexts (`WithCancel` / `WithTimeout` / `WithDeadline`)
* `WithCancel(parent)`: Returns a copy of parent with a new `Done` channel and a `cancel` function.
* `WithTimeout(parent, duration)` / `WithDeadline(parent, time)`: Automatically calls the cancel function when the duration/deadline is reached.

#### 3. Value Contexts (`WithValue`)
* `WithValue(parent, key, val)`: Returns a copy of parent that associates `key` with `val`.
* **Search Mechanics:** When `ctx.Value(key)` is invoked, it checks the current context node. If the key doesn't match, it traverses **upward** towards the parent node recursively until it finds the key or hits the root `Background` context (returning `nil`).

---

## 2. Edge Cases & Limitations

### 2.1 Context Leaks (The Missing Cancel)
When you create a child context with a timeout or deadline:
```go
ctx, cancel := context.WithTimeout(parent, 5*time.Second)
```
The Go runtime schedules a internal timer in the background. If the operation finishes in 1 second, but you **do not call `cancel()`**, the timer remains scheduled in memory for the remaining 4 seconds. The child context remains linked to the parent context, preventing garbage collection of the child context and any allocated resources.
* **Rule:** Always call the returned `cancel()` function, typically immediately via `defer cancel()`.

### 2.2 `WithValue` Abuse
Passing application parameters through context values is a major anti-pattern.
* **Bad Practice:** Storing database connection pools, logger instances, or optional function arguments in context values. This hides dependency declarations, breaks type-safety, and makes code difficult to maintain.
* **Good Practice:** Only use context values for **request-scoped data** that does not impact basic control flow:
  - Correlation IDs (Trace ID)
  - Authentication tokens / user scopes
  - Telemetry and tracing headers

---

## 3. Real-World Usage & Best Practices

### 3.1 Custom Context Keys
To prevent key collisions between different third-party packages sharing the same context, **never use raw strings or basic types** as keys. Always declare a custom, unexported struct type for your keys:

```go
package auth

import "context"

// Unexported custom type prevents key collisions
type contextKey struct{}

var userKey = contextKey{}

// Helper to inject value type-safely
func WithUser(ctx context.Context, userID string) context.Context {
	return context.WithValue(ctx, userKey, userID)
}

// Helper to extract value type-safely
func UserFromContext(ctx context.Context) (string, bool) {
	val, ok := ctx.Value(userKey).(string)
	return val, ok
}
```

---

## 4. Code Examples: Good vs. Bad

### ❌ Bad: Context Leak & Raw String Keys
This code fails to cancel the timeout context, causing a timer leak. It also uses a raw string key, which risk namespace clashes.

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func processRequest(ctx context.Context) {
	// BAD: raw string key
	userID := ctx.Value("userID") 
	fmt.Println("Processing user:", userID)
}

func handle() {
	ctx := context.WithValue(context.Background(), "userID", "user-123")
	
	// BAD: cancel is never called, leaking the timer memory until 10 minutes pass
	ctxWithTimeout, _ := context.WithTimeout(ctx, 10*time.Minute) 
	
	processRequest(ctxWithTimeout)
}
```

### Good: Deferred Cancel & Type-Safe Context Keys
This approach uses a custom struct key to guarantee isolation and uses `defer cancel()` to release the timer resources immediately once the function returns.

```go
package main

import (
	"context"
	"fmt"
	"time"
)

type userCtxKey struct{} // Custom unexported type

var userIDKey = userCtxKey{}

func WithUserID(ctx context.Context, userID string) context.Context {
	return context.WithValue(ctx, userIDKey, userID)
}

func UserIDFromContext(ctx context.Context) (string, bool) {
	val, ok := ctx.Value(userIDKey).(string)
	return val, ok
}

func processRequest(ctx context.Context) {
	if userID, ok := UserIDFromContext(ctx); ok {
		fmt.Println("Processing user:", userID)
	}
}

func handle() {
	ctx := WithUserID(context.Background(), "user-123")

	// Create timeout context
	ctxWithTimeout, cancel := context.WithTimeout(ctx, 10*time.Minute)
	// Good: Always defer cancel to release timer allocations immediately
	defer cancel() 

	processRequest(ctxWithTimeout)
}
```

### Good: Propagating Context to SQL Queries
Most database drivers support context-aware queries. If a client disconnects or times out, the database driver cancels the SQL execution instantly, saving database resources.

```go
package main

import (
	"context"
	"database/sql"
	"time"
)

type UserRepository struct {
	db *sql.DB
}

func (r *UserRepository) GetUserProfile(ctx context.Context, userID string) (string, error) {
	// Create a query timeout context
	queryCtx, cancel := context.WithTimeout(ctx, 2*time.Second)
	defer cancel()

	var profile string
	// Pass queryCtx to standard library database call
	err := r.db.QueryRowContext(queryCtx, "SELECT profile FROM users WHERE id = $1", userID).Scan(&profile)
	if err != nil {
		return "", err // If queryCtx times out, this returns sql.ErrTxDone or query cancelled error
	}
	return profile, nil
}
```
