# Topic 2.3: Error Wrapping, Is, and As

Go 1.13 introduced a standardized system for **wrapping** errors into chains and **inspecting** those chains. This allows you to add context to errors as they propagate up the call stack while preserving the ability to programmatically check the original root cause.

---

## 1. Concepts & Theory

### 1.1 Error Wrapping with `fmt.Errorf` and `%w`

The `%w` verb in `fmt.Errorf` **wraps** an error, embedding the original error inside a new error with additional context:

```go
func readConfig(path string) error {
    data, err := os.ReadFile(path)
    if err != nil {
        return fmt.Errorf("readConfig: reading file %s: %w", path, err)
    }
    // ...
    return nil
}
```

The resulting error is a **chain** — the outer error contains the inner error. This chain can be traversed using `errors.Is()` and `errors.As()`.

#### `%w` vs `%v` — The Critical Difference:

| Verb | Wraps the error? | Preserves error chain? | `errors.Is`/`errors.As` works? |
|:-----|:----------------|:----------------------|:------------------------------|
| `%w` | ✅ Yes | ✅ Yes | ✅ Yes |
| `%v` | ❌ No | ❌ No — converts to plain string | ❌ No |

```go
// Using %w — error chain is preserved
err1 := fmt.Errorf("layer 1: %w", io.EOF)
errors.Is(err1, io.EOF) // true ✅

// Using %v — error chain is broken
err2 := fmt.Errorf("layer 1: %v", io.EOF)
errors.Is(err2, io.EOF) // false ❌ (io.EOF is lost, converted to string "EOF")
```

**Rule:** Use `%w` when you want callers to be able to inspect the wrapped error. Use `%v` when you intentionally want to **hide** the internal error from callers (API boundary, security reasons).

---

### 1.2 The `Unwrap()` Method

The wrapping mechanism works through the `Unwrap()` method. Any error type that implements `Unwrap() error` participates in the error chain:

```go
type wrappedError struct {
    msg string
    err error // the inner/wrapped error
}

func (e *wrappedError) Error() string {
    return e.msg
}

func (e *wrappedError) Unwrap() error {
    return e.err // exposes the inner error for chain traversal
}
```

When you use `fmt.Errorf("...: %w", err)`, the returned error internally stores `err` and implements `Unwrap()` to return it.

#### The Error Chain:
Each call to `fmt.Errorf("...: %w", innerErr)` creates a linked list of errors:

```
outerError → middleError → rootError
   ↓            ↓            ↓
 Unwrap()    Unwrap()     Unwrap() → nil
```

`errors.Is` and `errors.As` walk this chain from outer to inner, checking each error in sequence.

---

### 1.3 `errors.Is()` — Identity Matching Through the Chain

`errors.Is(err, target)` checks if any error in the chain **matches** the target. It walks the chain by repeatedly calling `Unwrap()`.

```go
import (
    "errors"
    "fmt"
    "io"
)

func process() error {
    return fmt.Errorf("process: %w",
        fmt.Errorf("read data: %w", io.EOF))
}

func main() {
    err := process()
    // Chain: "process: read data: EOF" → "read data: EOF" → io.EOF

    fmt.Println(errors.Is(err, io.EOF)) // true — found io.EOF deep in the chain
}
```

#### How `errors.Is` Works Internally:

```go
func Is(err, target error) bool {
    for {
        // 1. Direct comparison (pointer identity)
        if err == target {
            return true
        }
        // 2. Check if err has a custom Is() method
        if x, ok := err.(interface{ Is(error) bool }); ok {
            if x.Is(target) {
                return true
            }
        }
        // 3. Unwrap and check the next error in the chain
        if err = errors.Unwrap(err); err == nil {
            return false
        }
    }
}
```

#### Custom `Is()` Method:
You can define a custom `Is()` method on your error type to control matching behavior:

```go
type NotFoundError struct {
    Resource string
}

func (e *NotFoundError) Error() string {
    return fmt.Sprintf("%s not found", e.Resource)
}

// Custom Is: any NotFoundError matches ErrNotFound sentinel
func (e *NotFoundError) Is(target error) bool {
    return target == ErrNotFound
}

var ErrNotFound = errors.New("not found")
```

