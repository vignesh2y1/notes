# Topic 2.4: Structured Logging with `slog`

Go 1.21 introduced the `log/slog` package — a **structured, leveled logging** library in the standard library. Before `slog`, Go developers relied on third-party libraries like `zerolog`, `zap`, or `logrus` for structured logging. Now, the standard library provides a production-grade solution with JSON and text output, log levels, contextual attributes, and custom handlers.

---

## 1. Concepts & Theory

### 1.1 Why Structured Logging?

Traditional `log.Println` produces **unstructured** text logs:

```
2024/01/15 10:30:45 User login failed for user_id=42 from ip=192.168.1.100
```

This is human-readable but **machine-unfriendly**. Parsing this with log aggregation tools (Elasticsearch, Datadog, Grafana Loki) requires fragile regex patterns that break whenever the message format changes.

**Structured logging** outputs logs as key-value pairs (typically JSON):

```json
{"time":"2024-01-15T10:30:45Z","level":"WARN","msg":"User login failed","user_id":42,"ip":"192.168.1.100"}
```

This is:
* **Machine-parseable** — log aggregators index fields automatically.
* **Queryable** — filter by `user_id=42` or `level="ERROR"` directly.
* **Consistent** — adding new fields doesn't break existing parsers.

---

### 1.2 Core Concepts of `slog`

The `slog` package is built around three key types:

#### 1. `slog.Logger`
The main type you interact with. It provides leveled logging methods: `Debug`, `Info`, `Warn`, `Error`.

```go
logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
logger.Info("server started", "port", 8080)
```

#### 2. `slog.Handler`
An interface that controls **how** and **where** logs are written. `slog` ships with two built-in handlers:

| Handler | Output Format | Use Case |
|:--------|:-------------|:---------|
| `slog.NewTextHandler(w, opts)` | `time=... level=INFO msg="..." key=value` | Local development |
| `slog.NewJSONHandler(w, opts)` | `{"time":"...","level":"INFO","msg":"...","key":"value"}` | Production (machine parsing) |

#### 3. `slog.Attr`
A key-value pair attached to a log record. Attributes can be strings, ints, durations, errors, or any value.

---

### 1.3 Basic Usage

```go
package main

import (
	"log/slog"
	"os"
)

func main() {
	// Create a JSON logger writing to stdout
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	// Log with key-value pairs
	logger.Info("user registered",
		"user_id", 42,
		"email", "alice@example.com",
		"plan", "premium",
	)
}
```

#### Output (JSON):
```json
{"time":"2024-01-15T10:30:45.123Z","level":"INFO","msg":"user registered","user_id":42,"email":"alice@example.com","plan":"premium"}
```

#### Output (Text handler):
```
time=2024-01-15T10:30:45.123Z level=INFO msg="user registered" user_id=42 email=alice@example.com plan=premium
```

---

### 1.4 Log Levels

`slog` defines four standard log levels:

| Level | Numeric Value | Purpose |
|:------|:-------------|:--------|
| `slog.LevelDebug` | -4 | Detailed internal state, only for development |
| `slog.LevelInfo` | 0 | Normal operational events |
| `slog.LevelWarn` | 4 | Potential issues that aren't errors yet |
| `slog.LevelError` | 8 | Errors that need attention |

#### Setting the Minimum Level:
By default, `slog` logs at `Info` level and above (Debug is suppressed). To change this:

```go
opts := &slog.HandlerOptions{
	Level: slog.LevelDebug, // Show all levels including Debug
}
logger := slog.New(slog.NewJSONHandler(os.Stdout, opts))
logger.Debug("cache miss", "key", "user:42") // Now visible
```

#### Dynamic Level Changing at Runtime:
Use `slog.LevelVar` for dynamic level control without restarting:

```go
var logLevel slog.LevelVar
logLevel.Set(slog.LevelInfo) // Start at INFO

opts := &slog.HandlerOptions{Level: &logLevel}
logger := slog.New(slog.NewJSONHandler(os.Stdout, opts))

// Later, perhaps via an admin API endpoint:
logLevel.Set(slog.LevelDebug) // Increase verbosity without restart
```

---

### 1.5 Setting the Default Logger

Instead of passing a logger everywhere, you can set a global default:

```go
logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
slog.SetDefault(logger)

// Now you can use the package-level functions:
slog.Info("using default logger", "key", "value")
```

After `slog.SetDefault`, even the old `log.Println` calls route through the slog handler.

---

### 1.6 Type-Safe Attributes with `slog.Attr`

Instead of alternating `key, value, key, value` arguments (which can cause misalignment bugs), use strongly-typed `slog.Attr`:

