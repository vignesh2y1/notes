# Solutions: The `sync` Package

Here are the step-by-step solutions, code fixes, and system analyses for the Topic 3.4 exercises.

---

## 🧠 Conceptual Questions

### Solution 1: Mutex Normal vs. Starvation Mode

#### 1. Transition Condition:
* A `sync.Mutex` switches from normal mode to starvation mode when a waiting goroutine in the FIFO queue has been unable to acquire the lock for more than **1 millisecond**.

#### 2. Architectural Differences:
* **Normal Mode:** 
  - Woken-up waiters compete with newly arrived goroutines currently running on the CPU for lock acquisition. 
  - If a newly arrived goroutine wins the competition, the woken-up waiter is placed back at the *front* of the FIFO wait queue.
* **Starvation Mode:**
  - The releasing goroutine hands over ownership of the lock **directly** to the waiter at the head of the FIFO queue.
  - Newly arrived goroutines do not attempt to acquire the lock or spin; they are directly appended to the *tail* of the FIFO wait queue.

#### 3. Why Starvation Mode is Not the Default:
* **Throughput:** Normal mode has significantly higher performance and throughput because newly arriving goroutines are already running on the CPU and can acquire the lock immediately, avoiding context switch costs.
* Starvation mode forces thread scheduling context switches, which degrades throughput but is necessary to prevent extreme tail-latency starvation.

---

### Solution 2: The Double-Checked Locking in `sync.Once`

#### 1. Double-Check Logic:
* The first `if dbConn == nil` check is outside the lock to avoid acquiring the lock once the connection is initialized (fast-path).
* The second `if dbConn == nil` check is inside the lock (`mu.Lock()`) to prevent multiple concurrent goroutines that bypassed the first check from initializing the variable multiple times.

#### 2. Compiler/Memory Race Condition in Go:
* Without atomic operations, standard reads (`if dbConn == nil`) and writes (`dbConn = initDB()`) can execute concurrently across CPU cores.
* Go's memory model does not guarantee that writes to a multi-word variable are visible to all cores instantly. 
* A core might see a non-nil pointer value for `dbConn` but access it before the memory fields inside the struct are fully initialized (out-of-order execution / memory barrier reordering). This causes a nil pointer panic or memory corruption.

#### 3. How `sync.Once` Solves It:
* `sync.Once` uses an atomic variable `done` and double-checked locking:
  ```go
  if atomic.LoadUint32(&o.done) == 0 {
      o.doSlow(f)
  }
  ```
* The fast-path uses `atomic.LoadUint32`, which creates a **memory fence (barrier)**, ensuring that any subsequent reads see the fully initialized state of the initialized object.

---

## 🛠️ Practical Problems

### Solution 3: Thread-Safe Generic Cache

We use `sync.RWMutex` because reading cache items is typically much more frequent than writing/updating items. Multiple readers can lookup keys in parallel without blocking each other.

```go
package main

import (
	"fmt"
	"sync"
)

type Cache[K comparable, V any] struct {
	mu   sync.RWMutex
	data map[K]V
}

func NewCache[K comparable, V any]() *Cache[K, V] {
	return &Cache[K, V]{
		data: make(map[K]V),
	}
}

func (c *Cache[K, V]) Get(key K) (V, bool) {
	c.mu.RLock()         // Acquire read lock (multiple readers can read concurrently)
	defer c.mu.RUnlock() // Release read lock

	val, ok := c.data[key]
	return val, ok
}

func (c *Cache[K, V]) Set(key K, val V) {
	c.mu.Lock()         // Acquire write lock (blocks all readers and other writers)
	defer c.mu.Unlock() // Release write lock

	c.data[key] = val
}

func main() {
	cache := NewCache[string, int]()
	cache.Set("apples", 5)

	val, found := cache.Get("apples")
	fmt.Printf("Found: %t, Value: %d\n", found, val)
}
```

---

### Solution 4: The Mutex Copy Bug

