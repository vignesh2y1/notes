# Exercises: Error Wrapping, Is, and As

Test your understanding of `%w` vs `%v`, error chains, `errors.Is`, `errors.As`, `Unwrap`, custom `Is()` methods, and wrapping strategies.

---

## 🧠 Conceptual Questions

### Question 1: Chain Traversal
Given the following error chain, predict the output of each `errors.Is` and `errors.As` call:

```go
package main

import (
	"errors"
	"fmt"
	"io"
	"os"
)

func main() {
	// Build a chain: outerErr → middleErr → io.EOF
	innerErr := io.EOF
	middleErr := fmt.Errorf("read failed: %w", innerErr)
	outerErr := fmt.Errorf("processFile: %w", middleErr)

	fmt.Println("A:", errors.Is(outerErr, io.EOF))
	fmt.Println("B:", errors.Is(outerErr, middleErr))
	fmt.Println("C:", errors.Is(middleErr, outerErr))
	fmt.Println("D:", errors.Is(outerErr, os.ErrNotExist))

	var pathErr *os.PathError
	fmt.Println("E:", errors.As(outerErr, &pathErr))
}
```

---

### Question 2: `%w` vs `%v` Consequences
A junior developer wrote the following error propagation code. Identify the bug and explain the consequence:

```go
func fetchData(url string) error {
	resp, err := http.Get(url)
	if err != nil {
		return fmt.Errorf("fetchData: %v", err) // Line A
	}
	defer resp.Body.Close()

	if resp.StatusCode != 200 {
		return fmt.Errorf("fetchData: unexpected status %d", resp.StatusCode)
	}
	return nil
}

func main() {
	err := fetchData("https://api.example.com/data")
	if err != nil {
		// The caller wants to check if the error is a timeout
		var netErr net.Error
		if errors.As(err, &netErr) && netErr.Timeout() {
			fmt.Println("Retrying after timeout...")
		} else {
			fmt.Println("Fatal error:", err)
		}
	}
}
```

---

## 🛠️ Practical Problems

### Question 3: Implement a Custom Wrapping Error Type
Create a `ServiceError` struct that:
1. Contains fields: `Operation string`, `Err error` (the wrapped inner error).
2. Implements `Error() string` returning a formatted message.
3. Implements `Unwrap() error` to expose the inner error.
4. Demonstrate that `errors.Is` can find a sentinel error wrapped inside a `ServiceError`.

```go
// Expected usage:
var ErrNotFound = errors.New("not found")

err := &ServiceError{
    Operation: "GetUser",
    Err:       ErrNotFound,
}

fmt.Println(errors.Is(err, ErrNotFound)) // Should print: true
```

---

### Question 4: Custom `Is()` Method
You have a `RetryableError` type that marks errors as retryable. You also have a sentinel `ErrRetryable` that callers check:

```go
var ErrRetryable = errors.New("retryable error")
```

Implement the `RetryableError` struct with:
1. Fields: `Err error` (the original error), `MaxRetries int`.
2. An `Error() string` method.
3. A custom `Is(target error) bool` method that returns `true` when the target is `ErrRetryable`.
4. An `Unwrap() error` method that returns the inner error.

Then demonstrate:
```go
origErr := errors.New("connection reset")
retryErr := &RetryableError{Err: origErr, MaxRetries: 3}
wrapped := fmt.Errorf("calling API: %w", retryErr)

fmt.Println(errors.Is(wrapped, ErrRetryable))    // true (via custom Is)
fmt.Println(errors.Is(wrapped, origErr))          // true (via Unwrap chain)
```

---

### Question 5: Unwrap Chain Debugging
Write a utility function `PrintErrorChain(err error)` that walks the entire error chain and prints each layer with its index:

Expected output format:
```
[0] processFile: reading data: file not found
[1] reading data: file not found
[2] file not found
```

The function should handle:
* Errors wrapped with `fmt.Errorf("...: %w", err)`
* Custom errors implementing `Unwrap() error`
* Errors with no `Unwrap` (end of chain)

---

## 💼 Scenario-Based Questions

### Question 6: API Gateway Error Classification
You are building an API gateway that proxies requests to multiple downstream services. The gateway needs to classify errors from downstream services into three categories for the client:

1. **Retryable** — network timeouts, temporary DNS failures → return `503 Service Unavailable` with `Retry-After` header
2. **Client Error** — bad request from the user → return `400 Bad Request`
3. **Fatal** — all other errors → return `500 Internal Server Error`

Design an error classification system using custom error types and `errors.As`:

1. Define `DownstreamError` struct with fields: `Service string`, `StatusCode int`, `Err error`.
2. Define `TimeoutError` struct with fields: `Service string`, `Duration time.Duration`, `Err error`. Include `Unwrap()`.
3. Write a function `classifyError(err error) int` that returns the appropriate HTTP status code by checking the error chain:
   * If chain contains `*TimeoutError` → return 503
   * If chain contains `*DownstreamError` with `StatusCode >= 400 && < 500` → return 400
   * Otherwise → return 500
4. Show how a wrapped error like `fmt.Errorf("proxy: %w", &TimeoutError{...})` is correctly classified.

---

### Question 7: The Accidental Chain Break
You are debugging a production issue where `errors.Is(err, sql.ErrNoRows)` returns `false` even though the root error IS `sql.ErrNoRows`. Your error wrapping code has been working for months. A recent PR changed one line. Find the bug:

**Before the PR (working):**
```go
func (r *UserRepo) FindByID(id int) (*User, error) {
    err := r.db.QueryRow("SELECT ...").Scan(&user)
    if err != nil {
        return nil, fmt.Errorf("FindByID(%d): %w", id, err)
    }
    return &user, nil
}
```

**After the PR (broken):**
```go
func (r *UserRepo) FindByID(id int) (*User, error) {
    err := r.db.QueryRow("SELECT ...").Scan(&user)
    if err != nil {
        return nil, fmt.Errorf("FindByID(%d): %v", id, err)
    }
    return &user, nil
}
```

1. Explain exactly what changed and why it breaks the error chain.
2. How would you prevent this type of bug in the future?
