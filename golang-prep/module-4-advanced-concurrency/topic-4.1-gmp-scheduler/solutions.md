# Solutions: The GMP Scheduler Model

Here are the step-by-step solutions, runtime analyses, and explanations for the Topic 4.1 exercises.

---

## 🧠 Conceptual Questions

### Solution 1: GMP Status Mapping

#### 1. G waiting to read from an open channel:
* **G:** Set to state `_Gwaiting`. It is stored in the channel's `recvq` wait queue (wrapped in a `sudog` struct). It is not attached to any P or M.
* **M:** Active, running other runnable goroutines from P's Local Run Queue (LRQ).
* **P:** Active, associated with M.

#### 2. G executing a CPU-intensive math formula:
* **G:** Set to state `_Grunning`.
* **M:** Active, executing G on the CPU core.
* **P:** Active, associated with M (holding the execution context).

#### 3. G calling blocking CGO library (C-sleep):
* **G:** Set to state `_Gwaiting` (or runtime equivalent for system calls).
* **M:** Blocked inside the OS kernel executing the C function.
* **P:** Detached from M (via P-Handoff) and assigned to a different thread M's run-loop.

#### 4. G calling `net.Dial` (waiting for network socket):
* **G:** Set to state `_Gwaiting` and registered with the runtime **Network Poller**.
* **M:** Active, running other goroutines on P.
* **P:** Active, associated with M.

---

### Solution 2: Work-Stealing Sequence

