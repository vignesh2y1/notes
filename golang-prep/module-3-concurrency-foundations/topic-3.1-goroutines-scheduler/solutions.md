# Solutions: Goroutines & The Scheduler (Intro)

Here are the step-by-step solutions, code implementations, and analytical explanations for the Topic 3.1 exercises.

---

## 🧠 Conceptual Questions

### Solution 1: Memory Scaling Math

#### 1. Java Thread Limit Calculation:
* **Available Memory:** $16\text{ GB} - 2\text{ GB (Reserved for OS/Runtime)} = 14\text{ GB} = 14,680,064\text{ KB}$.
* **Thread Stack Size:** $1\text{ MB} = 1024\text{ KB}$.
* **Theoretical Maximum:**
  $$\frac{14,680,064\text{ KB}}{1024\text{ KB}} \approx 14,336 \text{ threads}$$
* *Real-world note:* Operating systems typically impose hard limits on the maximum number of threads (e.g., via `sysctl` limits like `kern.num_threads` or `ulimit -u`) well before physical memory is exhausted, typically failing at around 5,000 to 10,000 threads.

#### 2. Go Goroutine Limit Calculation:
* **Goroutine Starting Stack Size:** $2\text{ KB}$.
* **Theoretical Maximum:**
  $$\frac{14,680,064\text{ KB}}{2\text{ KB}} \approx 7,340,032 \text{ goroutines}$$
* *Real-world note:* Go handles millions of inactive or lightly-loaded goroutines easily, making it highly scale-efficient compared to thread-bound languages.

#### 3. Growth Safety Mechanism:
If a goroutine needs more than 2KB, Go uses **contiguous stack allocation**:
1. When a function is invoked, the compiler checks the stack frame requirements.
2. If the current stack capacity is insufficient, the runtime is invoked to allocate a new contiguous stack block (double the previous size).
3. The runtime copies the old stack contents to the new block.
4. Pointers to variables on the old stack are adjusted to point to their new addresses in the new block.
5. The old stack memory is freed, and execution resumes.

---

### Solution 2: Thread vs. Goroutine Context Switching

#### 1. TLB Flush on Thread Switch:
The Translation Lookaside Buffer (TLB) caches virtual-to-physical address mappings. 
* OS threads can belong to different processes. Even when switching threads within the same process, the operating system kernel is responsible for managing thread scheduling.
* When scheduling across processes, the OS must switch the entire virtual memory address space. This requires invalidating (flushing) the TLB because the virtual address mappings are completely different for the new process. This flush causes memory address lookup delays (cache misses) on the CPU.

#### 2. Goroutine Context Switch Speed:
A Goroutine context switch is managed entirely by the Go runtime within the same application process.
* Because the OS threads running the Go scheduler do not change, the CPU virtual address space remains identical.
* The TLB does **not** need to be flushed. The CPU continues to resolve memory addresses at hardware cache speed, avoiding costly context restore delays.

#### 3. Management Layer:
* **Goroutine Context Switch:** Managed in **User-Space** by the Go runtime scheduler code. CPU registers are saved/restored by Go assembler code.
* **OS Thread Context Switch:** Managed in **Kernel-Space** by the Operating System Kernel. It involves CPU interrupts, privileged kernel instruction sets, and CPU state register swaps.

---

## 🛠️ Practical Problems

### Solution 3: Dynamic Stack Growth Demonstration

#### Code Implementation:
The following program prints the memory address of a local variable at each level of deep recursion. You will notice that at certain points, the address of the variable shifts dramatically as the stack is reallocated to a new contiguous memory space.

```go
package main

import "fmt"

func recursiveFunc(depth int, previousAddr *int) {
	var x int = depth // Stack variable
	currentAddr := &x

	// Print address at intervals to witness stack movement
	if depth%100 == 0 {
		diff := 0
		if previousAddr != nil {
			// Calculate numeric difference between pointer addresses
			diff = int(uintptr(previousAddr) - uintptr(currentAddr))
		}
		fmt.Printf("Depth: %4d | Var Address: %p | Diff: %d bytes\n", depth, currentAddr, diff)
	}

	if depth <= 0 {
		return
	}

	// Recursive call
	recursiveFunc(depth-1, currentAddr)
}

func main() {
	recursiveFunc(1000, nil)
}
```

