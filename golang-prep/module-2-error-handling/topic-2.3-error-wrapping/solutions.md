# Solutions: Error Wrapping, Is, and As

Below are the detailed solutions for all Topic 2.3 exercises.

---

## 🧠 Conceptual Questions

### Solution 1: Chain Traversal

#### Output:
```
A: true
B: true
C: false
D: false
E: false
```

#### Explanation:

**Chain structure:** `outerErr` → `middleErr` → `io.EOF`

* **A: `errors.Is(outerErr, io.EOF)` → `true`**
  `errors.Is` walks the chain: `outerErr` ≠ `io.EOF` → unwrap → `middleErr` ≠ `io.EOF` → unwrap → `io.EOF` == `io.EOF` ✅

* **B: `errors.Is(outerErr, middleErr)` → `true`**
  Walk chain: `outerErr` ≠ `middleErr` → unwrap → `middleErr` == `middleErr` ✅

* **C: `errors.Is(middleErr, outerErr)` → `false`**
  Walk chain: `middleErr` ≠ `outerErr` → unwrap → `io.EOF` ≠ `outerErr` → unwrap → `nil`. `outerErr` wraps `middleErr`, not the other way around. **The chain is directional** — you can only search downward (inner), never upward (outer).

* **D: `errors.Is(outerErr, os.ErrNotExist)` → `false`**
  None of the errors in the chain (`outerErr`, `middleErr`, `io.EOF`) match `os.ErrNotExist`.

* **E: `errors.As(outerErr, &pathErr)` → `false`**
  None of the errors in the chain are of type `*os.PathError`. The chain contains only `*fmt.wrapError` wrappers and `io.EOF` (which is an `*errors.errorString`).

---

### Solution 2: `%w` vs `%v` Consequences

#### The Bug:
On **Line A**, the developer used `%v` instead of `%w`:
```go
return fmt.Errorf("fetchData: %v", err) // %v converts err to a string
```

`%v` calls `err.Error()` and embeds the **string representation** of the error, not the error itself. The original error value (which may be a `*net.OpError` wrapping a `net.Error` with timeout info) is discarded.

#### The Consequence:
In the caller, `errors.As(err, &netErr)` traverses the error chain looking for a `net.Error`. But the chain only contains a `*fmt.wrapError` wrapping a plain string — the `net.Error` interface information was lost when `%v` stringified it. So `errors.As` returns `false`, and the timeout retry logic **never triggers**.

#### The Fix:
```go
return fmt.Errorf("fetchData: %w", err) // Preserve the error chain
```

---

## 🛠️ Practical Problems

### Solution 3: Custom Wrapping Error Type

```go
package main

import (
	"errors"
	"fmt"
)

// Sentinel
var ErrNotFound = errors.New("not found")

// Custom wrapping error
type ServiceError struct {
	Operation string
	Err       error // the wrapped inner error
}

func (e *ServiceError) Error() string {
	return fmt.Sprintf("service error in %s: %v", e.Operation, e.Err)
}

func (e *ServiceError) Unwrap() error {
	return e.Err // exposes inner error for chain traversal
}

func main() {
	err := &ServiceError{
		Operation: "GetUser",
		Err:       ErrNotFound,
	}

	// errors.Is walks the chain:
	// ServiceError → Unwrap() → ErrNotFound == ErrNotFound ✅
	fmt.Println(errors.Is(err, ErrNotFound)) // true

	// Also works when further wrapped:
	wrapped := fmt.Errorf("handler: %w", err)
	fmt.Println(errors.Is(wrapped, ErrNotFound)) // true

	// errors.As can extract the ServiceError:
	var svcErr *ServiceError
	if errors.As(wrapped, &svcErr) {
		fmt.Println("Operation:", svcErr.Operation) // "GetUser"
	}
}
```

#### How It Works:
1. `ServiceError` implements `Unwrap()`, returning `Err`.
2. When `errors.Is(err, ErrNotFound)` is called, it checks `err` (the `ServiceError`) — no match. Then it calls `Unwrap()` → gets `ErrNotFound` → match! Returns `true`.
3. Further wrapping with `fmt.Errorf("...: %w", err)` adds another layer, but `errors.Is` and `errors.As` traverse through all layers.

---

### Solution 4: Custom `Is()` Method

```go
package main

import (
	"errors"
	"fmt"
)

var ErrRetryable = errors.New("retryable error")

type RetryableError struct {
	Err        error
	MaxRetries int
}

func (e *RetryableError) Error() string {
	return fmt.Sprintf("retryable (max %d): %v", e.MaxRetries, e.Err)
}

// Custom Is: makes this error match the ErrRetryable sentinel
func (e *RetryableError) Is(target error) bool {
	return target == ErrRetryable
}

// Unwrap exposes the inner error
func (e *RetryableError) Unwrap() error {
	return e.Err
}

func main() {
	origErr := errors.New("connection reset")
	retryErr := &RetryableError{Err: origErr, MaxRetries: 3}
	wrapped := fmt.Errorf("calling API: %w", retryErr)

	// errors.Is walks: wrapped → RetryableError.Is(ErrRetryable) → true
	fmt.Println(errors.Is(wrapped, ErrRetryable)) // true

	// errors.Is walks: wrapped → RetryableError ≠ origErr → Unwrap → origErr == origErr
	fmt.Println(errors.Is(wrapped, origErr)) // true

	// errors.As extracts RetryableError metadata
	var rErr *RetryableError
	if errors.As(wrapped, &rErr) {
		fmt.Printf("Retry up to %d times for: %v\n", rErr.MaxRetries, rErr.Err)
	}
}
```

