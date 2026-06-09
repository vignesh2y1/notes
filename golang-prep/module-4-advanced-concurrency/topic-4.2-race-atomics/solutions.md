# Solutions: Race Conditions & Atomics

Here are the step-by-step solutions, explanations, and code implementations for the Topic 4.2 exercises.

---

## 🧠 Conceptual Questions

### Solution 1: Data Race vs. Race Condition

#### 1. Technical Difference:
* A **data race** is a low-level memory access conflict (unsynchronized concurrent access to a memory location where at least one is a write). It is defined by Go's language memory model specifications.
* A **race condition** is a high-level logical flaw in application execution sequence. It occurs when the correctness of a program's output depends on the relative timing or interleaving of concurrent execution threads.
* **Can you have a race condition with no data races?**
  Yes. Suppose you have a bank transfer protected by mutex locks:
  ```go
  func (b *Bank) Transfer(from, to string, amount int) {
      b.mu.Lock()
      defer b.mu.Unlock()
      if b.balances[from] >= amount {
          b.balances[from] -= amount
          b.balances[to] += amount
      }
  }
  ```
  If thread 1 checks `Transfer(A, B, 100)` and thread 2 checks `Transfer(A, C, 100)` when A only has 100 credits, one will succeed and the other will fail. The result depends on which thread executes first (a race condition), but because every operation is protected by a mutex, there are **no data races** (no concurrent unsynchronized memory operations).

#### 2. Why Go Maps Panic on Concurrent Writes:
* Go maps are complex internal data structures containing bucket directories, pointers to hash maps, overflows, and slice indices. If a map is mutated while being read/written, pointers can become corrupted, leading to wild memory pointers (security exploits) and segmentation faults.
* To prevent this, the Go runtime adds a internal status flag (`hashWriting`) during operations. If a read/write occurs while `hashWriting` is set, the runtime raises a fatal, un-recoverable panic (`fatal error: concurrent map writes`).
* Go slices are simple headers containing a backing pointer, length, and capacity. If concurrently mutated, the pointer remains valid (it just points to a contiguous block), but elements will get scrambled (lost updates) without causing a runtime panic.

---

### Solution 2: The Compare-And-Swap (CAS) Hardware Step

#### 1. CPU CAS Logic:
```
function CompareAndSwap(address, expectedVal, newVal) -> bool:
    acquire hardware bus lock on cache line containing address (atomic instruction)
    currentVal = read memory at address
    if currentVal == expectedVal:
        write newVal to address
        release hardware bus lock
        return true
    else:
        release hardware bus lock
        return false
```

#### 2. Why "Lock-Free" at the Application Level:
* It is "lock-free" because it does not use software locks (mutexes) that suspend/park threads. If a CAS fails due to contention, the thread does not block in the OS; it simply loops (spins) and tries again. No OS thread context switching occurs.

#### 3. The ABA Problem:
* Suppose G1 reads value `A` from address `X`.
* G2 interrupts G1, changes the value at `X` to `B`, and then changes it back to `A`.
* When G1 resumes and executes `CompareAndSwap(X, A, C)`, the CAS succeeds because the value is still `A`.
* However, G1 is unaware that the memory state has mutated. In systems where memory is recycled (like custom memory allocators or stacks), G1 might access a recycled object, causing memory corruption.
* **Relation to CAS:** CAS only checks *value equality*, not *mutation history*. Go resolves this via garbage collection (which prevents memory from being recycled while pointers exist).

---

## 🛠️ Practical Problems

### Solution 3: Lock-Free Stack Pop

To implement `Pop`, we read the current top node, fetch the next node, and attempt to swap top with next. If another thread mutated top in the meantime, the CAS fails, and we loop to retry.

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
)

type Node struct {
	value int
	next  *Node
}

type LockFreeStack struct {
	top atomic.Pointer[Node]
}

func (s *LockFreeStack) Push(val int) {
	newNode := &Node{value: val}
	for {
		currentTop := s.top.Load()
		newNode.next = currentTop
		if s.top.CompareAndSwap(currentTop, newNode) {
			return
		}
	}
}

func (s *LockFreeStack) Pop() (int, bool) {
	for {
		currentTop := s.top.Load()
		if currentTop == nil {
			return 0, false // Stack is empty
		}
		
		next := currentTop.next
		// Attempt to update Top pointer to next node
		if s.top.CompareAndSwap(currentTop, next) {
			return currentTop.value, true // Success
		}
		// If CAS fails, another goroutine popped or pushed; retry loop
	}
}

