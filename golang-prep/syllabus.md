# Golang Developer Prep Syllabus

Welcome to the structured learning prep for Golang. This curriculum is designed to take you from a basic understanding of programming to having the core Golang concepts "at your fingertips," matching the expectations of a developer with 2+ years of professional Go experience.

The curriculum is structured into 9 modules, ordered logically from fundamentals with deep memory/internal explanations, up to concurrency, clean architecture, testing, and production deployments.

---

## 🗺️ Curriculum Overview

| Module                                                        | Level                    | Focus Areas                                                                                  |
| :------------------------------------------------------------ | :----------------------- | :------------------------------------------------------------------------------------------- |
| **Module 1: Go Fundamentals & Memory Models**           | Beginner -> Intermediate | Type system, Slices/Arrays/Maps internals, Pointers, Structs, Interface internals.           |
| **Module 2: Control Flow, Error Handling & Logging**    | Intermediate             | Idiomatic errors, wrapping/unwrapping, defer/panic/recover, structured logging.              |
| **Module 3: Concurrency Foundations**                   | Intermediate             | Goroutines, Channels (buffered vs unbuffered), Select, sync package (Mutex/WaitGroup).       |
| **Module 4: Advanced Concurrency & Memory Safety**      | Advanced                 | Go Scheduler (GMP), Race Conditions, Atomic operations, Context package deep-dive.           |
| **Module 5: Project Structure & Dependency Management** | Intermediate             | Go modules, standard layout, Clean Architecture, Manual Dependency Injection.                |
| **Module 6: Building REST APIs & Middleware**           | Intermediate             | `net/http` standard library, routing, middleware chaining, JSON serialization/validation.  |
| **Module 7: Database Interaction & Storage Layer**      | Intermediate             | `database/sql`, connection pooling, transactions, GORM vs Raw SQL, N+1 query problem.      |
| **Module 8: Testing, Mocking & Profiling**              | Intermediate -> Advanced | Table-driven tests, mock generation, integration testing, benchmarks, and profiling (pprof). |
| **Module 9: Production Deployment & Monitoring**        | Intermediate             | Multi-stage Docker builds, configuration patterns, graceful shutdowns, health checks.        |

---

## 📚 Module-by-Module Topic Breakdown

### 📂 Module 1: Go Fundamentals & Memory Models (Level: Beginner -> Intermediate)

* **Topic 1.1: Variables, Constants, and Type System**
  * Type inference (`:=` vs `var`), typed/untyped constants, `iota` usage.
* **Topic 1.2: Slices & Arrays under the Hood**
  * Array memory allocation vs Slice headers (pointer, len, cap).
  * Slice slicing/resizing, `append()` mechanics, slice growth algorithm, and backing array sharing gotchas.
* **Topic 1.3: Maps & Internals**
  * Hash map bucket layout, hash function, overflow buckets.
  * Map initialization (`make` vs `nil` map), panic cases, key types constraint, and why map iteration is randomized.
* **Topic 1.4: Pointers & Memory Allocation**
  * Stack vs Heap allocation, Escape Analysis (how the compiler decides).
  * Passing values vs pointers to functions, performance implications of heap allocation.
* **Topic 1.5: Structs, Methods & Receivers**
  * Value vs Pointer Receivers and when to use which. Struct padding and memory alignment.
* **Topic 1.6: Interfaces & Implicit Implementation**
  * Go's composition philosophy (why no inheritance).
  * Interface internals (`iface` and `eface` structs, runtime `itab`), comparison of nil interfaces (the famous `err != nil` interface gotcha).

---

### 📂 Module 2: Control Flow, Error Handling & Logging (Level: Intermediate)

* **Topic 2.1: Defer, Panic, and Recover**
  * Defer evaluation rules (arguments evaluated immediately), execution stack order.
  * When to use `panic`/`recover` (and why they should not be used as standard control flow exceptions).
* **Topic 2.2: Idiomatic Error Handling**
  * The Go philosophy of errors as values (`if err != nil`).
  * Creating custom errors, Sentinel errors vs custom struct errors.
* **Topic 2.3: Error Wrapping, Is, and As**
  * Wrapping errors with `fmt.Errorf("%w", err)`.
  * Using `errors.Is` and `errors.As` to inspect wrapped errors.
* **Topic 2.4: Structured Logging**
  * Standard `slog` library (introduced in 1.21), logging levels, output formatting (JSON/Text), context propagation.

---

### 📂 Module 3: Concurrency Foundations (Level: Intermediate)

* **Topic 3.1: Goroutines & The Scheduler (Intro)**
  * Goroutines vs OS threads (memory overhead, context switch costs).