Now `errors.Is(err, ErrNotFound)` returns `true` for any `*NotFoundError`, regardless of the `Resource` field value.

---

### 1.4 `errors.As()` — Type Extraction Through the Chain

`errors.As(err, &target)` searches the error chain for an error that matches the **type** of `target` and, if found, sets `target` to that error value.

```go
type QueryError struct {
    Query   string
    Message string
}

func (e *QueryError) Error() string {
    return fmt.Sprintf("query %q: %s", e.Query, e.Message)
}

func runQuery() error {
    qErr := &QueryError{Query: "SELECT * FROM users", Message: "table not found"}
    return fmt.Errorf("runQuery: %w", qErr) // Wrapped in context
}

func main() {
    err := runQuery()

    var qErr *QueryError
    if errors.As(err, &qErr) {
        fmt.Println("Failed query:", qErr.Query)   // "SELECT * FROM users"
        fmt.Println("Error:", qErr.Message)          // "table not found"
    }
}
```

#### How `errors.As` Works Internally:

```go
func As(err error, target interface{}) bool {
    for {
        // Check if current error is assignable to target type
        if reflect.TypeOf(err).AssignableTo(reflect.TypeOf(target).Elem()) {
            // Set target to this error
            reflect.ValueOf(target).Elem().Set(reflect.ValueOf(err))
            return true
        }
        // Check custom As() method
        if x, ok := err.(interface{ As(interface{}) bool }); ok {
            if x.As(target) {
                return true
            }
        }
        // Unwrap and try next error
        if err = errors.Unwrap(err); err == nil {
            return false
        }
    }
}
```

---

### 1.5 `errors.Is` vs `errors.As` — When to Use Which

| Use Case | Function | Example |
|:---------|:---------|:--------|
| Check for a specific sentinel error | `errors.Is` | `errors.Is(err, sql.ErrNoRows)` |
| Check for a specific error type and extract data | `errors.As` | `errors.As(err, &pathErr)` |
| Simple "is this the error?" check | `errors.Is` | `errors.Is(err, context.Canceled)` |
| "What kind of error is this and what details does it carry?" | `errors.As` | Extract status code from `*APIError` |

---

### 1.6 Multiple Wrapping (Go 1.20+)

Starting with Go 1.20, `fmt.Errorf` supports wrapping **multiple errors** using multiple `%w` verbs:

```go
err := fmt.Errorf("operation failed: %w and %w", err1, err2)
```

This creates an error that contains **two** wrapped errors. `errors.Is` and `errors.As` will check both branches of the chain.

For types implementing multiple wraps, Go 1.20 also added an alternative `Unwrap` signature:

```go
func (e *MultiError) Unwrap() []error {
    return e.errs // returns multiple wrapped errors
}
```

---

## 2. Edge Cases & Limitations

### 2.1 Breaking the Chain with `%v`

A common mistake is using `%v` instead of `%w`, which silently breaks the error chain:

```go
inner := errors.New("database connection lost")

// ❌ Chain broken — inner error becomes a string
outer := fmt.Errorf("query failed: %v", inner)
errors.Is(outer, inner) // false!

// ✅ Chain preserved
outer := fmt.Errorf("query failed: %w", inner)
errors.Is(outer, inner) // true
```

### 2.2 Wrapping Nil Errors

If you wrap a `nil` error, `fmt.Errorf` returns a **non-nil** error:

```go
var err error = nil
wrapped := fmt.Errorf("something: %w", err)
fmt.Println(wrapped == nil) // false! wrapped is "something: <nil>"
```

Always check if the error is nil **before** wrapping:
```go
if err != nil {
    return fmt.Errorf("operation: %w", err)
}
return nil
```

### 2.3 Over-Wrapping (Duplicate Context)

Avoid adding the same context at multiple levels:

```go
// ❌ Over-wrapped — "read config" appears twice
// readConfig: reading file: readConfig: open config.yaml: no such file or directory
return fmt.Errorf("readConfig: %w", fmt.Errorf("readConfig: %w", err))
```

Each function should add **its own unique context** — the operation it was performing, not the function name of what it called.

---

## 3. Real-World Usage & Best Practices

### 3.1 The Wrapping Strategy

Follow this pattern when wrapping errors through layers:

