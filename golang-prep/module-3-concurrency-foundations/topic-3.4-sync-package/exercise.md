# Exercises: The `sync` Package

Test your understanding of Go's low-level synchronization primitives, memory barriers, non-reentrant locks, copy diagnostics, and thread-safe design patterns.

---

## 🧠 Conceptual Questions

### Question 1: Mutex Normal vs. Starvation Mode
1. Under what specific condition does a `sync.Mutex` switch from normal mode to starvation mode?
2. What are the key architectural differences in how lock acquisition occurs in starvation mode versus normal mode?
3. Why doesn't Go's Mutex always operate in starvation mode (which seems fairer)?

---

### Question 2: The Double-Checked Locking in `sync.Once`
A developer wants to implement a singleton database pool without `sync.Once`. They write:

```go
var dbConn *DB
var mu sync.Mutex

func GetConnection() *DB {
	if dbConn == nil {
		mu.Lock()
		defer mu.Unlock()
		if dbConn == nil {
			dbConn = initDB()
		}
	}
	return dbConn
}
```

1. Explain the "double check" logic used here.
2. In Go, why is this code technically prone to a memory/compiler race condition if we do not use atomic load/stores?
3. How does `sync.Once` solve this problem internally?

---

## 🛠️ Practical Problems

### Question 3: Thread-Safe Generic Cache
Write a complete Go implementation of a thread-safe cache:
1. Define a struct `type Cache[K comparable, V any] struct`.
2. Implement `Get(key K) (V, bool)` using read locks.
3. Implement `Set(key K, val V)` using write locks.
4. Explain why you chose `sync.RWMutex` instead of `sync.Mutex`.

---

### Question 4: The Mutex Copy Bug
Identify and fix the compile-time/runtime bug in the following struct definition and usage:

```go
package main

import (
	"fmt"
	"sync"
)

type WorkerMetrics struct {
	mu     sync.Mutex
	counts map[string]int
}

func (wm WorkerMetrics) Increment(metric string) {
	wm.mu.Lock()
	defer wm.mu.Unlock()
	wm.counts[metric]++
}

func main() {
	wm := WorkerMetrics{
		counts: make(map[string]int),
	}
	var wg sync.WaitGroup
	for i := 0; i < 100; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			wm.Increment("requests")
		}()
	}
	wg.Wait()
	fmt.Println("Requests:", wm.counts["requests"])
}
```

1. Why does this code cause a data race, even though `wm.mu.Lock()` is called?
2. How do you fix it?

---

## 💼 Scenario-Based Questions

### Question 5: The Recursive Lock Deadlock
You are building an account manager. You write the following two methods:

```go
type Account struct {
	mu      sync.Mutex
	balance int
}

func (a *Account) GetBalance() int {
	a.mu.Lock()
	defer a.mu.Unlock()
	return a.balance
}

func (a *Account) Withdraw(amount int) bool {
	a.mu.Lock()
	defer a.mu.Unlock()

	// Check balance using the helper method
	if a.GetBalance() < amount {
		return false
	}
	a.balance -= amount
	return true
}
```

1. What happens when a user calls `Withdraw` on an account?
2. Explain the system state of the goroutine executing this call.
3. How do you rewrite the `Account` struct methods to prevent this issue while maintaining thread-safety?

---

### Question 6: The Passed-by-Value WaitGroup
A developer is writing a batch processing tool. They pass a `sync.WaitGroup` to workers:

```go
package main

import (
	"fmt"
	"sync"
)

func processTask(id int, wg sync.WaitGroup) {
	defer wg.Done()
	fmt.Println("Processing task:", id)
}

func main() {
	var wg sync.WaitGroup
	for i := 1; i <= 5; i++ {
		wg.Add(1)
		go processTask(i, wg)
	}
	wg.Wait()
	fmt.Println("All tasks processed.")
}
```

1. Explain what happens when you run this program.
2. Does the program execute successfully, crash, or deadlock? Explain the system mechanics.
3. What is the correct way to pass `sync.WaitGroup` to a function?
