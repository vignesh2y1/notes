# Topic 4.1: The GMP Scheduler Model

Go's runtime scheduler is one of the most sophisticated parts of the Go runtime engine. Operating in user-space, it multiplexes thousands of goroutines onto a small pool of operating system threads.

This topic covers the **GMP model** (Goroutine, Machine, Processor), run queues, work-stealing algorithms, syscall handling, the network poller, and containerized deployment considerations.

---

## 1. Concepts & Theory

### 1.1 The Entities: G, M, and P

The scheduler uses three core structures:

```
              +-----------------------+
              |   Global Run Queue    |
              +-----------+-----------+
                          |
            +-------------+-------------+
            |                           |
    +-------v-------+           +-------v-------+
    |  Processor P  |           |  Processor P  | (GOMAXPROCS)
    |  [Local Queue]|           |  [Local Queue]|
    +-------+-------+           +-------+-------+
            |                           |
    +-------v-------+           +-------v-------+
    |   Thread M    |           |   Thread M    | (OS Thread)
    +-------+-------+           +-------+-------+
            |                           |
    +-------v-------+           +-------v-------+
    |  Goroutine G  |           |  Goroutine G  | (Executing)
    +---------------+           +---------------+
```

#### 1. **G (Goroutine)**
* Represents the goroutine runtime instance.
* Contains the execution stack (starting at 2KB), program counter (PC), and local registers.
* Status states include: `_Grunnable` (in run queue), `_Grunning` (executing on an M), `_Gwaiting` (blocked on channel/lock/net I/O).

#### 2. **M (Machine / OS Thread)**
* Represents a physical Operating System thread.
* Created and scheduled by the OS kernel.
* To execute Go code, an M **must** be paired with a logical Processor P.
* The number of active threads can exceed `GOMAXPROCS` if threads block in system calls.

#### 3. **P (Processor / Execution Context)**
* Represents the logical resources required to execute Go code (e.g., local memory allocators, run queues).
* The number of Ps is strictly bounded by the environment variable `GOMAXPROCS` (defaults to the number of physical CPU cores).
* If `GOMAXPROCS` is 4, at most 4 threads (M) can execute Go code concurrently.

---

### 1.2 Local vs. Global Run Queues

To reduce lock contention across CPU threads, Go splits its run queues:
* **Local Run Queue (LRQ):** 
  - Every logical Processor P has its own LRQ.
  - It holds up to 256 runnable goroutines.
  - Access to the LRQ is lock-free (using atomic read/write pointers) because only the P's assigned M accesses it.
