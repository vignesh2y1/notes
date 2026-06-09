# Solutions: Structured Logging with `slog`

Below are the detailed solutions for all Topic 2.4 exercises.

---

## 🧠 Conceptual Questions

### Solution 1: Text vs JSON Handler

#### 1. The Two Built-in Handlers:
* **`slog.NewTextHandler(w, opts)`** — Outputs human-readable key=value format. Best for **local development** where you're reading logs in a terminal.
* **`slog.NewJSONHandler(w, opts)`** — Outputs machine-parseable JSON. Best for **production** where logs are consumed by aggregation tools (Elasticsearch, Datadog, Grafana Loki).

#### 2. Do Logging Calls Change?
**No.** The `slog.Handler` interface abstracts the output format. All logging calls (`slog.Info`, `slog.Error`, `logger.With`, etc.) remain identical. You only change the handler at initialization time. This is the **Strategy pattern** — the logger delegates formatting to the handler.

#### 3. Effect of `slog.SetDefault` on `log.Println`:
After calling `slog.SetDefault(logger)`, the standard library's `log` package is **bridged** to the slog handler. Calls to `log.Println("message")` will be routed through the slog handler and produce structured output (at `Info` level). This means:
* Legacy code using `log.Println` automatically gets structured logging.
* The output format matches your slog handler (JSON or Text).

---

### Solution 2: Log Level Filtering

#### Which Calls Produce Output:
```
logger.Debug("debug message")  → ❌ Suppressed (Debug=-4 < Warn=4)
logger.Info("info message")    → ❌ Suppressed (Info=0 < Warn=4)
logger.Warn("warn message")   → ✅ Output (Warn=4 >= Warn=4)
logger.Error("error message")  → ✅ Output (Error=8 >= Warn=4)
```

#### The Filtering Mechanism:
Each log level has a numeric value. The handler's `Level` option sets the **minimum** threshold. Before writing a log record, the handler calls `Enabled(ctx, level)` which returns `true` only if `level >= minimumLevel`. Any log call below the threshold is a no-op — the handler never receives the record.

---

### Solution 3: Key-Value Misalignment Bug

#### The Output:
```json
{"time":"...","level":"INFO","msg":"user signup","user_id":42,"email":"plan","!BADKEY":"premium"}
```

#### What Went Wrong:
The `"email"` key has no corresponding value. `slog` consumes the next argument `"plan"` as the value for `"email"`. Then `"premium"` is left without a key, so slog marks it as `!BADKEY`.

The intended call was:
```go
slog.Info("user signup",
	"user_id", 42,
	"email", "alice@example.com",  // Missing value on original!
	"plan", "premium",
)
```

#### Prevention:
Use type-safe `slog.Attr` helpers:
```go
slog.Info("user signup",
	slog.Int("user_id", 42),
	slog.String("email", "alice@example.com"),
	slog.String("plan", "premium"),
)
```

Each `slog.String`, `slog.Int`, etc. is a single argument that bundles key+value together, making misalignment impossible. Also, enable the `sloglint` linter in CI.

---

## 🛠️ Practical Problems

### Solution 4: Production Logger Setup

```go
package main

import (
	"log/slog"
	"os"
)

func SetupLogger(env string) *slog.Logger {
	var logger *slog.Logger

	switch env {
	case "production":
		opts := &slog.HandlerOptions{
			Level:     slog.LevelInfo,
			AddSource: true, // Include file:line in logs
		}
		logger = slog.New(slog.NewJSONHandler(os.Stdout, opts))

	default: // "development" or any other value
		opts := &slog.HandlerOptions{
			Level: slog.LevelDebug, // Show all levels
		}
		logger = slog.New(slog.NewTextHandler(os.Stderr, opts))
	}

	// Set as the global default logger
	slog.SetDefault(logger)

	return logger
}

func main() {
	env := os.Getenv("APP_ENV")
	if env == "" {
		env = "development"
	}

	logger := SetupLogger(env)

	logger.Info("application started", "environment", env)
	logger.Debug("debug mode active") // Only visible in development
}
```