#### 1. Step-by-Step Sequence for Idle P1 (Worker M1):
1. **Global Queue check (Starvation prevention):** If the schedule tick counter is a multiple of 61, check the Global Run Queue (GRQ) first.
2. **Local Queue check:** Check P1's Local Run Queue (LRQ) — empty.
3. **Work Stealing:**
   - M1 chooses a random Processor P (e.g., P3) from the global P list.
   - It acquires a lock-free view of P3's LRQ. If Gs are present, it locks and steals **half** of P3's queue (e.g., if P3 has 4 Gs, it takes 2, putting 1 onto CPU immediately and 1 in P1's LRQ).
   - If P3 is empty, it chooses another random P and repeats the attempt up to 4 times.
4. **Global Queue check (Fallback):** Check the GRQ.
5. **Network Poller check:** Check the Network Poller for completed asynchronous I/O events.
6. **Sleep:** If all the above fail, M1 dissociates from P1, puts P1 in the idle P list, and puts itself (M1) to sleep.

#### 2. Why Random Selection?
If M1 checked Ps sequentially (P2, P3, P4...), all idle threads would target P2 first, causing extreme lock contention on P2's LRQ atomic pointers. Random selection distributes stealing attempts evenly, minimizing CPU bus lock conflicts.

---

## 🛠️ Practical Problems

### Solution 3: Observing OS Thread Counts

#### 1. Code Implementation:
```go
package main

import (
	"fmt"
	"os"
	"runtime"
	"sync"
	"syscall"
)

func main() {
	var wg sync.WaitGroup
	const numGoroutines = 1000

	fmt.Printf("GOMAXPROCS: %d\n", runtime.GOMAXPROCS(0))
	fmt.Printf("Initial Goroutines: %d\n", runtime.NumGoroutine())

	for i := 0; i < numGoroutines; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			// Force synchronous blocking call using syscall
			// Reading from /dev/null blocks thread context in kernel
			fd, _ := syscall.Open("/dev/null", syscall.O_RDONLY, 0)
			defer syscall.Close(fd)

			var buf [1]byte
			syscall.Read(fd, buf[:])
		}(i)
	}

	fmt.Printf("Active Goroutines during syscalls: %d\n", runtime.NumGoroutine())
	wg.Wait()
}
```

#### 2. How to check OS threads on macOS:
You can find the thread count of your compiled Go binary using the terminal command `ps` or `top`:
```bash
# Get thread count of a process by name
ps -M $(pgrep <binary_name>) | wc -l
```
Or view the `TH` (threads) column in `top`.

#### 3. Why Thread Count Exceeds `GOMAXPROCS`:
`GOMAXPROCS` limits the number of threads *actively running Go code concurrently*. When a thread (M) makes a blocking system call, it halts inside the kernel and cannot run other Gs. The runtime detaches P from it and spawns/allocates a new thread M to run the remaining Gs. Thus, under heavy concurrent syscalls, the runtime allocates up to 1,000 threads (capped at a default maximum of 10,000 by the runtime).

---

### Solution 4: Manual GOMAXPROCS Allocation

#### Benchmark Code:
```go
package main

import (
	"runtime"
	"testing"
)

func calculatePrimes(n int) int {
	count := 0
	for i := 2; i < n; i++ {
		isPrime := true
		for j := 2; j*j <= i; j++ {
			if i%j == 0 {
				isPrime = false
				break
			}
		}
		if isPrime {
			count++
		}
	}
	return count
}

func BenchmarkPrimes(b *testing.B) {
	for _, procs := range []int{1, 2, 4, 8} {
		b.Run(string(rune(procs)), func(b *testing.B) {
			runtime.GOMAXPROCS(procs)
			b.ResetTimer()
			b.RunParallel(func(pb *testing.PB) {
				for pb.Next() {
					calculatePrimes(1000)
				}
			})
		})
	}
}
```

#### Expected Trends on a 4-core machine:
* **GOMAXPROCS=1:** CPU usage hovers around 100% (1 core). Throughput is restricted since only one task runs at a time.
* **GOMAXPROCS=2:** Performance roughly doubles, CPU usage hits 200%.
* **GOMAXPROCS=4:** Performance peaks. All 4 physical cores are saturated (400% CPU).
* **GOMAXPROCS=8:** Performance plateaus or degrades slightly. Since there are only 4 physical cores, the OS must multiplex 8 threads across them, causing OS-level thread context switches and cache thrashing.

---

## 💼 Scenario-Based Questions

### Solution 5: The Kubernetes CPU Throttling Mystery

#### 1. Default `GOMAXPROCS` inside the Pod:
By default, the Go runtime calls `sysconf(_SC_NPROCESSORS_ONLN)` to determine CPU count. Since the container runs on a 64-core host, it defaults `GOMAXPROCS` to **64**, completely ignoring the container's 2-CPU resource limit.

#### 2. Why CFS Throttling Occurs:
* Go's scheduler spawns 64 logical contexts (P) and threads (M) designed to saturate 64 cores.
* Kubernetes limits CPU usage using the CFS quota system (based on 100ms periods). A limit of 2 CPUs means the container gets 200ms of CPU execution time every 100ms.
* With 64 threads actively scheduling, spawning, and running work, the container consumes its 200ms allowance within the first **3.1ms** of the cgroup period ($200\text{ms} / 64\text{ threads} = 3.125\text{ms}$).
* The OS CFS throttles the container for the remaining **96.8ms** of the period, freezing the application. This cycle repeats, resulting in severe latency spikes.

#### 3. Resolution:
Install the `automaxprocs` library in the gateway entry file:
```go
import _ "go.uber.org/automaxprocs"
```
This library reads the container's `/sys/fs/cgroup/cpu/cpu.cfs_quota_us` and `/sys/fs/cgroup/cpu/cpu.cfs_period_us` files, dividing quota by period to automatically set `GOMAXPROCS=2`, preventing CFS throttling.

---

### Solution 6: Anatomy of a Syscall Handoff

#### 1. G1 Status:
* G1 transitions from `_Grunning` to `_Gwaiting` (specifically `_Gsyscall`). It remains bound to M1.

#### 2. M1 Status:
* M1 executes the system read call, transitions to kernel space, and blocks waiting for the hard disk controller.

#### 3. P1 Status:
* The Go runtime monitor thread (`sysmon`) detects that P1 is blocked in a system call.
* It invokes `handoffp(p1)`. P1 is detached from M1.
* P1's association is transferred to a new or idle thread M2.
* M2 begins executing the remaining runnable goroutines (G2, G3) queued in P1's LRQ.

#### 4. Resuming execution:
* When the OS finishes reading, M1 wakes up and returns to user-space.
* M1 attempts to lock and associate itself with an idle logical Processor P.
* **Scenario A: If an idle P is found:** M1 binds to it, G1 status is changed to `_Grunnable`, and execution resumes on that P.
* **Scenario B: If no P is idle:** M1 cannot run Go code. It places G1 into the Global Run Queue (GRQ), changes G1's status to `_Grunnable`, dissociates from G1, and puts itself (M1) to sleep.
