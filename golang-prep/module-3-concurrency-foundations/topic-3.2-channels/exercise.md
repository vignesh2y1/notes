# Exercises: Channels (Buffered vs Unbuffered)

Test your understanding of Go channel states, the `hchan` internal struct, data layout, safety guarantees, and communication patterns. Try to solve these questions without executing code first.

---

## 🧠 Conceptual Questions

### Question 1: Fill the State Grid
Based on the channel state rules, predict what happens (Blocks / Returns Zero Value + `ok=false` / Panics / Succeeds) for each action:

1. Writing a value to a `nil` channel.
2. Reading a value from a `nil` channel.
3. Closing a `nil` channel.
4. Writing a value to a closed channel.
5. Reading a value from a closed, empty channel.
6. Closing an already closed channel.

---

### Question 2: Internals of the `hchan` Mutex
Since `hchan` contains a `lock mutex` field, channels are not lock-free data structures.
1. When a goroutine writes to a buffered channel, what operations are performed under this mutex lock?
2. When a goroutine receives from an empty unbuffered channel, what role does the `recvq` wait queue and the `sudog` struct play? How does the scheduler suspend the calling goroutine?

---

## 🛠️ Practical Problems

### Question 3: Non-Blocking Channel Drain
Write a Go function `DrainChannel(ch <-chan int) []int` that extracts all currently buffered elements from a channel and returns them as a slice. 
* The function must **never block** (if there are no elements or the channel is empty, return immediately with the elements retrieved so far).
* Hint: Use select statements.

---

### Question 4: Fan-in Merger
Write a function `Merge(ch1, ch2 <-chan string) <-chan string` that takes two read-only channels of strings and returns a single read-only channel. 
* Any string sent to `ch1` or `ch2` should be forwarded to the returned channel.
* The returned channel must be closed automatically once **both** input channels are closed.
* Do not block the caller of `Merge`.

---

## 💼 Scenario-Based Questions

### Question 5: The Multi-Writer Panic
Consider the following program where three writers write data to a shared channel:

```go
package main

import (
	"fmt"
	"time"
)

func worker(id int, ch chan int) {
	for i := 0; i < 3; i++ {
		ch <- id * 10
	}
	close(ch) // Close channel when done writing
}

func main() {
	ch := make(chan int)
	go worker(1, ch)
	go worker(2, ch)
	go worker(3, ch)

	for val := range ch {
		fmt.Println("Received:", val)
	}
}
```

1. Run this code mentally. Why is this code guaranteed to crash?
2. What specific panic message will be generated?
3. How do you rewrite the program so that all workers can write their data and the channel is closed safely?

---

### Question 6: The Blocked API Server
You are reviewing a code snippet in a REST API server. The developer wants to log metrics asynchronously:

```go
package main

import "net/http"

var metricsChan = make(chan string) // Unbuffered global channel

func startMetricsProcessor() {
	// A worker reads metrics from the channel
	go func() {
		for metric := range metricsChan {
			sendToMetricsCollector(metric)
		}
	}()
}

func handleUserLogin(w http.ResponseWriter, r *http.Request) {
	// Push login metric asynchronously
	metricsChan <- "user_logged_in"

	w.Write([]byte("Login successful"))
}
```

1. Suppose the worker reading from `metricsChan` crashes or stops reading (e.g., if `sendToMetricsCollector` blocks indefinitely).
2. What will happen to subsequent requests calling `handleUserLogin`?
3. What happens to the system's throughput?
4. How would you redesign this interaction to make it resilient?