#### How the Custom `Is` Works:
When `errors.Is` checks the `RetryableError`, it first does a pointer comparison (`retryErr == ErrRetryable` → false). Then it checks if `retryErr` has a custom `Is()` method. It does — and `e.Is(ErrRetryable)` returns `true` because `target == ErrRetryable`. This allows a **type** to match a **sentinel** — a powerful pattern for error classification.

---

### Solution 5: Unwrap Chain Debugging

```go
package main

import (
	"errors"
	"fmt"
)

func PrintErrorChain(err error) {
	for i := 0; err != nil; i++ {
		fmt.Printf("[%d] %v\n", i, err)
		err = errors.Unwrap(err)
	}
}

func main() {
	// Build a chain
	root := errors.New("file not found")
	mid := fmt.Errorf("reading data: %w", root)
	outer := fmt.Errorf("processFile: %w", mid)

	PrintErrorChain(outer)
}
```

#### Output:
```
[0] processFile: reading data: file not found
[1] reading data: file not found
[2] file not found
```

#### How It Works:
1. Start with the outermost error. Print it at index 0.
2. Call `errors.Unwrap(err)` to get the next inner error.
3. If `Unwrap` returns `nil`, the chain ends.
4. Otherwise, increment the index and repeat.

`errors.Unwrap` calls the error's `Unwrap()` method if it exists. If the error doesn't implement `Unwrap`, it returns `nil` (end of chain).

---

## 💼 Scenario-Based Questions

### Solution 6: API Gateway Error Classification

```go
package main

import (
	"errors"
	"fmt"
	"time"
)

// ─── Custom Error Types ──────────────────────────────────

type DownstreamError struct {
	Service    string
	StatusCode int
	Err        error
}

func (e *DownstreamError) Error() string {
	return fmt.Sprintf("downstream %s returned %d: %v", e.Service, e.StatusCode, e.Err)
}

func (e *DownstreamError) Unwrap() error {
	return e.Err
}

type TimeoutError struct {
	Service  string
	Duration time.Duration
	Err      error
}

func (e *TimeoutError) Error() string {
	return fmt.Sprintf("timeout calling %s after %v: %v", e.Service, e.Duration, e.Err)
}

func (e *TimeoutError) Unwrap() error {
	return e.Err
}

// ─── Error Classifier ────────────────────────────────────

func classifyError(err error) int {
	// Check for timeout first (most specific)
	var timeoutErr *TimeoutError
	if errors.As(err, &timeoutErr) {
		return 503 // Service Unavailable — retryable
	}

	// Check for downstream error with client-side status
	var downErr *DownstreamError
	if errors.As(err, &downErr) {
		if downErr.StatusCode >= 400 && downErr.StatusCode < 500 {
			return 400 // Client error passthrough
		}
	}

	// Everything else is a fatal internal error
	return 500
}

// ─── Demonstration ───────────────────────────────────────

func main() {
	// Scenario 1: Timeout error (wrapped)
	err1 := fmt.Errorf("proxy: %w", &TimeoutError{
		Service:  "user-service",
		Duration: 5 * time.Second,
		Err:      errors.New("context deadline exceeded"),
	})
	fmt.Printf("Timeout → HTTP %d\n", classifyError(err1)) // 503

	// Scenario 2: Downstream 422 (client error)
	err2 := fmt.Errorf("proxy: %w", &DownstreamError{
		Service:    "payment-service",
		StatusCode: 422,
		Err:        errors.New("invalid card number"),
	})
	fmt.Printf("Client error → HTTP %d\n", classifyError(err2)) // 400

	// Scenario 3: Unknown error
	err3 := errors.New("unexpected EOF")
	fmt.Printf("Unknown → HTTP %d\n", classifyError(err3)) // 500
}
```

#### Output:
```
Timeout → HTTP 503
Client error → HTTP 400
Unknown → HTTP 500
```

The `classifyError` function uses `errors.As` to search the chain for specific error types, extracts the structured metadata, and makes branching decisions. This approach is:
* **Composable** — new error types can be added without modifying existing classification logic.
* **Chain-safe** — works even when errors are wrapped in multiple layers of `fmt.Errorf("...: %w", ...)`.

---

### Solution 7: The Accidental Chain Break

#### 1. What Changed:
The PR changed `%w` to `%v` in the `fmt.Errorf` call:

```diff
- return nil, fmt.Errorf("FindByID(%d): %w", id, err)
+ return nil, fmt.Errorf("FindByID(%d): %v", id, err)
```

**With `%w`:** The original error (`sql.ErrNoRows`) is wrapped and preserved in the chain. `errors.Is(err, sql.ErrNoRows)` traverses: outer → inner → `sql.ErrNoRows` ✅.

**With `%v`:** The original error is converted to its **string representation** (`"sql: no rows in result set"`). The error chain now contains only a `*fmt.wrapError` with a plain string message. `errors.Is(err, sql.ErrNoRows)` finds no matching pointer in the chain → returns `false` ❌.

#### 2. Prevention Strategies:

1. **Linting:** Use the `errorlint` linter (part of `golangci-lint`) which flags `%v` usage with error values where `%w` was likely intended:
   ```bash
   golangci-lint run --enable errorlint
   ```

2. **Code Review Checklist:** Any PR changing `fmt.Errorf` lines should be reviewed for `%w` vs `%v` verb usage.

3. **Unit Tests:** Write tests that verify error chain behavior:
   ```go
   func TestFindByID_NotFound_ReturnsErrNoRows(t *testing.T) {
       _, err := repo.FindByID(999)
       if !errors.Is(err, sql.ErrNoRows) {
           t.Errorf("expected sql.ErrNoRows in chain, got: %v", err)
       }
   }
   ```
   This test would have caught the regression immediately.
