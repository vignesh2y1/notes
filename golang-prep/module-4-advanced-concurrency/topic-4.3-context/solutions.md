# Solutions: The Context Package

Here are the step-by-step solutions, code implementations, and architectural fixes for the Topic 4.3 exercises.

---

## 🧠 Conceptual Questions

### Solution 1: Context Value Search Internals

#### 1. Tree Representation:
```
           [context.Background] (Root)
                    ^
                    | (parent)
          [valueCtx: keyA -> "valA"] (ctx1)
                    ^
                    | (parent)
          [valueCtx: keyB -> "valB"] (ctx2)
                    ^
                    | (parent)
          [valueCtx: keyC -> "valC"] (ctx3)
```

#### 2. Search Algorithm & Complexity:
* When `ctx3.Value(keyA)` is invoked:
  1. It checks if the key inside `ctx3` matches `keyA`. It doesn't (`keyC != keyA`).
  2. It follows the parent pointer to `ctx2` and checks its key. It doesn't (`keyB != keyA`).
  3. It follows the parent pointer to `ctx1` and checks its key. It matches (`keyA == keyA`). It returns `"valA"`.
* **Complexity:** This is a sequential linked-list traversal upwards. The time complexity is $O(N)$ where $N$ is the depth of the context tree.

#### 3. Why a Linked List instead of a Map?
* **Immutability and Thread Safety:** Contexts must be safe for simultaneous use by multiple goroutines. If they shared a mutable hash map, writes would require locking, causing contention.
* **Allocation Overhead:** Most contexts contain zero or only one value. Allocating a complete hash map for a single value is extremely heavy. A small struct containing key, value, and parent pointer is lightweight, and the lookup list is short in normal applications (depth < 10), making $O(N)$ lookup faster than map hashing overhead.

---

### Solution 2: Anatomy of a Context Leak

#### 1. Leaked Resources:
* When `WithTimeout` is called, Go registers an OS timer via the runtime's timer heap to trigger the cancellation after `duration`.
* If `cancel()` is not called and the operation completes early, the timer remains scheduled in the runtime scheduler. The child context remains linked to the parent context (via internal child registration lists). Any values, channels, or pointers inside the child context cannot be garbage collected.

#### 2. GC Cleanup behavior:
* If the parent context (like a long-running app-level context) remains active, the GC **cannot** clean up the child context. The parent holds pointers to the child, which keeps the child context alive in memory until the timeout duration expires.

#### 3. Role of `cancel()`:
* The `cancel()` function:
  1. Removes the child context from the parent context's internal registry.
  2. Stops the background runtime timer immediately, removing it from the timer heap.
  3. Closes the context's `Done` channel, releasing any waiting goroutines.

---

## 🛠️ Practical Problems

### Solution 3: Cancellable Worker

We use a `select` statement inside the loop. Before reading an item, or during processing, we verify if the context's `Done` channel is closed.

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"time"
)

func ProcessItems(ctx context.Context, items <-chan string) error {
	for {
		select {
		case <-ctx.Done():
			// Context canceled or timed out; exit immediately
			return ctx.Err()
		case item, ok := <-items:
			if !ok {
				// Channel drained and closed
				return nil
			}
			// Simulate item processing
			fmt.Println("Processing:", item)
			time.Sleep(100 * time.Millisecond)
		}
	}
}
```

---

### Solution 4: Type-Safe Request Context Middleware

By using an unexported type `contextKey` and public helpers, we hide the key namespace, making key collisions impossible.

```go
package requestinfo

import "context"

// Unexported struct prevents key collisions
type contextKey struct{}

var requestIDKey = contextKey{}

// WithRequestID injects the Request ID into the context
func WithRequestID(ctx context.Context, reqID string) context.Context {
	return context.WithValue(ctx, requestIDKey, reqID)
}

// RequestIDFromContext extracts the Request ID from the context
func RequestIDFromContext(ctx context.Context) (string, bool) {
	val, ok := ctx.Value(requestIDKey).(string)
	return val, ok
}
```

---

## 💼 Scenario-Based Questions

### Solution 5: The "Zombie SQL Query" Database Leak

#### 1. Why SQL Queries keep running:
* If you execute a query using standard database calls like `db.Query(...)` or `db.Exec(...)`, they do not accept a context.
* When the client disconnects and the request context is canceled, the Go HTTP handler yields, but the network socket state does not propagate to the database driver. The database driver continues waiting for the database server to finish, and the database server keeps running the query, wasting CPU cycles.

#### 2. Refactoring using Context-Aware Queries:
* Change all SQL calls to use context-aware variants, such as `QueryContext`, `ExecContext`, or `QueryRowContext`, and pass the request context.
* Under the hood, if the context is canceled, the Go database driver detects this, intercepts the connection, sends an interrupt signal to PostgreSQL (canceling the query execution on the DB side), and closes the connection cleanly.

```go
// Corrected database query execution
func getUserData(ctx context.Context, db *sql.DB, id int) (string, error) {
	var name string
	// Database query will immediately stop if context is canceled
	err := db.QueryRowContext(ctx, "SELECT name FROM users WHERE id = $1", id).Scan(&name)
	return name, err
}
```

---

### Solution 6: Spawning Detached Background Tasks

#### 1. The Bug:
* Once `handleUserRegistration` completes, the HTTP router cancels `r.Context()`.
* The spawned goroutine executes `sendAnalytics` after a 3-second sleep. However, by that time, `r.Context()` is **already canceled**.
* Any context-aware actions inside `sendAnalytics` (like HTTP requests to analytics servers or DB saves) will fail immediately with `context canceled`.

#### 2. What `sendAnalytics` receives:
* It receives a canceled context where `ctx.Err() == context.Canceled`.

#### 3. How to Detach Context Safely:
We must create a **new context** that inherits request-scoped value fields (like Trace ID) but does not inherit the parent context's cancellation channel. We implement a custom "detached context" wrapper:

```go
package main

import (
	"context"
	"net/http"
	"time"
)

// DetachedContext wraps a parent context to inherit values,
// but overrides the Done and Deadline channels to prevent cancellation.
type DetachedContext struct {
	context.Context
}

func (d DetachedContext) Done() <-chan struct{} { return nil }
func (d DetachedContext) Deadline() (time.Time, bool) { return time.Time{}, false }
func (d DetachedContext) Err() error { return nil }

func handleUserRegistration(w http.ResponseWriter, r *http.Request) {
	// 1. Process registration...
	
	// Create a detached context carrying request values
	detachedCtx := DetachedContext{Context: r.Context()}

	// 2. Spawn async task safely
	go func() {
		// Timeout for the background task itself (independent of HTTP lifecycle)
		bgCtx, cancel := context.WithTimeout(detachedCtx, 5*time.Second)
		defer cancel()
		
		time.Sleep(3 * time.Second)
		sendAnalytics(bgCtx, "user_registered") // Success!
	}()

	w.Write([]byte("Registration successful"))
}

func sendAnalytics(ctx context.Context, event string) {}
```
* Note: Alternatively, in Go 1.21+, you can use `context.WithoutCancel(r.Context())` which is built directly into the standard library! It is the recommended modern approach.