```go
// Repository layer
func (r *UserRepo) FindByID(id int) (*User, error) {
    // ... DB query
    return nil, fmt.Errorf("find user by id %d: %w", id, err)
}

// Service layer
func (s *UserService) GetProfile(id int) (*Profile, error) {
    user, err := s.repo.FindByID(id)
    if err != nil {
        return nil, fmt.Errorf("get user profile: %w", err)
    }
    // ...
}

// Handler layer
func handleGetProfile(w http.ResponseWriter, r *http.Request) {
    profile, err := userService.GetProfile(userID)
    if err != nil {
        if errors.Is(err, ErrUserNotFound) {
            http.Error(w, "User not found", 404)
            return
        }
        log.Printf("ERROR: %v", err)
        // Logs: "ERROR: get user profile: find user by id 42: user not found"
    }
}
```

### 3.2 When NOT to Wrap

Do not wrap errors when:
1. **You are at an API boundary** and don't want to leak internal implementation details:
   ```go
   // Public API — use %v to hide internal details
   return fmt.Errorf("authentication failed: %v", internalErr)
   ```
2. **The error is already fully contextual** and adding more context would be redundant.

### 3.3 The `os.PathError` Pattern

The standard library uses custom error types with `Unwrap()` extensively. `os.PathError` is a perfect example:

```go
type PathError struct {
    Op   string // "open", "read", "write"
    Path string // file path
    Err  error  // underlying error (e.g., syscall.ENOENT)
}

func (e *PathError) Error() string {
    return e.Op + " " + e.Path + ": " + e.Err.Error()
}

func (e *PathError) Unwrap() error {
    return e.Err
}
```

This allows both:
* `errors.Is(err, os.ErrNotExist)` — check the root cause
* `errors.As(err, &pathErr)` — extract the operation and file path

---

## 4. Code Examples: Good vs. Bad

### ❌ Bad: Using `%v` and Breaking the Chain
```go
package main

import (
	"errors"
	"fmt"
	"io"
)

func readStream() error {
	return fmt.Errorf("readStream: %v", io.EOF) // %v breaks the chain
}

func main() {
	err := readStream()

	if errors.Is(err, io.EOF) {
		fmt.Println("End of file detected") // ❌ Never reached
	} else {
		fmt.Println("Unknown error:", err)
		// Output: Unknown error: readStream: EOF
	}
}
```

### ✅ Good: Using `%w` to Preserve the Chain
```go
package main

import (
	"errors"
	"fmt"
	"io"
)

func readStream() error {
	return fmt.Errorf("readStream: %w", io.EOF) // %w preserves the chain
}

func main() {
	err := readStream()

	if errors.Is(err, io.EOF) {
		fmt.Println("End of file detected") // ✅ Correctly detected
	}
}
```

---

### ❌ Bad: Type Assertion Instead of `errors.As`
```go
package main

import "fmt"

type DBError struct {
	Code    int
	Message string
}

func (e *DBError) Error() string {
	return fmt.Sprintf("db error %d: %s", e.Code, e.Message)
}

func query() error {
	return fmt.Errorf("query failed: %w", &DBError{Code: 1045, Message: "access denied"})
}

func main() {
	err := query()

	// ❌ Direct type assertion fails because err is a wrapped *fmt.wrapError, not *DBError
	if dbErr, ok := err.(*DBError); ok {
		fmt.Println("DB Code:", dbErr.Code)
	} else {
		fmt.Println("Could not extract DB error") // This branch executes
	}
}
```

### ✅ Good: Using `errors.As` to Search the Chain
```go
package main

import (
	"errors"
	"fmt"
)

type DBError struct {
	Code    int
	Message string
}

func (e *DBError) Error() string {
	return fmt.Sprintf("db error %d: %s", e.Code, e.Message)
}

func query() error {
	return fmt.Errorf("query failed: %w", &DBError{Code: 1045, Message: "access denied"})
}

func main() {
	err := query()

	var dbErr *DBError
	if errors.As(err, &dbErr) { // ✅ Searches through the entire chain
		fmt.Println("DB Code:", dbErr.Code)       // DB Code: 1045
		fmt.Println("DB Message:", dbErr.Message)  // DB Message: access denied
	}
}
```
