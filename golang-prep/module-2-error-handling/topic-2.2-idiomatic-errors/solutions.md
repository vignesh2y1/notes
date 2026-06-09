# Solutions: Idiomatic Error Handling

Below are the detailed solutions for all Topic 2.2 exercises.

---

## 🧠 Conceptual Questions

### Solution 1: Why Errors as Values?

#### 1. Why Go Chose Errors as Values:
Go's design philosophy prioritizes **explicitness and simplicity**. By making errors ordinary return values, every function call that can fail has a visible `error` in its return signature. The caller is forced to acknowledge the error immediately — they must assign it (or explicitly discard it with `_`). There is no hidden control flow; you can trace exactly what happens at every step by reading the code linearly.

#### 2. Problem This Solves:
Exceptions in Java/Python create **invisible control flow**. A function 5 levels deep can throw an exception that silently skips through 4 intermediate functions and gets caught in a distant `catch` block. This makes it difficult to:
* Reason about which functions can fail and how
* Understand the actual execution path during errors
* Avoid accidentally catching exceptions you didn't intend to handle

Go's approach eliminates this ambiguity. If a function returns an error, the error handling is right there, in the next 3 lines.

#### 3. One Disadvantage:
**Verbosity.** The `if err != nil { return err }` pattern is repeated extensively throughout Go code, making functions longer and more repetitive compared to equivalent exception-based code. This is a deliberate tradeoff — Go sacrifices conciseness for clarity.

---

### Solution 2: Sentinel vs. Custom Error — Design Decision

| Failure Condition | Error Type | Justification |
|:-----------------|:-----------|:-------------|
| **Card declined** | Sentinel error `ErrCardDeclined` | No additional data needed. Simple condition check. |
| **Insufficient funds** | Custom struct `InsufficientFundsError` | Caller needs `AvailableBalance` and `RequestedAmount` for user-facing messages. |
| **Card expired** | Custom struct `CardExpiredError` | Caller needs the `ExpiryDate` to show in the UI. |
| **Network timeout** | Sentinel error `ErrNetworkTimeout` | No additional data needed. Caller only needs to know "it timed out" for retry logic. |

```go
// Sentinels
var ErrCardDeclined   = errors.New("card declined")
var ErrNetworkTimeout = errors.New("network timeout")

// Custom struct errors
type InsufficientFundsError struct {
    Available float64
    Requested float64
}
func (e *InsufficientFundsError) Error() string {
    return fmt.Sprintf("insufficient funds: available %.2f, requested %.2f", e.Available, e.Requested)
}

type CardExpiredError struct {
    ExpiryDate time.Time
}
func (e *CardExpiredError) Error() string {
    return fmt.Sprintf("card expired on %s", e.ExpiryDate.Format("01/2006"))
}
```

---

### Solution 3: Error Identity

#### 1. Are `ErrA` and `ErrB` equal with `==`?
**No.** `errors.New()` returns a pointer to a newly allocated struct: `&errorString{s: "something failed"}`. Each call allocates a different struct at a different memory address. Since `==` on pointers compares memory addresses, `ErrA != ErrB` even though they have the same message string.

#### 2. Are they equal with `errors.Is(ErrA, ErrB)`?
**No.** `errors.Is()` uses pointer identity comparison (same as `==`) for basic errors. It only differs from `==` when the error implements a custom `Is(target error) bool` method or when dealing with wrapped error chains.

#### 3. What makes sentinel error comparison work?
**Pointer identity**, not the string message. This is why sentinels must be declared as package-level variables — the same variable pointer is used for all comparisons. Two different `errors.New("same message")` calls produce two different sentinel identities.

---

## 🛠️ Practical Problems

### Solution 4: Custom Error Type Implementation

```go
package main

import (
	"errors"
	"fmt"
)

// Custom error type
type APIError struct {
	StatusCode int
	Message    string
	RequestID  string
}

func (e *APIError) Error() string {
	return fmt.Sprintf("[%d] %s (request_id: %s)", e.StatusCode, e.Message, e.RequestID)
}

// User type
type User struct {
	ID    int
	Email string
}

// Function that returns APIError
func fetchUser(id int) (*User, error) {
	if id == 0 {
		return nil, &APIError{
			StatusCode: 403,
			Message:    "access denied",
			RequestID:  "req-abc-123",
		}
	}

	if id > 100 {
		return nil, &APIError{
			StatusCode: 404,
			Message:    fmt.Sprintf("user %d not found", id),
			RequestID:  "req-xyz-789",
		}
	}

	return &User{ID: id, Email: "user@example.com"}, nil
}

// Caller code using errors.As
func main() {
	_, err := fetchUser(999)
	if err != nil {
		var apiErr *APIError
		if errors.As(err, &apiErr) {
			fmt.Printf("API Error: status=%d, message=%s, request_id=%s\n",
				apiErr.StatusCode, apiErr.Message, apiErr.RequestID)
		} else {
			fmt.Println("Unknown error:", err)
		}
	}
}
```

#### Key Points:
* `errors.As(err, &apiErr)` checks if `err` (or any error in its wrapped chain) matches the target type `*APIError`.
* The target must be a **pointer to a pointer** for interface/struct error types: `&apiErr` where `apiErr` is `*APIError`.
* This decouples the caller from knowing the exact function that returned the error — they only need to know the error type.

---

### Solution 5: Swallowed Error Debugging

#### The Bugs:
There are **two** swallowed errors:

1. `data, _ := os.ReadFile("config.json")` — If the file doesn't exist, `data` is `nil` and the error is silently discarded.
2. `json.Unmarshal(data, &cfg)` — If `data` is `nil` or contains invalid JSON, the unmarshal error is silently discarded.