#### Key Design Decisions:
* **Production** uses JSON to stdout (container logs are typically captured from stdout).
* **Production** enables `AddSource` for debugging production issues.
* **Development** uses Text to stderr (human-readable, won't mix with application output on stdout).
* **Development** enables Debug level for full visibility.
* `slog.SetDefault` ensures that any code using `slog.Info()` (package-level) or `log.Println()` uses our configured handler.

---

### Solution 5: Implement `LogValuer` for CreditCard

```go
package main

import (
	"fmt"
	"log/slog"
	"os"
)

type CreditCard struct {
	Number     string
	CVV        string
	ExpiryDate string
	HolderName string
}

// LogValue implements slog.LogValuer — controls log output
func (c CreditCard) LogValue() slog.Value {
	// Mask card number: show only last 4 digits
	maskedNumber := "****"
	if len(c.Number) >= 4 {
		maskedNumber = "****" + c.Number[len(c.Number)-4:]
	}

	return slog.GroupValue(
		slog.String("number", maskedNumber),
		slog.String("cvv", "***"),
		slog.String("expiry_date", c.ExpiryDate),
		slog.String("holder_name", c.HolderName),
	)
}

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	card := CreditCard{
		Number:     "4111111111111111",
		CVV:        "123",
		ExpiryDate: "12/2025",
		HolderName: "Alice Smith",
	}

	logger.Info("payment processed", "card", card)
}
```

#### Output:
```json
{
  "time": "...",
  "level": "INFO",
  "msg": "payment processed",
  "card": {
    "number": "****1111",
    "cvv": "***",
    "expiry_date": "12/2025",
    "holder_name": "Alice Smith"
  }
}
```

#### How It Works:
When `slog` encounters a value that implements `slog.LogValuer`, it calls `LogValue()` instead of using the default struct representation. This gives you full control over what appears in logs — perfect for PII redaction, password masking, and sensitive data handling.

---

### Solution 6: Request-Scoped Logger Middleware

```go
package main

import (
	"context"
	"fmt"
	"log/slog"
	"net/http"
	"os"
	"time"
)

// Context key for storing the logger
type ctxKeyLogger struct{}

// LoggerFromContext retrieves the request-scoped logger, or returns the default
func LoggerFromContext(ctx context.Context) *slog.Logger {
	if logger, ok := ctx.Value(ctxKeyLogger{}).(*slog.Logger); ok {
		return logger
	}
	return slog.Default()
}

// RequestLoggerMiddleware adds structured logging to every request
func RequestLoggerMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		requestID := fmt.Sprintf("req-%d", time.Now().UnixNano())

		// Create request-scoped child logger
		logger := slog.With(
			slog.String("request_id", requestID),
			slog.String("method", r.Method),
			slog.String("path", r.URL.Path),
		)

		logger.Info("request started")

		// Store logger in context for downstream handlers
		ctx := context.WithValue(r.Context(), ctxKeyLogger{}, logger)

		// Serve the request with the enriched context
		next.ServeHTTP(w, r.WithContext(ctx))

		// Log completion with duration
		logger.Info("request completed",
			slog.Int64("duration_ms", time.Since(start).Milliseconds()),
		)
	})
}

// Example downstream handler using the context logger
func userHandler(w http.ResponseWriter, r *http.Request) {
	logger := LoggerFromContext(r.Context())
	logger.Info("fetching user data", slog.Int("user_id", 42))
	w.Write([]byte("OK"))
}

func main() {
	slog.SetDefault(slog.New(slog.NewJSONHandler(os.Stdout, nil)))

	mux := http.NewServeMux()
	mux.HandleFunc("/users", userHandler)

	handler := RequestLoggerMiddleware(mux)
	http.ListenAndServe(":8080", handler)
}
```

#### Key Design Points:
* **Custom context key:** Uses a private struct type `ctxKeyLogger{}` as the context key (prevents collisions with other packages using `string` keys).
* **Fallback to default:** `LoggerFromContext` returns `slog.Default()` if no logger is in context, so it never returns nil.
* **Child logger with `.With()`:** The request_id, method, and path are attached once and automatically included in all subsequent log calls from that logger.
* **Downstream access:** Any handler or service function can call `LoggerFromContext(ctx)` to get the request-scoped logger with all the correlation attributes.

---

## 💼 Scenario-Based Questions

### Solution 7: Dynamic Log Level for Production Debugging

#### 1. Logger Setup with `slog.LevelVar`:
```go
package main

import (
	"log/slog"
	"net/http"
	"os"
	"strings"
)

var logLevel slog.LevelVar

func init() {
	logLevel.Set(slog.LevelInfo) // Default production level

	opts := &slog.HandlerOptions{
		Level: &logLevel, // Pass pointer — changes are reflected dynamically
	}
	logger := slog.New(slog.NewJSONHandler(os.Stdout, opts))
	slog.SetDefault(logger)
}
```

#### 2. Admin Endpoint to Change Level:
```go
func handleLogLevel(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPost {
		http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
		return
	}

	level := strings.ToLower(r.URL.Query().Get("level"))

	switch level {
	case "debug":
		logLevel.Set(slog.LevelDebug)
	case "info":
		logLevel.Set(slog.LevelInfo)
	case "warn":
		logLevel.Set(slog.LevelWarn)
	case "error":
		logLevel.Set(slog.LevelError)
	default:
		http.Error(w, "Invalid level. Use: debug, info, warn, error", http.StatusBadRequest)
		return
	}

	slog.Info("log level changed", "new_level", level)
	w.Write([]byte("Log level set to: " + level))
}

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/admin/log-level", handleLogLevel)

	slog.Info("server starting", "port", 8080)
	http.ListenAndServe(":8080", mux)
}
```

#### Usage:
```bash
# Normal operation — only INFO and above
curl -X POST "http://localhost:8080/admin/log-level?level=debug"
# Now DEBUG logs are visible — investigate the bug
# ...
curl -X POST "http://localhost:8080/admin/log-level?level=info"
# Back to normal — DEBUG logs suppressed again
```

#### 3. Why `slog.LevelVar` Is Concurrent-Safe:
`slog.LevelVar` uses `sync/atomic` internally to store the level value. The `Set()` and `Level()` methods use atomic load/store operations, which are safe for concurrent access from multiple goroutines without requiring a mutex. This means the HTTP handler goroutine can change the level while request-processing goroutines are simultaneously checking it — no race conditions.

---

### Solution 8: Structured Error Logging Strategy

#### Before (Unstructured):
```go
if err != nil {
	log.Printf("ERROR: %v", err)
	return err
}
```

#### After (Structured with `slog`):
```go
package main

import (
	"errors"
	"fmt"
	"log/slog"
	"net"
	"os"
)

func callExternalService() error {
	// Simulated network error
	return &net.OpError{
		Op:  "dial",
		Net: "tcp",
		Err: errors.New("connection refused"),
	}
}

func processOrder(logger *slog.Logger, orderID string) error {
	err := callExternalService()
	if err != nil {
		logger.Error("external service call failed",
			slog.String("function", "processOrder"),
			slog.String("order_id", orderID),
			slog.String("error", err.Error()),
			slog.String("error_type", fmt.Sprintf("%T", err)),
		)
		return fmt.Errorf("processOrder %s: %w", orderID, err)
	}
	return nil
}

func main() {
	slog.SetDefault(slog.New(slog.NewJSONHandler(os.Stdout, nil)))

	// Simulate request-scoped logger
	logger := slog.With("request_id", "req-abc-123")

	err := processOrder(logger, "ORD-555")
	if err != nil {
		logger.Error("order processing failed", "error", err)
	}
}
```

#### Expected JSON Output:
```json
{
  "time": "2024-01-15T10:30:45.123Z",
  "level": "ERROR",
  "msg": "external service call failed",
  "request_id": "req-abc-123",
  "function": "processOrder",
  "order_id": "ORD-555",
  "error": "dial tcp: connection refused",
  "error_type": "*net.OpError"
}
```

#### Why This Is Better:
* **`request_id`** — correlate this error with all other logs from the same request.
* **`function`** — know exactly which function encountered the error.
* **`order_id`** — identify which business entity was affected.
* **`error`** — the human-readable error message.
* **`error_type`** — know the Go type for programmatic diagnosis.
* **JSON format** — instantly queryable in any log aggregation tool.

Compare this to the old `ERROR: connection refused` which tells you almost nothing.
