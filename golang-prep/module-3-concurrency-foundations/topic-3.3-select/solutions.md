# Solutions: Select Statement

Here are the step-by-step solutions, explanations, and corrected Go code implementations for the Topic 3.3 exercises.

---

## 🧠 Conceptual Questions

### Solution 1: Selection Bias & Randomization

#### 1. Output Distribution:
* If you run this program 100 times, the output will be roughly split: **~50% "apple" and ~50% "banana"**.

#### 2. Technical Mechanism:
* When entering the `select` block, the runtime calls `selectgo()` (defined in `runtime/select.go`).
* `selectgo` takes all the channel operations and uses a pseudo-random permutation generator to randomize their evaluation order.
* It iterates through this randomized list. If a channel operation is ready to proceed, it immediately locks that channel, completes the operation, and executes the case block.

#### 3. Why Randomization Was Chosen:
* If evaluation was sequential (top-to-bottom), developers would write programs that favor the first case. In systems with highly active channels, the top case would block or continuously consume execution cycles, starvation-blocking lower cases.
* Randomized selection ensures fairness and prevents starvation across concurrent processing ports, making Go's select highly stable in concurrent network environments.

---

### Solution 2: Execution Timing inside Case Statements

#### 1. Evaluation Order:
* In Go, expressions on the right-hand side of case operations (e.g., function arguments or function calls) are evaluated **immediately** upon entering the `select` block, **before** the runtime determines which channel is ready.
* Therefore, `heavyComputation()` runs **first, synchronously** on the calling thread.

#### 2. Performance Implications:
* If `heavyComputation()` takes 500ms to complete, the entire `select` statement blocks for 500ms *before* evaluating if `dataChan` or the `time.After` timeout channel is ready.
* This defeats the purpose of the 1-second timeout case. If `heavyComputation()` blocks, the timeout cannot interrupt it.
* **To fix this:** Execute the computation in a separate goroutine and pass the result via an intermediate channel, or run the computation only after confirming the channel is ready (if possible).

---

## 🛠️ Practical Problems

### Solution 3: Non-Blocking Logger

Using a `default` case inside the `Log` method ensures that if writing to the channel would block, it instantly branches to the default case and increments the drop counter.

```go
package main

import (
	"fmt"
	"sync/atomic"
	"time"
)

type AsyncLogger struct {
	logChan     chan string
	droppedLogs int64 // Use atomic to prevent race conditions on counter
}

func NewAsyncLogger() *AsyncLogger {
	logger := &AsyncLogger{
		logChan: make(chan string, 10),
	}
	go logger.startWorker()
	return logger
}

func (l *AsyncLogger) Log(msg string) {
	select {
	case l.logChan <- msg:
		// Message successfully queued
	default:
		// Channel buffer is full. Drop log and increment counter.
		atomic.AddInt64(&l.droppedLogs, 1)
	}
}

func (l *AsyncLogger) startWorker() {
	for msg := range l.logChan {
		fmt.Println("[LOG]:", msg)
		time.Sleep(10 * time.Millisecond) // Simulate slow log writing
	}
}

func (l *AsyncLogger) GetDroppedCount() int64 {
	return atomic.LoadInt64(&l.droppedLogs)
}

func main() {
	logger := NewAsyncLogger()

	// Fill buffer and force drops
	for i := 1; i <= 20; i++ {
		logger.Log(fmt.Sprintf("Event number %d", i))
	}

	time.Sleep(200 * time.Millisecond)
	fmt.Printf("Total dropped logs: %d\n", logger.GetDroppedCount())
}
```

---

### Solution 4: The Heartbeat Timeout Pattern

To prevent timer leaks in a long-running loop, we create a single `time.Timer` instance and manage its lifecycle via `.Stop()`, draining its channel if necessary, and calling `.Reset()`.

```go
package main

import (
	"fmt"
	"time"
)

func RunWorker(heartbeat <-chan struct{}, cancel <-chan struct{}) {
	// Initialize a single timer
	timer := time.NewTimer(3 * time.Second)
	defer timer.Stop() // Ensure resources are freed when function returns

	for {
		select {
		case <-heartbeat:
			fmt.Println("Heartbeat received")

			// Safely stop and reset the timer
			if !timer.Stop() {
				// Drain the timer channel if it already expired
				select {
				case <-timer.C:
				default:
				}
			}
			timer.Reset(3 * time.Second)

		case <-timer.C:
			fmt.Println("Worker timed out: No heartbeat")
			return

		case <-cancel:
			fmt.Println("Worker stopped")
			return
		}
	}
}

func main() {
	hb := make(chan struct{})
	cancel := make(chan struct{})

	go RunWorker(hb, cancel)

	// Send heartbeats
	time.Sleep(1 * time.Second)
	hb <- struct{}{}
	time.Sleep(1 * time.Second)
	hb <- struct{}{}
	time.Sleep(4 * time.Second) // Force a timeout
}
```

---

## 💼 Scenario-Based Questions

### Solution 5: The Nil Channel Spin Lock

#### 1. CPU Usage Characteristics:
* During the first 2 seconds, the program will consume **100% of a CPU core** (100% utilization of one OS thread).

#### 2. Does the program block?
* No, it does not block. 
* Reading from the nil channel `workChan` does block that specific case. However, because a `default` case is present inside the loop, the `select` statement immediately evaluates `default`, completes, and loops again.
* This causes an infinite loop running at maximum processor speed (busy-waiting / spinning), which wastes CPU cycles.

#### 3. The Fix:
Remove the `default` case. Without `default`, the loop will block on `stopChan` and the nil channel `workChan`. Since `workChan` is nil, it is ignored. The goroutine blocks cleanly on `stopChan` without spinning.

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	var workChan chan int // nil channel
	stopChan := make(chan struct{})

	go func() {
		time.Sleep(2 * time.Second)
		close(stopChan)
	}()

	for {
		select {
		case val := <-workChan: // This is ignored because workChan is nil
			fmt.Println("Processing:", val)
		case <-stopChan: // Blocks cleanly here until stopChan is closed
			fmt.Println("Stopped")
			return
		}
	}
}
```

---

### Solution 6: Memory Leak Audit

#### Why the leak occurs:
1. `time.After(5 * time.Minute)` creates a new `time.Timer` struct and a channel *every single iteration* of the loop.
2. In this loop, if an event arrives on `eventStream` (which occurs 5,000 times/sec), the first case evaluates.
3. The newly created timer from the second case is ignored, but **it remains scheduled in the Go runtime memory** until its 5-minute timeout expires.
4. With 5,000 events/sec, the loop creates 5,000 timers/sec. Over 5 minutes, the heap will accumulate:
   $$5,000\text{ timers/sec} \times 300\text{ seconds} = 1,500,000 \text{ active timer allocations}$$
5. These millions of timer allocations clog the memory allocator and trigger garbage collection thrashing, leading to memory exhaustion.

#### Corrected Code:
Create one persistent timer outside the loop and reuse it.

```go
package main

import (
	"log"
	"time"
)

type Event struct{}

func handleEvent(ev Event) {}

func processEvents(eventStream <-chan Event) {
	// Create a single timer outside the loop
	timer := time.NewTimer(5 * time.Minute)
	defer timer.Stop()

	for {
		select {
		case ev, ok := <-eventStream:
			if !ok {
				return
			}
			handleEvent(ev)

			// Reset timer to prevent it firing
			if !timer.Stop() {
				select {
				case <-timer.C:
				default:
				}
			}
			timer.Reset(5 * time.Minute)

		case <-timer.C:
			log.Println("No events for 5 minutes, shutting down stream.")
			return
		}
	}
}
```
