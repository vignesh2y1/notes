# Exercises: Structured Logging with `slog`

Test your understanding of `slog` handlers, log levels, attributes, groups, child loggers, `LogValuer`, and production logging patterns.

---

## 🧠 Conceptual Questions

### Question 1: Text vs JSON Handler
1. What are the two built-in handlers in `slog`? When would you use each one?
2. If you switch from `TextHandler` to `JSONHandler`, do you need to change any of your logging calls (`slog.Info`, `slog.Error`, etc.)?
3. How does `slog.SetDefault` affect calls to the old `log.Println`?

---

### Question 2: Log Level Filtering
Given the following logger setup:
```go
opts := &slog.HandlerOptions{
	Level: slog.LevelWarn,
}
logger := slog.New(slog.NewJSONHandler(os.Stdout, opts))
```

Which of these log calls will produce output?
```go
logger.Debug("debug message")
logger.Info("info message")
logger.Warn("warn message")
logger.Error("error message")
```
Explain the filtering mechanism.

---

### Question 3: Key-Value Misalignment Bug
Predict the output of this logging call. Explain what went wrong:
```go
slog.Info("user signup",
	"user_id", 42,
	"email",
	"plan", "premium",
)
```

How would you prevent this class of bug?

---

## 🛠️ Practical Problems

### Question 4: Production Logger Setup
Write a function `SetupLogger(env string) *slog.Logger` that:
1. If `env == "production"` → returns a JSON handler writing to `os.Stdout` at `Info` level with source location enabled.
2. If `env == "development"` → returns a Text handler writing to `os.Stderr` at `Debug` level.
3. Sets the returned logger as the default via `slog.SetDefault`.

---

### Question 5: Implement `LogValuer` for a Sensitive Type
You have the following `CreditCard` struct:
```go
type CreditCard struct {
	Number     string // "4111111111111111"
	CVV        string // "123"
	ExpiryDate string // "12/2025"
	HolderName string // "Alice Smith"
}
```

Implement the `slog.LogValuer` interface so that when this struct is logged:
* `Number` shows only the last 4 digits: `"****1111"`
* `CVV` is completely redacted: `"***"`
* `ExpiryDate` is shown as-is
* `HolderName` is shown as-is

Write test logging code that demonstrates the output.

---

### Question 6: Request-Scoped Logger Middleware
Write an HTTP middleware function `RequestLoggerMiddleware(next http.Handler) http.Handler` that:
1. Generates a unique request ID (use `fmt.Sprintf("req-%d", time.Now().UnixNano())`).
2. Creates a child logger with attributes: `request_id`, `method`, `path`.
3. Logs `"request started"` at Info level.
4. Calls the next handler.
5. Logs `"request completed"` with the `duration_ms` attribute.
6. Stores the logger in the request context so downstream handlers can retrieve it.

Also write a helper function `LoggerFromContext(ctx context.Context) *slog.Logger` that retrieves the logger from context (falling back to the default logger if none is set).

---

## 💼 Scenario-Based Questions

### Question 7: Dynamic Log Level for Production Debugging
You are running a production service that normally logs at `INFO` level. A critical bug appears, and you need to temporarily enable `DEBUG` logging to investigate — without restarting the service.

1. Show how to set up the logger using `slog.LevelVar` for dynamic level changes.
2. Write a simple HTTP handler `POST /admin/log-level` that accepts a query parameter `level` (values: `debug`, `info`, `warn`, `error`) and changes the log level at runtime.
3. Explain why `slog.LevelVar` is safe for concurrent use.

---

### Question 8: Structured Error Logging Strategy
Your team's current error logging looks like this:
```go
if err != nil {
	log.Printf("ERROR: %v", err)
	return err
}
```

This produces logs like:
```
2024/01/15 10:30:45 ERROR: connection refused
```

Rewrite this pattern using `slog` to produce structured error logs that include:
* The error message
* The error type (e.g., `*net.OpError`)
* The function name where the error occurred
* The request ID (from a child logger)

Show the expected JSON output.