#### Explanation:
* The address `&x` is printed. If you run this program, you will notice that the memory address starts out at one range, and after a few hundred iterations, the address jumps to a completely different location, and the `Diff` becomes very large (reflecting a stack relocation rather than simple frame-offset offset).
* Note: The Go compiler's escape analysis is smart. If we pass `&x` to a function that escapes (like `fmt.Printf`), the compiler might attempt to escape it to the heap. To prevent this, we calculate the difference using `uintptr` arithmetic locally before printing, keeping the variable on the stack.

---

### Solution 4: Concurrency Limiter (Semaphore)

#### Code Implementation:
```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func worker(id int, sem chan struct{}, wg *sync.WaitGroup) {
	defer wg.Done()

	// Acquire slot: write into channel. If full, blocks.
	sem <- struct{}{}
	fmt.Printf("[Worker %2d] Starting task...\n", id)

	// Simulate work
	time.Sleep(100 * time.Millisecond)

	fmt.Printf("[Worker %2d] Finished task.\n", id)
	// Release slot: read out of channel.
	<-sem
}

func main() {
	const totalTasks = 50
	const maxConcurrency = 5

	var wg sync.WaitGroup
	// Semaphore channel with buffer size of 5
	sem := make(chan struct{}, maxConcurrency)

	for i := 1; i <= totalTasks; i++ {
		wg.Add(1)
		go worker(i, sem, &wg)
	}

	// Wait for all goroutines to finish
	wg.Wait()
	fmt.Println("All 50 tasks finished successfully.")
}
```

---

## 💼 Scenario-Based Questions

### Solution 5: The "Silent Leak" HTTP Client

#### 1 & 2. The Bug & Goroutine Behavior:
* When `time.After(500 * time.Millisecond)` triggers first, `fetchUserProfile` returns `"Default Guest"`.
* The spawned goroutine is still running. After 2 seconds, it finishes the sleep and attempts to send `"User Profile Data"` to the channel `ch`:
  ```go
  ch <- "User Profile Data"
  ```
* Since `ch` is an **unbuffered channel** and the main goroutine has already returned (and is no longer listening to `ch`), there is no receiver.
* The spawned goroutine blocks **indefinitely** on this send line. It will remain locked in memory forever.

#### 3. Memory Footprint Impact:
Each blocked goroutine retains its 2KB stack, along with all variables captured in its closure (closure references, strings, etc.).
Under high traffic (e.g., thousands of requests per minute), these leaked goroutines stack up, resulting in a slow, continuous memory leak that eventually crashes the application via Out-Of-Memory (OOM).

#### 4. The Fix:
The easiest fix is to make the channel a **buffered channel of capacity 1**. This allows the spawned goroutine to write its result to the buffer and exit immediately, even if no one is listening.

```go
package main

import (
	"fmt"
	"net/http"
	"time"
)

func fetchUserProfile() string {
	// FIX: Use a buffered channel of size 1
	ch := make(chan string, 1) 

	go func() {
		time.Sleep(2 * time.Second)
		ch <- "User Profile Data" // Will not block, writes to buffer and exits
	}()

	select {
	case res := <-ch:
		return res
	case <-time.After(500 * time.Millisecond):
		return "Default Guest"
	}
}
```

---

### Solution 6: The Un-preemptible Spin-Lock

#### 1. Behavior on Go 1.13 with `GOMAXPROCS=1`:
* The goroutine running `spin()` will execute the `for` loop indefinitely.
* Because the loop is tight and contains no function calls, system calls, or channel operations, Go 1.13's cooperative scheduler will never yield execution.
* The single OS thread allocated to the process will be 100% occupied by `spin()`.
* **Result:** No other goroutine (including the one that is supposed to set `ready = true` or handle garbage collection) will ever run. The application hangs completely.

#### 2. Behavior on Go 1.14+:
In Go 1.14+, **asynchronous preemption** is enabled:
1. The Go runtime includes a background monitoring thread (`sysmon`) that runs periodically without a P.
2. If `sysmon` detects that a goroutine has been running on the same P for more than 10ms, it sends an OS signal (`SIGURG`) to the OS thread running that P.
3. The OS interrupts the thread and invokes the runtime's signal handler.
4. The signal handler saves the goroutine's registers and modifies its instruction pointer to call `gopreempt_m()`, suspending the spinning goroutine.
5. The scheduler runs other available goroutines on the P, allowing the goroutine that sets `ready = true` to run and break the spin-lock.
