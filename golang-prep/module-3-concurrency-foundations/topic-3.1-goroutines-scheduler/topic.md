# Topic 3.1: Goroutines & The Scheduler (Intro)

In Go, concurrency is a first-class citizen built directly into the language syntax and runtime. This topic covers the fundamentals of goroutines, how they differ from traditional Operating System (OS) threads, their memory footprint, context switching overhead, and how Go manages them at a system level.

---

## 1. Concepts & Theory

### 1.1 Concurrency vs. Parallelism
Before diving into goroutines, it is essential to distinguish between concurrency and parallelism:
* **Concurrency** is about **structure**. It is the composition of independently executing processes. A program is concurrent if it is designed to handle multiple tasks at the same time (e.g., handling incoming HTTP requests while querying a database).
* **Parallelism** is about **execution**. It is the simultaneous execution of multiple entities on multiple physical CPU cores.

Go's concurrency model allows you to write concurrent code that the Go runtime automatically maps onto multiple processor cores for parallel execution.

### 1.2 Goroutines vs. OS Threads
In traditional programming languages (like Java, C++, or Python), concurrency is achieved by mapping application-level threads directly to OS threads (often called a **1:1 mapping model**). Go uses an **M:N multiplexing model**, where `M` goroutines are multiplexed onto `N` OS threads.

Here is a system-level comparison between Goroutines and OS Threads:

| Attribute | OS Thread | Goroutine |
| :--- | :--- | :--- |
| **Creation Cost** | High (System call, memory allocation) | Low (User-space allocation, simple struct) |
| **Memory Footprint** | Large (typically 1MB - 8MB fixed stack) | Extremely Small (starts at 2KB, dynamic) |
| **Context Switch Cost** | High (~1 - 2 microseconds, CPU cache misses) | Low (~10 - 100 nanoseconds, user-space swap) |
| **Managed By** | OS Kernel | Go Runtime Scheduler |
| **Communication** | Shared memory (requires locks, mutexes) | Channels (Message passing) / Shared memory |

---

### 1.3 Memory Layout: Stack Allocation & Growth

#### OS Thread Stacks (Fixed & Guarded)
An OS thread is allocated a fixed-size stack (typically 1MB to 8MB) at creation. This size cannot change. 
* **Underflow/Overflow:** If a thread executes a deeply recursive function that exceeds the stack size, it crashes the program with a Stack Overflow. 
* **Guard Pages:** To detect this, the OS places a "guard page" (an unmapped memory page) at the end of the stack. If the stack pointer reaches the guard page, a hardware page fault is triggered, and the OS terminates the process.
* **Waste:** Allocating megabytes of stack space for threads that only execute simple tasks is highly wasteful, limiting the maximum number of concurrent threads to thousands on standard hardware.

#### Goroutine Stacks (Dynamic & Contiguous)
A Goroutine starts with an extremely lightweight stack of only **2KB**. 
* **Dynamic Growing:** If a goroutine exceeds its 2KB limit, the Go runtime allocates a new, contiguous memory block that is twice the size (e.g., 4KB, then 8KB, up to a maximum of 1GB on 64-bit systems), copies the old stack contents to the new block, updates pointers targeting the stack, and releases the old stack.
* **Stack Shrinking:** The Go garbage collector (GC) can shrink goroutine stacks during garbage collection cycles if they are under-utilized, freeing memory back to the heap.

```
+-----------------------------------+
|  Original 2KB Stack Block        |
|  [Func A Frame] [Func B Frame]    |
+-----------------------------------+
                 |
                 | (Exceeds capacity)
                 v
+-----------------------------------+-----------------------------------+
|  New Contiguous 4KB Stack Block                                       |
|  [Func A Frame] [Func B Frame] [New Func C Frame]                    |
+-----------------------------------+-----------------------------------+
```

---

### 1.4 Context Switch Cost

A **context switch** is the process of storing the state of a CPU so that execution can be resumed from the same point later.

#### OS Thread Context Switch (Kernel-Space)
When the OS switches threads:
1. It must transition from user-space to kernel-space.
2. It saves the CPU registers (registers, instruction pointer, stack pointer).
3. It updates the kernel scheduler state.
4. It restores the registers of the target thread.
5. It flushes or invalidates CPU caches, including the Translation Lookaside Buffer (TLB), which translates virtual memory addresses to physical memory addresses.
* **Cost:** This process requires ~1,000 to 2,000 nanoseconds and causes CPU cache misses.