```go
logger.LogAttrs(ctx, slog.LevelInfo, "order processed",
	slog.String("order_id", "ORD-12345"),
	slog.Int("items", 3),
	slog.Float64("total", 149.99),
	slog.Duration("processing_time", 250*time.Millisecond),
	slog.Time("timestamp", time.Now()),
	slog.Bool("express", true),
)
```

#### Why `slog.Attr` Over Raw Key-Value Pairs:
* **Type safety:** Compiler catches wrong types.
* **Performance:** Avoids `interface{}` boxing and reflection.
* **No misalignment:** With raw pairs, forgetting a value shifts all subsequent keys: `"key1", "val1", "key2"` — `key2` becomes a value.

---

### 1.7 Grouping Attributes

Group related attributes under a namespace to avoid key collisions:

```go
logger.Info("request completed",
	slog.Group("request",
		slog.String("method", "POST"),
		slog.String("path", "/api/users"),
		slog.Int("status", 201),
	),
	slog.Group("user",
		slog.Int("id", 42),
		slog.String("role", "admin"),
	),
)
```

#### JSON Output:
```json
{
  "time": "2024-01-15T10:30:45Z",
  "level": "INFO",
  "msg": "request completed",
  "request": {
    "method": "POST",
    "path": "/api/users",
    "status": 201
  },
  "user": {
    "id": 42,
    "role": "admin"
  }
}
```

---

### 1.8 Child Loggers with `Logger.With()`

Create child loggers that carry persistent attributes — these attributes are automatically included in every subsequent log call:

```go
// Base logger
logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

// Request-scoped child logger
reqLogger := logger.With(
	"request_id", "req-abc-123",
	"user_id", 42,
)

// Every log from reqLogger includes request_id and user_id
reqLogger.Info("processing order", "order_id", "ORD-555")
reqLogger.Warn("inventory low", "product_id", "SKU-100")
```

#### Output:
```json
{"time":"...","level":"INFO","msg":"processing order","request_id":"req-abc-123","user_id":42,"order_id":"ORD-555"}
{"time":"...","level":"WARN","msg":"inventory low","request_id":"req-abc-123","user_id":42,"product_id":"SKU-100"}
```

This is essential for **request tracing** — the `request_id` appears on every log line without manually passing it to every function.

---

### 1.9 Context Integration

`slog` methods accept `context.Context` for propagating request-scoped data:

```go
func handleRequest(ctx context.Context) {
	// Log with context — handlers can extract values from ctx
	slog.InfoContext(ctx, "handling request", "endpoint", "/api/users")
}
```

Context-aware logging is critical for integrating with:
* **Distributed tracing** (OpenTelemetry trace IDs)
* **Request cancellation** awareness
* **Middleware** that injects metadata into context

---

### 1.10 Source Code Location

Enable source file and line number in logs for easier debugging:

```go
opts := &slog.HandlerOptions{
	AddSource: true,
}
logger := slog.New(slog.NewJSONHandler(os.Stdout, opts))
logger.Info("debug info")
```

#### Output:
```json
{"time":"...","level":"INFO","source":{"function":"main.main","file":"main.go","line":15},"msg":"debug info"}
```

---

## 2. Edge Cases & Limitations

### 2.1 Key-Value Misalignment

When using alternating `key, value` arguments, a missing value silently corrupts the output:

```go
// ❌ Bug: "email" has no value, "plan" becomes the value for "email"
slog.Info("signup", "user_id", 42, "email", "plan", "premium")
```

This produces: `user_id=42 email=plan !BADKEY=premium`

**Prevention:** Use `slog.Attr` typed helpers (`slog.String`, `slog.Int`) instead of raw key-value pairs. The `sloglint` linter also catches these misalignment issues.

### 2.2 Logging Sensitive Data

`slog` does not automatically redact sensitive fields. Logging passwords, API keys, or PII violates security policies:

```go
// ❌ PII leak
slog.Info("user login", "email", user.Email, "password", user.Password)
```

Implement a custom `slog.Handler` or use `LogValuer` interface to redact sensitive fields (covered in best practices).

### 2.3 Performance: Avoid Expensive Operations in Log Arguments

Log arguments are **always evaluated**, even if the log level is suppressed:

```go
// ❌ json.Marshal runs even if Debug level is disabled
slog.Debug("full request", "body", string(mustMarshal(request)))
```

For expensive operations, check the level first:
```go
if logger.Enabled(ctx, slog.LevelDebug) {
    slog.Debug("full request", "body", string(mustMarshal(request)))
}
```