#### The Consequence:
`cfg` remains a zero-value `AppConfig{Port: 0, Database: ""}`. The server starts on port `0` (random OS-assigned port) and connects to an empty database string. In production, this could mean:
* The service is unreachable (wrong port).
* Database queries fail with cryptic connection errors.
* Hours of debugging because the root cause (missing config file) is hidden.

#### The Fix:
```go
package main

import (
	"encoding/json"
	"fmt"
	"log"
	"os"
)

type AppConfig struct {
	Port     int    `json:"port"`
	Database string `json:"database"`
}

func LoadConfig() (AppConfig, error) {
	data, err := os.ReadFile("config.json")
	if err != nil {
		return AppConfig{}, fmt.Errorf("read config file: %w", err)
	}

	var cfg AppConfig
	if err := json.Unmarshal(data, &cfg); err != nil {
		return AppConfig{}, fmt.Errorf("parse config JSON: %w", err)
	}

	return cfg, nil
}

func main() {
	cfg, err := LoadConfig()
	if err != nil {
		log.Fatalf("Failed to load configuration: %v", err)
	}
	fmt.Printf("Starting server on port %d, connecting to %s\n", cfg.Port, cfg.Database)
}
```

---

## 💼 Scenario-Based Questions

### Solution 6: Domain Error Translation Layer

#### 1. Domain-Level Errors (service package):
```go
package service

import "errors"

var (
	ErrUserNotFound  = errors.New("user not found")
	ErrDuplicateEmail = errors.New("email already registered")
)
```

#### 2. Repository Function:
```go
package repository

import (
	"database/sql"
	"errors"
	"fmt"

	"myapp/service"
)

type User struct {
	ID       int
	Email    string
	Password string
}

func GetUserByEmail(db *sql.DB, email string) (*User, error) {
	var u User
	err := db.QueryRow(
		"SELECT id, email, password FROM users WHERE email = $1", email,
	).Scan(&u.ID, &u.Email, &u.Password)

	if err != nil {
		if errors.Is(err, sql.ErrNoRows) {
			return nil, service.ErrUserNotFound // Translate to domain error
		}
		return nil, fmt.Errorf("query user by email: %w", err) // Unexpected DB error
	}
	return &u, nil
}
```

#### 3. Service Function:
```go
package service

import (
	"database/sql"
	"errors"
	"fmt"

	"myapp/repository"
)

func RegisterUser(db *sql.DB, email, password string) error {
	// Check if user already exists
	_, err := repository.GetUserByEmail(db, email)
	if err == nil {
		// User exists — duplicate email
		return ErrDuplicateEmail
	}
	if !errors.Is(err, ErrUserNotFound) {
		// Unexpected error (not "not found")
		return fmt.Errorf("check existing user: %w", err)
	}

	// User doesn't exist — create them
	if err := repository.CreateUser(db, email, password); err != nil {
		return fmt.Errorf("create user: %w", err)
	}

	return nil
}
```

#### 4. Handler Code:
```go
package handler

import (
	"errors"
	"net/http"

	"myapp/service"
)

func RegisterHandler(w http.ResponseWriter, r *http.Request) {
	email := r.FormValue("email")
	password := r.FormValue("password")

	err := service.RegisterUser(db, email, password)
	if err != nil {
		switch {
		case errors.Is(err, service.ErrDuplicateEmail):
			http.Error(w, "Email already registered", http.StatusConflict) // 409
		case errors.Is(err, service.ErrUserNotFound):
			http.Error(w, "User not found", http.StatusNotFound) // 404
		default:
			http.Error(w, "Internal server error", http.StatusInternalServerError) // 500
		}
		return
	}

	w.WriteHeader(http.StatusCreated) // 201
	w.Write([]byte("User registered successfully"))
}
```

#### Why This Design Works:
* The **repository** translates `sql.ErrNoRows` → `service.ErrUserNotFound`. Callers never know about `database/sql`.
* The **service** uses domain-level sentinels for business decisions.
* The **handler** maps domain errors → HTTP status codes. If you switch from PostgreSQL to MongoDB, only the repository changes — service and handler code remain untouched.

---

### Solution 7: The Error Log Explosion

#### 1. Bad Pattern — No Context:
```go
func callExternalAPI() error {
	return errors.New("connection refused")
}

func fetchUserProfile() error {
	return callExternalAPI() // bare return, no context
}

func handleRequest() error {
	return fetchUserProfile() // bare return, no context
}

func main() {
	if err := handleRequest(); err != nil {
		log.Printf("ERROR: %v", err)
	}
}
// Log output: ERROR: connection refused
// Which function? Which API? Which user? No idea.
```

#### 2. Good Pattern — Wrapping With Context:
```go
func callExternalAPI(endpoint string) error {
	// Simulated network failure
	return errors.New("connection refused")
}

func fetchUserProfile(userID int) error {
	err := callExternalAPI("https://api.example.com/users")
	if err != nil {
		return fmt.Errorf("fetch profile for user %d: %w", userID, err)
	}
	return nil
}

func handleRequest(requestID string, userID int) error {
	err := fetchUserProfile(userID)
	if err != nil {
		return fmt.Errorf("handleRequest [%s]: %w", requestID, err)
	}
	return nil
}

func main() {
	if err := handleRequest("req-abc-123", 42); err != nil {
		log.Printf("ERROR: %v", err)
	}
}
```

#### 3. Final Logged Error Message:
```
ERROR: handleRequest [req-abc-123]: fetch profile for user 42: connection refused
```

This single log line tells you:
* **Where** the request started: `handleRequest [req-abc-123]`
* **What operation** failed: `fetch profile for user 42`
* **Why** it failed: `connection refused`

Each layer adds its own context using `%w`, building a complete error chain that reads like a breadcrumb trail from the top-level handler down to the root cause.