#### Goroutine Context Switch (User-Space)
When the Go runtime scheduler switches goroutines:
1. It stays entirely in user-space (no kernel system calls).
2. It only needs to save and restore a small subset of registers (about 14 registers: execution registers, program counter, and stack pointer).
3. The CPU cache and TLB are not flushed because the underlying OS thread remains the same.
* **Cost:** This costs only ~10 to 100 nanoseconds, allowing Go programs to comfortably run hundreds of thousands of concurrent goroutines.

---

## 2. Edge Cases & Limitations

### 2.1 Stack Copying Pointer Gotchas
When a goroutine's stack grows, the runtime moves all local variables to a new memory address.
* If you store a pointer to a stack variable in another memory location (e.g., in a heap-allocated struct), Go's compiler and runtime must track and adjust these pointers during stack copy.
* Go's compiler performs **Escape Analysis** at compile-time to detect variables whose pointers might escape the stack. If a variable escapes (e.g., returned from a function or stored in a global variable), it is allocated directly on the **heap** instead of the stack. This mitigates pointer invalidation but adds garbage collection pressure.

### 2.2 Un-preemptible Goroutines (CPU-bound loops)
Historically (before Go 1.14), Go's scheduler was **cooperative**. A goroutine could only be preempted (suspended to let other goroutines run) at explicit checkpoints, such as:
* Channel operations (send/receive)
* System calls (I/O, file operations)
* Function calls (where the compiler inserts a stack-split check)

If a goroutine executed a tight, CPU-bound loop with no function calls:
```go
// Pre-Go 1.14: This could freeze the thread and block other goroutines forever
for {
    // No function calls, no I/O
}
```
* **Solution:** Go 1.14 introduced **asynchronous preemption**. The runtime uses OS signals (`SIGURG` on Unix-like systems) to interrupt executing threads and force a goroutine context switch, preventing CPU-bound loops from hogging resources.

---

## 3. Real-World Usage & Best Practices

### 3.1 Unbounded Goroutine Spawning (The "Fire-and-Forget" Anti-Pattern)
In production microservices, spawning an unconstrained number of goroutines is a common cause of memory exhaustion and cascade failures.

#### Scenario:
An HTTP API receives 10,000 requests/sec. On each request, it spawns a goroutine to log analytics to a remote database. If the database slows down, the goroutines block. The runtime keeps spawning new goroutines, consuming 2KB+ stack space and memory per request, eventually triggering an Out-Of-Memory (OOM) crash.

### 3.2 Concurrency Control (Semaphores & Worker Pools)
Always govern the maximum concurrency of your system using structured patterns:
* **Worker Pools:** A fixed number of worker goroutines consuming tasks from a shared channel.
* **Buffered Channel Semaphore:** Using a buffered channel of size `N` to rate-limit execution.

---

## 4. Code Examples: Good vs. Bad

### ❌ Bad: Spawning Unbounded Goroutines on External Requests
This code spawns a new goroutine for every request without limit. If `processPayload` blocks or slows down, memory usage will grow indefinitely.

```go
package main

import (
	"fmt"
	"net/http"
)

func handleRequest(w http.ResponseWriter, r *http.Request) {
	// Unbounded goroutine spawning - High vulnerability to OOM
	go func() {
		processPayload(r.Body)
	}()
	w.WriteHeader(http.StatusAccepted)
	fmt.Fprintln(w, "Processing...")
}

func processPayload(data interface{}) {
	// Simulated work (e.g., DB writes, API calls)
}

func main() {
	http.HandleFunc("/upload", handleRequest)
	http.ListenAndServe(":8080", nil)
}
```

### Good: Concurrency Limiting via Semaphore Channel
This approach uses a buffered channel as a semaphore to guarantee that no more than `100` payloads are processed concurrently. Any extra requests will block until a slot becomes free, protecting the server's memory.

```go
package main

import (
	"fmt"
	"net/http"
)

// Limit concurrency to 100 concurrent workers
var maxConcurrency = 100
var semaphore = make(chan struct{}, maxConcurrency)

func handleRequest(w http.ResponseWriter, r *http.Request) {
	// Non-blocking write check or blocking wait
	select {
	case semaphore <- struct{}{}:
		// Acquired semaphore slot, execute concurrently
		go func() {
			defer func() { <-semaphore }() // Release slot when finished
			processPayload(r.Body)
		}()
		w.WriteHeader(http.StatusAccepted)
		fmt.Fprintln(w, "Processing...")
	default:
		// Queue is full, return 429 Too Many Requests to preserve server health
		w.WriteHeader(http.StatusTooManyRequests)
		fmt.Fprintln(w, "Server busy. Try again later.")
	}
}

func processPayload(data interface{}) {
	// Process the resource-heavy request
}

func main() {
	http.HandleFunc("/upload", handleRequest)
	http.ListenAndServe(":8080", nil)
}
```