#### 1. Why the data race occurs:
* The receiver of `Increment` is defined as `(wm WorkerMetrics)` (value receiver).
* When a goroutine calls `wm.Increment("requests")`, a new copy of `WorkerMetrics` is created for that invocation, copying the mutex `wm.mu` by value.
* As a result, each goroutine locks a **different, copied Mutex instance**, which does not protect the shared map from concurrent writes.
* The map is also passed by pointer reference internally (maps are references in Go), so all goroutines write to the same map concurrently, causing a fatal panic: `fatal error: concurrent map writes`.

#### 2. The Fix:
Change the method receiver of `Increment` to a pointer receiver (`*WorkerMetrics`) so that all invocations share the exact same Mutex.

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

// FIX: Use pointer receiver to prevent Mutex copying
func (wm *WorkerMetrics) Increment(metric string) {
	wm.mu.Lock()
	defer wm.mu.Unlock()
	wm.counts[metric]++
}

func main() {
	// Use pointer reference
	wm := &WorkerMetrics{
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
	fmt.Println("Requests:", wm.counts["requests"]) // Requests: 100
}
```

---

## 💼 Scenario-Based Questions

### Solution 5: The Recursive Lock Deadlock

#### 1 & 2. What happens and System State:
* When a goroutine calls `Withdraw`, it acquires the lock: `a.mu.Lock()`.
* Inside `Withdraw`, it evaluates `a.GetBalance()`.
* `GetBalance` attempts to acquire the lock again: `a.mu.Lock()`.
* Because Go's Mutex is **non-reentrant**, the goroutine blocks, waiting for the lock to be released.
* However, the only entity that can release the lock is the goroutine itself, which is blocked.
* **Result:** The goroutine blocks indefinitely, resulting in a **deadlock**.

#### 3. Corrected Implementation (Separating exported locked methods from internal unlocked methods):
The best practice to resolve recursive locking is to split your methods into public (locked) and private (unlocked helper) methods:

```go
package main

import "sync"

type Account struct {
	mu      sync.Mutex
	balance int
}

// Public locked method
func (a *Account) GetBalance() int {
	a.mu.Lock()
	defer a.mu.Unlock()
	return a.getBalanceUnlocked()
}

// Private unlocked helper
func (a *Account) getBalanceUnlocked() int {
	return a.balance
}

// Public locked method
func (a *Account) Withdraw(amount int) bool {
	a.mu.Lock()
	defer a.mu.Unlock()

	// Call private unlocked helper to prevent recursive lock acquisition
	if a.getBalanceUnlocked() < amount {
		return false
	}
	a.balance -= amount
	return true
}
```

---

### Solution 6: The Passed-by-Value WaitGroup

#### 1. What happens:
* In `main()`, `wg.Add(1)` increments the main WaitGroup's counter.
* When spawning `processTask(i, wg)`, the WaitGroup is passed **by value** (copied).
* Inside `processTask`, calling `wg.Done()` decrements the *copied* WaitGroup instance.
* The original WaitGroup `wg` in `main()` is never decremented.

#### 2. System Mechanics and Output:
* The workers complete and execute `wg.Done()` on their separate copies, which has no effect on the main coordinator.
* `main()` executes `wg.Wait()`. Since the original WaitGroup counter is stuck at `5`, `main()` blocks forever.
* Since all worker goroutines terminate and only `main()` remains blocked, the Go runtime detects this and crashes the application:
  `fatal error: all goroutines are asleep - deadlock!`

#### 3. Correct way to pass `sync.WaitGroup`:
Always pass `sync.WaitGroup` by **pointer** (`*sync.WaitGroup`), or access it through a closure instead of passing it as a parameter:

```go
package main

import (
	"fmt"
	"sync"
)

// FIX: Pass by pointer
func processTask(id int, wg *sync.WaitGroup) {
	defer wg.Done()
	fmt.Println("Processing task:", id)
}

func main() {
	var wg sync.WaitGroup
	for i := 1; i <= 5; i++ {
		wg.Add(1)
		go processTask(i, &wg) // Pass address of WaitGroup
	}
	wg.Wait()
	fmt.Println("All tasks processed.")
}
```