* **Topic 3.2: Channels (Buffered vs Unbuffered)**
  * Channel states table: actions on `nil`, `closed`, and `open` channels (which panics, blocks, or returns zero value).
  * Communication synchronization vs buffering.
* **Topic 3.3: Select Statement**
  * Multiplexing channel operations, non-blocking sends/receives, timeouts with `time.After`.
* **Topic 3.4: The `sync` Package**
  * Mutual exclusion (`sync.Mutex` and `sync.RWMutex`).
  * Synchronization groups (`sync.WaitGroup`) and single execution (`sync.Once`).

---

### 📂 Module 4: Advanced Concurrency & Memory Safety (Level: Advanced)

* **Topic 4.1: The GMP Scheduler Model**
  * Goroutine (G), Machine (M), Processor (P) architecture. Work stealing and preemption.
* **Topic 4.2: Race Conditions & Atomics**
  * Race detector tool (`-race`). Low-level thread safety using `sync/atomic`.
* **Topic 4.3: The Context Package**
  * Context propagation, cancellations, deadlines/timeouts, and passing request-scoped values.
* **Topic 4.4: Advanced Concurrency Patterns**
  * Worker Pools, Fan-in / Fan-out, Generator, Pipeline, and rate-limiting patterns.

---

### 📂 Module 5: Project Structure & Dependency Management (Level: Intermediate)

* **Topic 5.1: Go Modules and Dependency Management**
  * `go.mod`, `go.sum`, `go mod tidy`, `go mod vendor`. Semantic versioning, upgrading, and dealing with private modules.
* **Topic 5.2: Standard Project Layout (Standard Go Project Layout)**
  * Structure: `/cmd`, `/internal`, `/pkg`, `/api`, `/configs`.
* **Topic 5.3: Clean Architecture & Decoupling**
  * Designing clear application boundaries: Handlers -> Services -> Repositories -> Database.
  * Dependency Injection (DI) without heavy frameworks (manual constructors).

---

### 📂 Module 6: Building REST APIs & Middleware (Level: Intermediate)

* **Topic 6.1: HTTP Fundamentals & Standard Routing**
  * `net/http` standard library. Routing options (standard library in Go 1.22+ vs `go-chi`).
* **Topic 6.2: Handlers, Request/Response Lifecycle**
  * JSON encoding/decoding, writing proper status codes, headers. Custom HTTP types.
* **Topic 6.3: Middleware Design**
  * Chaining functions, passing context between middlewares, writing auth, logging, and recovery middleware.
* **Topic 6.4: Input Validation**
  * Validating requests, structured API error responses.

---

### 📂 Module 7: Database Interaction & Storage Layer (Level: Intermediate)

* **Topic 7.1: database/sql Standard Library**
  * Drivers, connection pools, querying rows vs single row, parsing null values (`sql.NullString`).
* **Topic 7.2: SQL Injection & Prepared Statements**
  * Preventing SQL injection using placeholders, optimizing queries with prepared statements.
* **Topic 7.3: Transactions & Isolation Levels**
  * Managing database transactions (`db.BeginTx`), rollback and commit logic.
* **Topic 7.4: GORM vs Raw SQL**
  * Pros/cons of ORM vs raw/builder SQL (GORM, sqlc). Avoid the N+1 query problem, indexing, and connection leak debugging.

---

### 📂 Module 8: Testing, Mocking & Profiling (Level: Intermediate -> Advanced)

* **Topic 8.1: Table-Driven Unit Testing**
  * Write clean unit tests using Go's `testing` library. Parallel tests (`t.Parallel()`).
* **Topic 8.2: Interfaces & Mocking**
  * Designing testable code via interfaces. Mocking database and external services (using tools like `mockgen` or manual mocks).
* **Topic 8.3: Integration & Database Testing**
  * Testing with database connections, Docker containers (using `testcontainers-go`), and cleanups.
* **Topic 8.4: Benchmarking & Profiling (pprof)**
  * Benchmarking code, finding memory leaks and CPU bottlenecks using `go tool pprof`.

---

### 📂 Module 9: Production Deployment & Monitoring (Level: Intermediate)

* **Topic 9.1: Dockerizing Go Applications**
  * Writing efficient Dockerfiles (multi-stage builds, distroless images for security and size optimization).
* **Topic 9.2: Application Configuration**
  * Managing config with environment variables, `godotenv`, or config engines (e.g., `viper`).
* **Topic 9.3: Production Monitoring & Graceful Shutdown**
  * Capturing OS signals (`SIGTERM`, `SIGINT`) to shut down HTTP servers and DB connections cleanly. Health checks and Prometheus metrics.
