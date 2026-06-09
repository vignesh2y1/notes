# Solutions: Channels (Buffered vs Unbuffered)

Here are the step-by-step solutions, explanations, and corrected Go implementations for the Topic 3.2 exercises.

---

## 🧠 Conceptual Questions

### Solution 1: Fill the State Grid

Here is the behavior for each requested action:

1. **Writing to a `nil` channel:** Blocks forever (the goroutine is placed in wait state and never woken up).
2. **Reading from a `nil` channel:** Blocks forever.
3. **Closing a `nil` channel:** **Panics** with `panic: close of nil channel`.
4. **Writing to a closed channel:** **Panics** with `panic: send on closed channel`.
5. **Reading from a closed, empty channel:** Returns the type's zero value immediately, with the second return parameter (`ok`) set to `false`.
6. **Closing an already closed channel:** **Panics** with `panic: close of closed channel`.

---

### Solution 2: Internals of the `hchan` Mutex

#### 1. Under-the-Hood Actions for Writing to a Buffered Channel:
When a goroutine executes `ch <- val` on a buffered channel:
1. The writing goroutine acquires the low-level `lock` of the `hchan` struct.
2. It checks if the circular buffer `buf` has space (`qcount < dataqsiz`).
3. If space exists, it copies `val` into the slot pointed to by `sendx` in the buffer array.
4. It increments `sendx` (wrapping around using modulo arithmetic if it reaches the end of the buffer) and increments `qcount`.
5. It releases the `lock` and returns.

#### 2. Suspending a Goroutine on an Empty Unbuffered Channel:
When a goroutine reads from an empty unbuffered channel (`<-ch`):
1. The reading goroutine acquires the `lock`.
2. It detects no matching sender and no buffer.
3. It allocates or retrieves a `sudog` struct representing itself, storing the target pointer where it wants the returned data to go.
4. It enqueues this `sudog` into the channel's `recvq` wait list.
5. It releases the channel `lock` and calls the runtime scheduler function `gopark()`.
6. The scheduler changes the status of this goroutine from `_Grunning` to `_Gwaiting` and schedules a different goroutine on the OS thread (P), suspending execution of the reading goroutine without blocking the underlying OS thread.

---

## 🛠️ Practical Problems

### Solution 3: Non-Blocking Channel Drain

To achieve a non-blocking channel drain, we use a `select` statement with a `default` case. The `default` case is selected immediately if no other case is ready (i.e., if reading would block).

```go
package main

import "fmt"

func DrainChannel(ch <-chan int) []int {
	var elements []int

	for {
		select {
		case val, ok := <-ch:
			if !ok {
				// Channel is closed and completely empty
				return elements
			}
			elements = append(elements, val)
		default:
			// If reading would block (channel is open but empty), return immediately
			return elements
		}
	}
}

func main() {
	ch := make(chan int, 5)
	ch <- 10
	ch <- 20
	ch <- 30

	fmt.Println("Drained:", DrainChannel(ch)) // Drained: [10 20 30]
	fmt.Println("Drained again:", DrainChannel(ch)) // Drained again: [] (non-blocking)
}
```

---

### Solution 4: Fan-in Merger

To merge two channels asynchronously without blocking the caller, we spawn an internal orchestrator goroutine. We also use a `sync.WaitGroup` to track when both channels are closed before closing the output channel.

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func Merge(ch1, ch2 <-chan string) <-chan string {
	out := make(chan string)
	var wg sync.WaitGroup

	// Helper function to forward values from one channel
	forward := func(ch <-chan string) {
		defer wg.Done()
		for val := range ch {
			out <- val
		}
	}

	wg.Add(2)
	go forward(ch1)
	go forward(ch2)

	// Monitor goroutine to close the output channel once both workers finish
	go func() {
		wg.Wait()
		close(out)
	}()

	return out
}

func main() {
	c1 := make(chan string)
	c2 := make(chan string)

	merged := Merge(c1, c2)

	go func() {
		c1 <- "Hello"
		c2 <- "World"
		close(c1)
		close(c2)
	}()

	for val := range merged {
		fmt.Println(val)
	}
}
```

---

## 💼 Scenario-Based Questions

### Solution 5: The Multi-Writer Panic

#### 1 & 2. Why the code crashes and Panic message:
* The code starts three goroutines representing producers writing to `ch`.
* The first goroutine that finishes writing its 3 elements calls `close(ch)`.
* Once the channel is closed, the remaining active goroutines that are still executing their loop will attempt to write (`ch <- id * 10`) to the closed channel.
* **Result:** The application immediately crashes with the panic message: `panic: send on closed channel`.

#### 3. Corrected Implementation using `sync.WaitGroup`:
We must coordinate the writers. We only close the channel when **all** writers have completed their work.

```go
package main

import (
	"fmt"
	"sync"
)

func worker(id int, ch chan int, wg *sync.WaitGroup) {
	defer wg.Done()
	for i := 0; i < 3; i++ {
		ch <- id * 10
	}
}

func main() {
	ch := make(chan int)
	var wg sync.WaitGroup

	// Track 3 workers
	for i := 1; i <= 3; i++ {
		wg.Add(1)
		go worker(i, ch, &wg)
	}

	// Coordinator goroutine to close channel when all workers are done
	go func() {
		wg.Wait()
		close(ch)
	}()

	// Read until closed
	for val := range ch {
		fmt.Println("Received:", val)
	}
}
```

---

### Solution 6: The Blocked API Server

#### 1 & 2. Impact on subsequent requests:
* If the consumer goroutine stops reading from `metricsChan`, the unbuffered channel becomes blocked.
* In `handleUserLogin`, the line `metricsChan <- "user_logged_in"` requires a concurrent reader.
* Because the channel is unbuffered, subsequent API requests calling `handleUserLogin` will block indefinitely on the send line, waiting for a reader.

#### 3. Impact on system throughput:
* The HTTP handlers handling the requests will never return a response to the user.
* All incoming client request threads/goroutines will block and hang.
* The system throughput drops to **zero** for the login endpoint, and server memory will inflate due to the accumulation of blocked request handler goroutines, eventually causing an OOM crash.

#### 4. Resilient Redesign:
To make metrics emission resilient and non-blocking:
1. **Use a Buffered Channel:** Add a buffer so that spikes in metric emissions don't block the request handler thread.
2. **Use a Non-blocking Select:** Drop the metric if the channel is full, ensuring that application execution is preferred over telemetry data collection under heavy stress.

```go
package main

import (
	"fmt"
	"net/http"
)

// Introduce a buffer to absorb spikes
var metricsChan = make(chan string, 1000)

func handleUserLogin(w http.ResponseWriter, r *http.Request) {
	// Non-blocking write: drop metric if the buffer is full
	select {
	case metricsChan <- "user_logged_in":
		// Metric queued successfully
	default:
		// Drop metric or log a warning locally (prevents API hang)
		fmt.Println("Warning: metrics queue full, dropping log")
	}

	w.Write([]byte("Login successful"))
}
```