---

## 3. Real-World Usage & Best Practices

### 3.1 The `LogValuer` Interface — Custom Type Logging

Implement `slog.LogValuer` on your types to control how they appear in logs:

```go
type User struct {
	ID       int
	Email    string
	Password string // sensitive!
}

// LogValue controls what gets logged — redacts sensitive fields
func (u User) LogValue() slog.Value {
	return slog.GroupValue(
		slog.Int("id", u.ID),
		slog.String("email", u.Email),
		// Password is deliberately omitted
	)
}
```

Now when you log a `User`:
```go
slog.Info("user action", "user", user)
// Output: {"user":{"id":42,"email":"alice@example.com"}} — no password!
```

### 3.2 HTTP Middleware Logger

A production-grade request logging middleware:

```go
func LoggingMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		requestID := r.Header.Get("X-Request-ID")

		// Create request-scoped logger
		logger := slog.With(
			"request_id", requestID,
			"method", r.Method,
			"path", r.URL.Path,
			"remote_addr", r.RemoteAddr,
		)

		logger.Info("request started")

		// Wrap response writer to capture status code
		wrapped := &statusWriter{ResponseWriter: w, status: 200}
		next.ServeHTTP(wrapped, r)

		logger.Info("request completed",
			"status", wrapped.status,
			"duration_ms", time.Since(start).Milliseconds(),
		)
	})
}

type statusWriter struct {
	http.ResponseWriter
	status int
}

func (w *statusWriter) WriteHeader(code int) {
	w.status = code
	w.ResponseWriter.WriteHeader(code)
}
```

### 3.3 Error Logging with Stack Context

When logging errors, always include contextual attributes:

```go
func processOrder(orderID string) error {
	err := chargePayment(orderID)
	if err != nil {
		slog.Error("payment processing failed",
			"order_id", orderID,
			"error", err,                    // the error message
			"error_type", fmt.Sprintf("%T", err), // the error type
		)
		return fmt.Errorf("processOrder %s: %w", orderID, err)
	}
	return nil
}
```

---

## 4. Code Examples: Good vs. Bad

### ❌ Bad: Unstructured Logging with `log.Printf`
```go
package main

import "log"

func main() {
	userID := 42
	action := "login"
	ip := "192.168.1.100"

	// Unstructured — hard to parse, impossible to query
	log.Printf("User %d performed %s from %s", userID, action, ip)
}
// Output: 2024/01/15 10:30:45 User 42 performed login from 192.168.1.100
```

### ✅ Good: Structured Logging with `slog`
```go
package main

import (
	"log/slog"
	"os"
)

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	logger.Info("user action",
		slog.Int("user_id", 42),
		slog.String("action", "login"),
		slog.String("ip", "192.168.1.100"),
	)
}
// Output: {"time":"...","level":"INFO","msg":"user action","user_id":42,"action":"login","ip":"192.168.1.100"}
```

---

### ❌ Bad: Logging Without Request Context
```go
package main

import "log/slog"

func processPayment(amount float64) {
	// No request ID, no user ID — impossible to correlate in production
	slog.Info("payment processed", "amount", amount)
}
```

### ✅ Good: Using Child Loggers for Request Tracing
```go
package main

import "log/slog"

func handlePaymentRequest(requestID string, userID int) {
	logger := slog.With(
		"request_id", requestID,
		"user_id", userID,
	)

	logger.Info("payment request received")

	processPayment(logger, 99.99)

	logger.Info("payment request completed")
}

func processPayment(logger *slog.Logger, amount float64) {
	logger.Info("charging payment", "amount", amount)
	// Output includes request_id and user_id automatically
}
```

---

### ❌ Bad: Logging Sensitive Data
```go
package main

import "log/slog"

type LoginRequest struct {
	Email    string
	Password string
}

func main() {
	req := LoginRequest{Email: "alice@example.com", Password: "s3cret!"}
	// ❌ Password visible in logs!
	slog.Info("login attempt", "request", req)
}
```

### ✅ Good: Using `LogValuer` to Redact Sensitive Fields
```go
package main

import (
	"log/slog"
	"os"
)

type LoginRequest struct {
	Email    string
	Password string
}

func (r LoginRequest) LogValue() slog.Value {
	return slog.GroupValue(
		slog.String("email", r.Email),
		slog.String("password", "***REDACTED***"),
	)
}

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
	req := LoginRequest{Email: "alice@example.com", Password: "s3cret!"}
	logger.Info("login attempt", "request", req)
	// Output: {"request":{"email":"alice@example.com","password":"***REDACTED***"}}
}
```