func main() {
	stack := &LockFreeStack{}
	stack.Push(10)
	stack.Push(20)

	val, ok := stack.Pop()
	fmt.Printf("Popped: %d, Ok: %t\n", val, ok) // Popped: 20, Ok: true

	val2, ok2 := stack.Pop()
	fmt.Printf("Popped: %d, Ok: %t\n", val2, ok2) // Popped: 10, Ok: true
}
```

---

### Solution 4: Debugging the Un-synchronized Analytics Counter

#### 1. The Data Race:
In the HTTP server, multiple handler goroutines execute `ServeHTTP` concurrently. 
* The line `a.visitorCount++` compiles to three assembly instructions: (1) read `visitorCount` from memory to CPU register, (2) increment register, (3) write register back to memory.
* If two handlers execute this concurrently, they read the same initial value, causing a lost update. Furthermore, `a.visitorCount++` is a write, and `fmt.Fprintf` reads `a.visitorCount`, causing a read-write data race.

#### 2. The Corrected Code using `sync/atomic`:

```go
package main

import (
	"fmt"
	"net/http"
	"sync/atomic"
)

type Analytics struct {
	visitorCount atomic.Int64 // Use atomic struct type
}

func (a *Analytics) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	// Increment visitor count atomically and return the new value
	newVal := a.visitorCount.Add(1)
	fmt.Fprintf(w, "You are visitor #%d", newVal)
}

func main() {
	analytics := &Analytics{}
	http.ListenAndServe(":8080", analytics)
}
```

---

## 💼 Scenario-Based Questions

### Solution 5: The Flag Reload Race

#### 1 & 2. Race Detector output and consequences:
* **Race detector output:** It will flag a WRITE on `GlobalFlags = newFlags` in `reloadFlagsLoop` and a concurrent READ on `GlobalFlags.MaxUploadSizeMB` in `handleUpload`.
* **Consequences:** Because pointers are not written atomically on CPUs, a reader could capture a half-written pointer address (corrupted memory address), causing a segmentation fault crash. Additionally, compiler optimization could cache the old config pointer in CPU registers, preventing some goroutines from ever seeing the updated configuration.

#### 3. Redesigned Solution using `atomic.Pointer`:

```go
package main

import (
	"fmt"
	"sync/atomic"
	"time"
)

type Flags struct {
	BetaFeaturesEnabled bool
	MaxUploadSizeMB     int
}

type FeatureManager struct {
	flags atomic.Pointer[Flags]
}

func NewFeatureManager() *FeatureManager {
	fm := &FeatureManager{}
	initial := &Flags{BetaFeaturesEnabled: false, MaxUploadSizeMB: 10}
	fm.flags.Store(initial)
	return fm
}

func (fm *FeatureManager) GetFlags() *Flags {
	return fm.flags.Load() // Safe, concurrent-safe loading
}

func (fm *FeatureManager) ReloadLoop() {
	for {
		time.Sleep(10 * time.Second)
		newFlags := fetchFlagsFromAPI()
		fm.flags.Store(newFlags) // Safe swap
	}
}

func handleUpload(fm *FeatureManager, size int) {
	flags := fm.GetFlags()
	if size > flags.MaxUploadSizeMB {
		fmt.Println("Upload too large")
		return
	}
	fmt.Println("Upload successful")
}

func fetchFlagsFromAPI() *Flags {
	return &Flags{BetaFeaturesEnabled: true, MaxUploadSizeMB: 50}
}

func main() {
	fm := NewFeatureManager()
	go fm.ReloadLoop()

	for i := 0; i < 5; i++ {
		go func() {
			for {
				handleUpload(fm, 20)
				time.Sleep(100 * time.Millisecond)
			}
		}()
	}
	time.Sleep(100 * time.Millisecond)
}
```

---

### Solution 6: The Unaligned Struct Crash

#### 1. Why it Crashes on 32-bit CPUs:
On a 32-bit architecture, memory reads and writes occur in 4-byte (32-bit) intervals.
* For a 64-bit atomic instruction to execute, the variable **must be 8-byte aligned** (aligned to 64-bit boundaries).
* In the `Telemetry` struct, `DeviceID` takes 4 bytes. The subsequent field `TotalBytes` starts at offset 4 (not a multiple of 8).
* Because it is unaligned, performing atomic operations on it requires two memory access instructions. Since atomic hardware operations must be executed in a single instruction cycle, the CPU generates an unaligned memory access exception, causing the Go runtime to panic.
* On 64-bit architectures, variables are naturally aligned to 8-byte boundaries, so no crash occurs.

#### 2. Reorganizing Struct Fields (Manual Alignment):
Reorder fields so that the 8-byte (64-bit) field is placed first, ensuring it starts at offset 0 (which is always 8-byte aligned).

```go
type Telemetry struct {
	TotalBytes uint64 // Offset 0 (aligned)
	DeviceID   uint32 // Offset 8 (aligned)
	Status     uint32 // Offset 12 (aligned)
}
```

#### 3. Reorganizing using Go 1.19+ Typed Atomics:
Using `atomic.Uint64` ensures that the compiler automatically handles memory padding and alignment under the hood on both 32-bit and 64-bit architectures:

```go
import "sync/atomic"

type Telemetry struct {
	DeviceID   uint32
	TotalBytes atomic.Uint64 // Compiler guarantees alignment automatically!
	Status     uint32
}
```
* Note: The type `atomic.Uint64` automatically incorporates the compiler directives for memory alignment, protecting the code from architecture-specific lock panics.
