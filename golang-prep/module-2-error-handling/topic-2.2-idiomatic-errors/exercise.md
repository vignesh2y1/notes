# Exercises: Idiomatic Error Handling

Test your understanding of Go's error philosophy, sentinel errors, custom error types, error swallowing, and domain error translation patterns.

---

## 🧠 Conceptual Questions

### Question 1: Why Errors as Values?
1. Explain in your own words why Go chose "errors as values" instead of exceptions (try/catch).
2. What problem does this solve that exceptions in Java/Python don't?
3. Name one disadvantage of Go's error handling approach compared to exceptions.

---

### Question 2: Sentinel vs. Custom Error — Design Decision
You are designing a `payment` package that processes credit card transactions. The package needs to communicate the following failure conditions to callers:
* Card declined (no additional data needed)
* Insufficient funds (need to include the available balance and requested amount)
* Card expired (need to include the expiry date)
* Network timeout (no additional data needed)

For each failure condition, decide whether you would use a **sentinel error** or a **custom error struct**. Justify each decision.

---

### Question 3: Error Identity
Consider the following code:
```go
var ErrA = errors.New("something failed")
var ErrB = errors.New("something failed")
```
1. Are `ErrA` and `ErrB` equal when compared with `==`? Why or why not?
2. Are they equal when compared with `errors.Is(ErrA, ErrB)`?
3. What makes sentinel error comparison work — the string message or the pointer identity?

---

## 🛠️ Practical Problems

### Question 4: Custom Error Type Implementation
Implement a custom error type `APIError` that satisfies the `error` interface and carries structured metadata for an HTTP API:
* `StatusCode int` — the HTTP status code
* `Message string` — human-readable error message
* `RequestID string` — unique request identifier for tracing

Requirements:
1. Implement the `Error() string` method.
2. Write a function `fetchUser(id int) (*User, error)` that returns an `*APIError` when the user is not found (404) or when access is denied (403).
3. Write caller code that uses `errors.As` to extract the `APIError` and log the status code and request ID.

---

### Question 5: Swallowed Error Debugging
The following code silently swallows an error. The application works in development but fails mysteriously in production. Find the bug, explain the consequence, and fix it.

```go
package main

import (
	"encoding/json"
	"fmt"
	"os"
)

type AppConfig struct {
	Port     int    `json:"port"`
	Database string `json:"database"`
}

func LoadConfig() AppConfig {
	data, _ := os.ReadFile("config.json")

	var cfg AppConfig
	json.Unmarshal(data, &cfg)

	return cfg
}

func main() {
	cfg := LoadConfig()
	fmt.Printf("Starting server on port %d, connecting to %s\n", cfg.Port, cfg.Database)
}
```

---

## 💼 Scenario-Based Questions

### Question 6: Domain Error Translation Layer
You are building a user registration service with three layers:
1. **Handler layer** (HTTP) — receives requests, returns JSON responses
2. **Service layer** — business logic
3. **Repository layer** — database queries

The repository layer uses `database/sql` and may return `sql.ErrNoRows`. The handler layer needs to return proper HTTP status codes (404 for not found, 409 for duplicate, 500 for internal errors).

Design:
1. Define domain-level sentinel errors in the service package: `ErrUserNotFound`, `ErrDuplicateEmail`.
2. Write a repository function `GetUserByEmail(email string) (*User, error)` that translates `sql.ErrNoRows` into the domain sentinel.
3. Write a service function `RegisterUser(email, password string) error` that:
   * Checks if user exists (calls repository)
   * If user exists, returns `ErrDuplicateEmail`
   * If user doesn't exist, creates the user
4. Write handler code that maps domain errors to HTTP status codes.

---

### Question 7: The Error Log Explosion
Your team's microservice logs contain thousands of error entries like:
```
ERROR: connection refused
ERROR: connection refused
ERROR: connection refused
```
These errors provide no context about which function, which service call, or which user triggered the failure.

Given this call chain:
```
main() → handleRequest() → fetchUserProfile() → callExternalAPI()
```
Where `callExternalAPI` returns `errors.New("connection refused")`:

1. Show the **bad pattern** where each function does `return err` without context.
2. Show the **good pattern** where each function wraps the error with context using `fmt.Errorf`.
3. What should the final logged error message look like?
