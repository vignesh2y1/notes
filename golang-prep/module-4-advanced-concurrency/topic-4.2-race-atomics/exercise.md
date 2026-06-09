# Exercises: Race Conditions & Atomics

Test your understanding of data races, CPU bus locking, atomic primitives, Compare-And-Swap (CAS) algorithms, and the Go race detector.

---

## 🧠 Conceptual Questions

### Question 1: Data Race vs. Race Condition
1. Explain the technical difference between a **data race** and a **race condition**. (Can you have a program with a race condition that has *no* data races?)
2. Why does a Go map read/write collision result in a fatal runtime panic (`fatal error: concurrent map writes`), while a slice collision does not panic but yields corrupted data?

---

### Question 2: The Compare-And-Swap (CAS) Hardware Step
1. In pseudocode or simple steps, detail what a CPU's atomic CAS instruction does.
2. Why is a CAS operation considered "lock-free" from the application's perspective, even though the CPU may temporarily lock the hardware memory bus?
3. What is the ABA problem in lock-free programming, and how does it relate to CAS?

---

## 🛠️ Practical Problems

### Question 3: Lock-Free Stack Pop
Consider a lock-free singly-linked stack:

```go
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
```

Write the implementation of `Pop() (int, bool)`:
1. It must retrieve and remove the top element.
2. It must execute concurrently in a lock-free loop using `CompareAndSwap`.
3. It must return `(value, true)` if an item exists, or `(0, false)` if the stack is empty.

---

### Question 4: Debugging the Un-synchronized Analytics Counter
Find and fix the data race in the following HTTP endpoint metrics counter:

```go
package main

import (
	"fmt"
	"net/http"
)

type Analytics struct {
	visitorCount int64
}

func (a *Analytics) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	a.visitorCount++
	fmt.Fprintf(w, "You are visitor #%d", a.visitorCount)
}

func main() {
	analytics := &Analytics{}
	http.ListenAndServe(":8080", analytics)
}
```

1. Explain the data race in the implementation above.
2. Write the corrected code using `sync/atomic`.

---

## 💼 Scenario-Based Questions

### Question 5: The Flag Reload Race
A microservice loads feature flags from a remote API every 10 seconds. The developer writes the following reload loop:

```go
package main

import (
	"fmt"
	"time"
)

type Flags struct {
	BetaFeaturesEnabled bool
	MaxUploadSizeMB     int
}

var GlobalFlags = &Flags{BetaFeaturesEnabled: false, MaxUploadSizeMB: 10}

func reloadFlagsLoop() {
	for {
		time.Sleep(10 * time.Second)
		newFlags := fetchFlagsFromAPI()
		// Overwrite the global flags pointer
		GlobalFlags = newFlags
	}
}

func handleUpload(size int) {
	// Read feature flags
	if size > GlobalFlags.MaxUploadSizeMB {
		fmt.Println("Upload too large")
		return
	}
	fmt.Println("Upload successful")
}

func fetchFlagsFromAPI() *Flags {
	return &Flags{BetaFeaturesEnabled: true, MaxUploadSizeMB: 50}
}

func main() {
	go reloadFlagsLoop()
	// Start web server simulation
	for i := 0; i < 5; i++ {
		go func() {
			for {
				handleUpload(20)
				time.Sleep(100 * time.Millisecond)
			}
		}()
	}
	select {}
}
```

1. Run the Go race detector on this scenario. What issues will it report?
2. What are the potential runtime consequences of overwriting the `GlobalFlags` pointer concurrently?
3. Redesign the features flag storage and hot-swapping logic using `atomic.Pointer[Flags]` to make it 100% thread-safe.

---

### Question 6: The Unaligned Struct Crash
An old IoT edge application compiles a Go binary for a 32-bit ARM CPU. It defines a telemetry struct:

```go
type Telemetry struct {
	DeviceID   uint32
	TotalBytes uint64 // Atomic operations executed on this field
	Status     uint32
}
```

When executing `atomic.AddUint64(&telemetry.TotalBytes, 1)`, the program crashes instantly with a CPU panic.
1. Why does this crash occur on a 32-bit architecture but not on a 64-bit architecture?
2. How would you reorganize the struct fields to prevent the crash?
3. How do you resolve this using Go 1.19+ typed atomic variables?
