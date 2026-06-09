# Topic 3.4: The `sync` Package

While Go encourages message passing via channels, shared memory concurrency remains crucial for high-performance systems. The standard library's `sync` package provides low-level synchronization primitives. 

This topic covers `sync.Mutex`, `sync.RWMutex`, `sync.WaitGroup`, and `sync.Once`, including their system-level internals, copy constraints, and production edge cases.

---

## 1. Concepts & Theory

### 1.1 `sync.Mutex` (Mutual Exclusion Lock)
A `sync.Mutex` protects shared resources from concurrent writes and concurrent read-writes. It has two methods: `Lock()` and `Unlock()`.

#### Mutex Internals: Normal vs. Starvation Mode
To prevent starvation, the Go runtime operates `sync.Mutex` in two distinct modes:
1. **Normal Mode:**
   - Waiters are queued in a FIFO list.
   - However, a blocked goroutine that is woken up does not automatically get the lock. It must compete with newly arriving goroutines that are already running on the CPU.
   - Because newly running goroutines are already scheduled on the CPU, they have a massive advantage, which yields much higher overall throughput.
2. **Starvation Mode:**
   - If a waiter fails to acquire the lock for more than **1 millisecond**, the Mutex switches to **Starvation Mode**.
   - In this mode, the lock is handed off **directly** from the releasing goroutine to the waiter at the head of the queue.
   - Newly arriving goroutines do not attempt to acquire the lock; they are placed at the end of the wait queue immediately.
   - Once the wait queue is empty or the last waiter's wait time is less than 1ms, the Mutex switches back to normal mode.

---

### 1.2 `sync.RWMutex` (Reader-Writer Mutex)
A `sync.RWMutex` allows **multiple readers** or **a single writer** to hold the lock, but never both.
* **Methods:**
  - `Lock()` / `Unlock()` for Writers.
  - `RLock()` / `RUnlock()` for Readers.

#### When to use:
* Use `sync.RWMutex` when reads heavily outnumber writes (e.g., a shared cache or database metadata configuration).
* *System Note:* If the write volume is high, an `RWMutex` actually performs *worse* than a standard `Mutex` due to the overhead of managing atomic reader counts.

---

### 1.3 `sync.WaitGroup` (Synchronization Groups)
A `sync.WaitGroup` waits for a collection of goroutines to finish.
* **`Add(delta int)`:** Increments the internal counter.
* **`Done()`:** Decrements the internal counter (shorthand for `Add(-1)`).
* **`Wait()`:** Blocks the calling goroutine until the counter reaches `0`.

#### WaitGroup Internals:
A `WaitGroup` contains a state value representing both the counter and the waiter count, along with a sema variable. Under the hood, Go performs atomic operations on a 64-bit alignment to ensure updates are synchronized across CPU cores.

---

### 1.4 `sync.Once` (Single Execution)
`sync.Once` guarantees that a function is executed exactly once, regardless of how many goroutines call it.

#### Once Internals: Double-Checked Locking
`sync.Once` uses a fast-path flag followed by a slow-path mutex check:
```go
type Once struct {
    done uint32
    m    Mutex
}
```
1. **Fast Path:** It uses `atomic.LoadUint32(&o.done)` to check if the function has already run. If `1`, it returns immediately (very fast).
2. **Slow Path:** If `0`, it acquires the mutex `m`. It checks the flag again under the mutex (double-check). If still `0`, it executes the function and sets `done = 1` atomically, then releases the mutex.

---

## 2. Edge Cases & Limitations

### 2.1 Copying Locks (The `go vet` Trap)
Primitives in the `sync` package must **never be copied**. If you copy a struct containing a `Mutex` or a `WaitGroup`, the lock state (whether it is locked, the number of waiters) is duplicated. This leads to undefined behavior, data races, and deadlocks.

```go
type Cache struct {
    mu sync.Mutex // Never pass Cache by value!
    data map[string]string
}
```
* **Tooling:** Run `go vet` on your codebase. The compiler's vet checker has a `copylocks` diagnostic that warns you of lock copying.
* **Solution:** Always pass pointers to structs containing sync primitives, or make the mutex a pointer field.

---

### 2.2 Non-Reentrant Mutexes
Unlike Java or C# where locks are reentrant (a thread can acquire the same lock multiple times recursively without blocking), Go's `sync.Mutex` is **non-reentrant**.

```go
func RecursiveFunc() {
    mu.Lock()
    defer mu.Unlock()
    
    // PANIC / DEADLOCK:
    // Attempting to lock again inside the same goroutine will freeze forever
    RecursiveFunc() 
}
```

---

## 3. Real-World Usage & Best Practices

### 3.1 Struct-Level Lock Encapsulation
Always keep the mutex close to the data it protects. The standard pattern is to define the mutex directly above the field(s) it guards, and defer the unlock immediately after locking:

```go
type SafeCounter struct {
    mu    sync.Mutex // Guards counter
    count int
}

func (c *SafeCounter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock() // Releases lock even if panic occurs
    c.count++
}
```

---

## 4. Code Examples: Good vs. Bad

### ❌ Bad: Copying Mutex by Value
In this code, the receiver is not a pointer (`(c SafeCounter)`), so calling `Increment` copies the entire struct—including the Mutex—by value. The lock is never updated on the original instance.

```go
package main

import (
	"fmt"
	"sync"
)

type SafeCounter struct {
	mu    sync.Mutex
	count int
}

// BAD: c is passed by value (copies the Mutex)
func (c SafeCounter) Increment() {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.count++
}

func main() {
	c := SafeCounter{}
	c.Increment()
	c.Increment()
	// Count will still be 0!
	fmt.Println("Count:", c.count) 
}
```

### Good: Encapsulated Pointer Receivers
This code uses pointer receivers to ensure the same Mutex instance is targeted, preventing copy issues.

```go
package main

import (
	"fmt"
	"sync"
)

type SafeCounter struct {
	mu    sync.Mutex
	count int
}

// Good: c is a pointer receiver
func (c *SafeCounter) Increment() {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.count++
}

func (c *SafeCounter) Value() int {
	c.mu.Lock()
	defer c.mu.Unlock()
	return c.count
}

func main() {
	c := &SafeCounter{}
	var wg sync.WaitGroup

	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			c.Increment()
		}()
	}

	wg.Wait()
	fmt.Println("Count:", c.Value()) // Count: 1000
}
```

### ❌ Bad: Calling `WaitGroup.Add()` inside Goroutine
This is a common race condition bug. If the scheduler delays starting the goroutines, `wg.Wait()` might execute before any `wg.Add(1)` has run, causing the program to exit early.

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var wg sync.WaitGroup

	for i := 0; i < 3; i++ {
		go func(id int) {
			wg.Add(1) // BAD: Race condition - Add is inside the goroutine!
			defer wg.Done()
			fmt.Println("Worker:", id)
		}(i)
	}

	wg.Wait() // May exit immediately with no workers running
	fmt.Println("Done")
}
```

### Good: Calling `WaitGroup.Add()` in Coordinator Scope
Always call `wg.Add()` in the coordinator thread *before* launching the goroutine.

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var wg sync.WaitGroup

	for i := 0; i < 3; i++ {
		wg.Add(1) // Good: Called in coordinator scope before spawning
		go func(id int) {
			defer wg.Done()
			fmt.Println("Worker:", id)
		}(i)
	}

	wg.Wait() // Guaranteed to wait for all 3 workers
	fmt.Println("Done")
}
```