* **Global Run Queue (GRQ):**
  - A single queue shared by all Ps in the system.
  - Used for overflow (when a P's LRQ exceeds 256 items) or when goroutines are yielded.
  - Accessing the GRQ requires acquiring a global mutex lock, making it slow.

---

### 1.3 The Scheduling Cycle & Work Stealing

When an OS thread M is paired with a P and finishes running its current goroutine, it searches for a new `_Grunnable` G to execute using the following algorithm:

1. **Starvation Prevention Check:**
   - Every 61 scheduling ticks, M checks the **Global Run Queue (GRQ)**. This prevents starvation of goroutines placed in the global queue.
2. **Local Queue Check:**
   - M looks at its own P's **Local Run Queue (LRQ)**. If a G is ready, it runs it.
3. **Work Stealing:**
   - If P's LRQ is empty, M attempts to **steal** work.
   - It randomly selects another Processor P and attempts to steal **half** of its LRQ to populate its own queue.
4. **Fallback Check:**
   - If work stealing fails, M checks the GRQ.
   - If still empty, it checks the **Network Poller** for completed asynchronous I/O events.
5. **Sleep:**
   - If no work is found, the thread M releases its P and goes to sleep (becomes idle).

---

### 1.4 Blocking Calls: Syscalls vs. Network Poller

#### 1. Asynchronous I/O (The Network Poller)
When a Goroutine blocks on network socket reads/writes, Go does **not** block the underlying OS thread (M).
* The G is marked `_Gwaiting` and registered with the **Network Poller** (which uses OS-native async systems like `epoll` on Linux, `kqueue` on macOS).
* The thread M continues running other goroutines from P's local queue.
* When the I/O event is ready, the Netpoller wakes G up, and it is placed back in a run queue.

#### 2. Synchronous System Calls (File I/O, CGO)
If a Goroutine calls a blocking OS system call (like `syscall.Read` for local files, which does not support asynchronous polling, or CGO calls):
* The thread M blocks in the OS kernel.
* To prevent the execution context from halting, the Go runtime performs a **P-Handoff**:
  - The logical Processor P is detached from the blocked thread M.
  - P is assigned to another idle thread M (or a new thread is spawned).
  - The new thread M continues executing the remaining goroutines in P's LRQ.
* Once the syscall on the original thread M finishes, G attempts to acquire an idle P to resume execution. If no Ps are available, G is moved to the Global Run Queue, and thread M goes to sleep.

```
Blocking Syscall:
[M1 (blocked in kernel)] <--- [G1]
  \ (Detaches)
   v
[P1] <--- Assigned to ---> [M2 (new/idle thread)] <--- [G2, G3...]
```

---

## 2. Edge Cases & Limitations

### 2.1 Thread Explosion (CGO / Blocking Syscalls)
While `GOMAXPROCS` limits the number of threads actively executing Go code, it does **not** limit the number of threads blocked in system calls.
* If your application frequently triggers blocking file system operations or CGO bindings under high concurrency, the Go runtime will spawn a new OS thread (M) for each blocked call.
* This can cause "thread explosion" (thousands of OS threads), which degrades system performance due to memory consumption and OS context switching.
* **Solution:** Limit concurrency at the application layer using worker pools, or use asynchronous file libraries when possible.

---

## 3. Real-World Usage & Best Practices

### 3.1 Docker Container CPU Limits & `GOMAXPROCS`
In production Kubernetes/Docker environments, containers are assigned fractional CPU limits (e.g., `cpu-limit: 2`).
* **The Problem:** By default, Go's runtime queries the host operating system's CPU count, not the container's quota. If the host has 64 cores, Go sets `GOMAXPROCS=64`.
* **The Symptom:** Since the container is throttled to 2 CPU cores by the OS CFS (Completely Fair Scheduler), but Go spawns scheduler cycles for 64 logical contexts, the OS frequently throttles the container. This causes severe latency spikes and CPU throttling.
* **The Fix:** Import `go.uber.org/automaxprocs` in your `main.go`. It queries `/sys/fs/cgroup` and automatically configures `GOMAXPROCS` to match the container's CPU quota.

```go
package main

import (
	_ "go.uber.org/automaxprocs" // Automatically sets GOMAXPROCS to container limits
)

func main() {
	// Your application entry point
}
```

---

## 4. Code Examples: Good vs. Bad

### ❌ Bad: Running Blocking CGO / File I/O on High-Concurrency Workers
Spawning thousands of goroutines that run heavy, synchronous file writes causes the Go runtime to spawn a massive number of OS threads to manage the handoffs.

```go
package main

import (
	"os"
	"sync"
)

func writeDataSync(data []byte) {
	// Synchronous blocking system call (cannot be multiplexed by netpoller)
	f, _ := os.OpenFile("log.txt", os.O_APPEND|os.O_WRONLY|os.O_CREATE, 0644)
	defer f.Close()
	f.Write(data)
}

func main() {
	var wg sync.WaitGroup
	// Spawning 5000 concurrent goroutines doing blocking writes
	for i := 0; i < 5000; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			writeDataSync([]byte("analytics data\n"))
		}()
	}
	wg.Wait()
}
```

### Good: Concurrency-Governed File Writing (Worker Pool)
We bound the maximum concurrency of blocking file operations to a fixed pool of workers. This prevents thread allocation spikes (M-handoffs) in the runtime.

```go
package main

import (
	"os"
	"sync"
)

type WriteJob struct {
	data []byte
}

func fileWorker(jobs <-chan WriteJob, wg *sync.WaitGroup) {
	defer wg.Done()
	// Open file once per worker thread to optimize handles
	f, _ := os.OpenFile("log.txt", os.O_APPEND|os.O_WRONLY|os.O_CREATE, 0644)
	defer f.Close()

	for job := range jobs {
		f.Write(job.data)
	}
}

func main() {
	const numWorkers = 10
	jobs := make(chan WriteJob, 1000)
	var wg sync.WaitGroup

	// Start a fixed pool of workers
	for i := 0; i < numWorkers; i++ {
		wg.Add(1)
		go fileWorker(jobs, &wg)
	}

	// Push jobs to queue
	for i := 0; i < 5000; i++ {
		jobs <- WriteJob{data: []byte("analytics data\n")}
	}
	close(jobs)

	wg.Wait() // Wait for pool to empty
}
```
