# Topic 2.2: Idiomatic Error Handling

Go deliberately does not have exceptions. Errors in Go are **values** — ordinary values returned from functions, inspected with `if` statements, and passed around like any other data. This design forces developers to handle errors explicitly at every call site, leading to more reliable and predictable software.

---

## 1. Concepts & Theory

### 1.1 The `error` Interface

The entire error system in Go is built on a single, tiny interface defined in the `builtin` package:

```go
type error interface {
    Error() string
}
```

Any type that implements an `Error() string` method satisfies the `error` interface. This simplicity is intentional — it makes errors composable, wrappable, and inspectable.

---

### 1.2 The `if err != nil` Pattern

The canonical Go error handling pattern returns errors as the **last return value** and checks them immediately:

```go
result, err := doSomething()
if err != nil {
    return fmt.Errorf("doSomething failed: %w", err)
}
// Use result safely here
```

#### Why Go Chose This Over Exceptions:

| Aspect | Exceptions (Java/Python) | Errors as Values (Go) |
|:-------|:------------------------|:---------------------|
| Control flow | Invisible — exceptions jump across stack frames | Explicit — errors are checked at each call site |
| Forgetting to handle | Silent — unhandled exceptions propagate up | The compiler forces you to assign the return value |
| Performance | Exception creation captures expensive stack traces | Error creation is a simple struct allocation |
| Readability | Error handling is scattered in distant `catch` blocks | Error handling is co-located with the call |

Go's approach trades brevity for **explicitness**. Every error is visible in the code, at the exact point where it occurs.

---

### 1.3 Creating Errors

#### Method 1: `errors.New()` — Simple Static Errors
Use `errors.New()` for simple error messages without dynamic data:

```go
import "errors"

var ErrNotFound = errors.New("resource not found")
```

Internally, `errors.New` returns a pointer to a struct containing the message string. Because it returns a **pointer**, each call to `errors.New` with the same message returns a **different pointer** (unique error identity).

#### Method 2: `fmt.Errorf()` — Formatted Dynamic Errors
Use `fmt.Errorf` to create errors with dynamic context:

```go
return fmt.Errorf("user %d not found in database %s", userID, dbName)
```

---

### 1.4 Sentinel Errors

A **sentinel error** is a pre-declared, package-level error variable used as a fixed identity for specific error conditions. Callers compare against the sentinel using `errors.Is()`.

```go
package repository

import "errors"

// Sentinel errors — exported, documented, and stable
var (
    ErrNotFound     = errors.New("not found")
    ErrUnauthorized = errors.New("unauthorized")
    ErrConflict     = errors.New("resource already exists")
)
```

#### Naming Convention:
Sentinel errors in Go follow the pattern `Err<Condition>`:
* `io.EOF`
* `sql.ErrNoRows`
* `os.ErrNotExist`
* `context.DeadlineExceeded`

#### When to Use Sentinels:
Use sentinel errors when:
* The error represents a **specific, well-known condition** the caller needs to branch on.
* The error is **part of your package's public API contract**.

Do NOT use sentinels when:
* The error message needs dynamic context (user IDs, file paths) — use `fmt.Errorf` or custom errors instead.
* The error is internal and callers don't need to distinguish it from other errors.

---

### 1.5 Custom Error Types (Struct Errors)

When you need to attach structured metadata to an error (e.g., HTTP status codes, field names, retry info), define a custom error struct:

```go
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation error: field '%s' — %s", e.Field, e.Message)
}
```

#### Returning Custom Errors:
```go
func validateAge(age int) error {
    if age < 0 || age > 150 {
        return &ValidationError{
            Field:   "age",
            Message: "must be between 0 and 150",
        }
    }
    return nil
}
```

Callers can use `errors.As()` to extract the structured metadata:
```go
var valErr *ValidationError
if errors.As(err, &valErr) {
    fmt.Println("Bad field:", valErr.Field)
}
```

---

### 1.6 Sentinel Errors vs. Custom Struct Errors — When to Use Which

| Criteria | Sentinel Error (`errors.New`) | Custom Struct Error |
|:---------|:-----------------------------|:-------------------|
| Contains dynamic data | ❌ No | ✅ Yes (fields) |
| Caller needs metadata | ❌ No | ✅ Yes (`errors.As`) |
| Identity comparison | ✅ `errors.Is(err, ErrNotFound)` | ✅ `errors.As(err, &target)` |
| Simple condition check | ✅ Best choice | ❌ Overkill |
| API error with status code | ❌ Insufficient | ✅ Best choice |

---

### 1.7 The Error Handling Anti-Pattern: Swallowing Errors

**Never ignore errors silently.** This is the single most dangerous anti-pattern in Go:

```go
// ❌ NEVER DO THIS
result, _ := riskyOperation()
```

If `riskyOperation` fails, you proceed with a zero-value `result`, causing unpredictable downstream bugs. The only acceptable use of `_` for errors is when you are certain the operation cannot fail and you have documented why:

```go
// Acceptable: fmt.Fprintf to stdout never returns an error in practice
_, _ = fmt.Fprintf(os.Stdout, "debug: %v\n", data)
```

---

## 2. Edge Cases & Limitations

### 2.1 Error String Comparison Is Fragile

Never compare errors by their string message:

```go
// ❌ Fragile — breaks if the message changes
if err.Error() == "not found" {
    // ...
}
```

Use `errors.Is()` for sentinel comparisons and `errors.As()` for type checks instead. String messages are for humans (logging); identity checks are for code branching.

### 2.2 The Blank Identifier `_` Hides Bugs

Go does not force you to use the error return value. The blank identifier `_` silently discards it:

```go
f, _ := os.Open("config.yaml") // If this fails, f is nil
f.Read(buf)                     // nil pointer dereference → panic
```

Linters like `errcheck` and `golangci-lint` catch these. Always configure them in CI.

---

## 3. Real-World Usage & Best Practices

### 3.1 Adding Context to Errors (Error Wrapping Preview)

When propagating errors up the call stack, **always add context** about what operation failed. Without context, a bare `return err` produces cryptic error messages like `"connection refused"` with no indication of where it came from.

```go
// ❌ Bad: no context
func loadConfig() error {
    data, err := os.ReadFile("config.yaml")
    if err != nil {
        return err // "open config.yaml: no such file or directory" — which function?
    }
    // ...
}

// ✅ Good: with context
func loadConfig() error {
    data, err := os.ReadFile("config.yaml")
    if err != nil {
        return fmt.Errorf("loadConfig: reading config file: %w", err)
    }
    // ...
}
```

### 3.2 Organizing Errors in a Package

For well-structured packages, declare all sentinel errors at the top of a dedicated `errors.go` file:

```go
// errors.go
package user

import "errors"

var (
    ErrNotFound       = errors.New("user not found")
    ErrDuplicateEmail = errors.New("email already registered")
    ErrInvalidInput   = errors.New("invalid input")
)
```

This provides a single source of truth for all error conditions your package exposes.

### 3.3 Common Standard Library Sentinels to Know

| Package | Sentinel | Meaning |
|:--------|:---------|:--------|
| `io` | `io.EOF` | End of input stream |
| `database/sql` | `sql.ErrNoRows` | Query returned zero rows |
| `os` | `os.ErrNotExist` | File/directory does not exist |
| `os` | `os.ErrPermission` | Insufficient permissions |
| `context` | `context.Canceled` | Context was canceled |
| `context` | `context.DeadlineExceeded` | Context deadline passed |
| `net/http` | `http.ErrServerClosed` | Server was shut down gracefully |

---

## 4. Code Examples: Good vs. Bad

### ❌ Bad: Comparing Error Strings
```go
package main

import (
	"fmt"
	"os"
)

func main() {
	_, err := os.Open("missing.txt")
	if err != nil {
		// Fragile: this string could change across Go versions
		if err.Error() == "open missing.txt: no such file or directory" {
			fmt.Println("File not found")
		}
	}
}
```

### ✅ Good: Using `errors.Is` with `os.ErrNotExist`
```go
package main

import (
	"errors"
	"fmt"
	"os"
)

func main() {
	_, err := os.Open("missing.txt")
	if err != nil {
		if errors.Is(err, os.ErrNotExist) {
			fmt.Println("File not found")
		} else {
			fmt.Println("Unexpected error:", err)
		}
	}
}
```

---

### ❌ Bad: Exposing Implementation Details in Error Returns
```go
package main

import (
	"database/sql"
	"fmt"
)

// Exposes sql.ErrNoRows to callers — coupling them to your database choice
func GetUserEmail(db *sql.DB, id int) (string, error) {
	var email string
	err := db.QueryRow("SELECT email FROM users WHERE id = $1", id).Scan(&email)
	return email, err // Caller must know about sql.ErrNoRows
}

func main() {
	// Caller is now tightly coupled to database/sql
	email, err := GetUserEmail(nil, 1)
	if err == sql.ErrNoRows {
		fmt.Println("User not found")
	}
	_ = email
}
```

### ✅ Good: Translating Internal Errors to Domain Errors
```go
package main

import (
	"database/sql"
	"errors"
	"fmt"
)

// Domain-level sentinel error
var ErrUserNotFound = errors.New("user not found")

func GetUserEmail(db *sql.DB, id int) (string, error) {
	var email string
	err := db.QueryRow("SELECT email FROM users WHERE id = $1", id).Scan(&email)
	if err != nil {
		if errors.Is(err, sql.ErrNoRows) {
			return "", ErrUserNotFound // Translate to domain error
		}
		return "", fmt.Errorf("query user %d: %w", id, err)
	}
	return email, nil
}

func main() {
	email, err := GetUserEmail(nil, 1)
	if errors.Is(err, ErrUserNotFound) {
		fmt.Println("User not found") // Caller uses domain error, decoupled from DB
	}
	_ = email
}
```
