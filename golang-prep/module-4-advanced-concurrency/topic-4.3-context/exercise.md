# Exercises: The Context Package

Test your understanding of Go's context propagation trees, cancellation mechanics, request-scoped value safety, resource leakages, and context-detaching patterns.

---

## 🧠 Conceptual Questions

### Question 1: Context Value Search Internals
A developer constructs a deeply nested context chain:
```go
ctx := context.Background()
ctx1 := context.WithValue(ctx, keyA, "valA")
ctx2 := context.WithValue(ctx1, keyB, "valB")
ctx3 := context.WithValue(ctx2, keyC, "valC")
```
1. Draw the tree representing this chain.
2. When you execute `ctx3.Value(keyA)`, explain the search algorithm. What is the time complexity in relation to the tree depth?
3. Why did Go's designers use a linked list node structure for values instead of wrapping a standard Go `map` inside the context?

---

### Question 2: Anatomy of a Context Leak
Explain:
1. What resources are leaked when `context.WithTimeout` is called, but the returned `cancel()` function is never executed?
2. Does the garbage collector (GC) clean up the timer if the parent context remains active?
3. What is the exact role of the `cancel()` function in cleanups?

---

## 🛠️ Practical Problems

### Question 3: Cancellable Worker
Write a Go function `func ProcessItems(ctx context.Context, items <-chan string) error` that:
1. Processes items from the channel sequentially.
2. If `ctx` is canceled (or times out) at any point, the function must stop processing immediately and return the context's error (`ctx.Err()`).
3. If the channel is closed and empty, return `nil`.

---

### Question 4: Type-Safe Request Context Middleware
Implement a middleware pattern:
1. Define a package `requestinfo`.
2. Declare a custom context key type.
3. Write `WithRequestID(ctx context.Context, reqID string) context.Context`.
4. Write `RequestIDFromContext(ctx context.Context) (string, bool)`.
5. Ensure that no other package can access or modify this value under the same key without calling your public functions.

---

## 💼 Scenario-Based Questions

### Question 5: The "Zombie SQL Query" Database Leak
You run a high-traffic web server. When users click "Cancel" on their web browsers, or when the connection drops, your HTTP server's middleware cancels the request context.
However, your DBA reports that your PostgreSQL database CPU is running at 100% due to long-running analytical queries.

1. Why are these database queries still running even though the HTTP client disconnected and the request context was canceled?
2. How do you refactor your database query code to ensure that PostgreSQL immediately halts execution of a query when a client cancels their request?

---

### Question 6: Spawning Detached Background Tasks
A developer wants to write a log tracker in an HTTP handler. They write:

```go
func handleUserRegistration(w http.ResponseWriter, r *http.Request) {
	// 1. Process registration...
	registerUserInDB(r.Context(), r.FormValue("username"))

	// 2. Spawn async background analytics task
	go func() {
		// Simulates 3-second background logging
		time.Sleep(3 * time.Second)
		sendAnalytics(r.Context(), "user_registered") // BAD: Uses request context
	}()

	w.Write([]byte("Registration successful"))
}
```

1. Explain the critical bug in this code. What happens to `r.Context()` once `handleUserRegistration` writes the response header and returns?
2. What will `sendAnalytics` receive when it evaluates the context?
3. How do you fix this code so that the background task can run asynchronously, but still carry request variables (like Trace ID) without being prematurely canceled by the HTTP lifecycle?
