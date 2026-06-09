# Topic 3.3: Select Statement

The `select` statement in Go is used to multiplex communications on multiple channels. It is similar to a `switch` statement, but instead of evaluating expressions, it blocks until one of its channel operations is ready to proceed.

This topic covers how `select` operates, its internal selection logic, how to perform non-blocking operations, timeouts, and critical system gotchas.

---

## 1. Concepts & Theory

### 1.1 Multiplexing Channel Operations
A `select` statement evaluates multiple send and receive channel operations.
* If one of the cases is ready to proceed, its channel operation executes, and the corresponding code block is run.
* If multiple cases are ready, Go **selects one at random**.
* If no case is ready:
  - If a `default` case is present, it executes immediately (non-blocking).
  - If no `default` case is present, the calling goroutine is suspended (blocked) until any of the channels in the cases become ready.

### 1.2 Under the Hood: Pseudo-Random Selection
Unlike an `if-else` or `switch` block, which executes cases top-to-bottom, the `select` statement uses a **pseudo-random permutation generator** at runtime:

```go
select {
case a := <-ch1:
    // ...
case b := <-ch2:
    // ...
}
```

#### Why Random?
If `select` evaluated cases top-to-bottom, and both `ch1` and `ch2` were continuously flooded with data, `ch1` would always be selected, completely starving `ch2` of CPU time.
* By randomizing case evaluation when multiple are ready, the Go runtime ensures fair scheduling and prevents starvation across concurrent data streams.

---

### 1.3 Non-Blocking Operations via `default`
If you include a `default` case, the `select` statement will never block. It acts as a polling mechanism:

```go
select {
case msg := <-ch:
    fmt.Println("Received:", msg)
default:
    fmt.Println("No message available, continuing...")
}
```

* **System Impact:** Non-blocking operations are crucial in low-latency path handling (like network loop servers or gaming engines) where you cannot afford to halt a thread waiting for slow logs or telemetry channels.

---

### 1.4 Timeouts with `time.After`
To prevent goroutines from hanging indefinitely on blocked operations, you can use `time.After(duration)`, which returns a channel that receives the current time after the specified duration.

```go
select {
case res := <-dataChan:
    process(res)
case <-time.After(1 * time.Second):
    fmt.Println("Timeout exceeded!")
}
```

---

## 2. Edge Cases & Limitations

### 2.1 The Empty Select (`select {}`)
An empty select statement has zero cases:
```go
select {}
```
* **Behavior:** It blocks the calling goroutine forever.
* **System Impact:** Since there are no channels to wake it up, it will never resume. If all other goroutines exit, this will trigger a runtime crash: `fatal error: all goroutines are asleep - deadlock!`. It is occasionally used in main functions of daemons, but graceful signals are preferred.

---

### 2.2 Selecting on `nil` Channels
If a channel in a `select` case is `nil`, that case is **completely ignored**. It does not block the select, nor does it succeed.

This is a powerful pattern for dynamically enabling or disabling select cases during execution:

```go
var ch chan int // nil channel

select {
case val := <-ch:
    // This case will NEVER run
default:
    // The select goes here because case 1 is ignored
}
```

---

### 2.3 Evaluation of Case Expressions
Any expression on the right-hand side of a select case is evaluated **when entering the select statement**, not when the case is chosen.

```go
select {
case ch <- getVal(): // getVal() runs IMMEDIATELY upon entering the select
    // ...
}
```
If `getVal()` blocks or takes time, it blocks the main thread *before* the select evaluation even begins.

---

## 3. Real-World Usage & Best Practices

### 3.1 The `time.After` Memory Leak in Loops
Calling `time.After` inside a long-running or high-frequency loop is a common cause of memory leaks in production.

#### The Problem:
`time.After` allocates a new `time.Timer` object and a channel every time it is called. If the other channel cases in the select are selected first, the timer channel is ignored. However, the timer object remains in the Go runtime heap until the timer duration expires.

If a loop runs 100,000 times per second with a 5-second timeout, you will accumulate 500,000 active timer objects in memory, causing CPU spikes and memory bloat.

#### The Fix:
Create and reuse a single `time.Timer` object, stopping and resetting it as needed.

---

## 4. Code Examples: Good vs. Bad

### ❌ Bad: Memory Leak via `time.After` in a Loop
This code creates a new timer channel on every tick of the loop, leaking memory if messages arrive rapidly.

```go
package main

import (
	"fmt"
	"time"
)

func processStream(ch chan string) {
	for {
		select {
		case msg, ok := <-ch:
			if !ok {
				return
			}
			fmt.Println("Msg:", msg)
		case <-time.After(30 * time.Second): // BAD: Leaks one timer per message loop!
			fmt.Println("Idle timeout, exiting.")
			return
		}
	}
}
```

### Good: Reusing `time.Timer` in a Loop
This approach manages a single timer instance, resetting it on each incoming message to prevent garbage collection spikes.

```go
package main

import (
	"fmt"
	"time"
)

func processStream(ch chan string) {
	// Create a single timer
	timer := time.NewTimer(30 * time.Second)
	defer timer.Stop()

	for {
		select {
		case msg, ok := <-ch:
			if !ok {
				return
			}
			fmt.Println("Msg:", msg)

			// Reset the timer for the next idle interval
			if !timer.Stop() {
				select {
				case <-timer.C:
				default:
				}
			}
			timer.Reset(30 * time.Second)

		case <-timer.C:
			fmt.Println("Idle timeout, exiting.")
			return
		}
	}
}
```

### Good: Dynamic Channel Disabling (Nil Channels)
This code processes data from two channels and stops reading from each channel as soon as it is closed, without busy-waiting.

```go
package main

import "fmt"

func consumeTwo(ch1, ch2 chan int) {
	for ch1 != nil || ch2 != nil {
		select {
		case val, ok := <-ch1:
			if !ok {
				ch1 = nil // Disable this case in future iterations
				fmt.Println("ch1 closed")
				continue
			}
			fmt.Println("Received from ch1:", val)

		case val, ok := <-ch2:
			if !ok {
				ch2 = nil // Disable this case in future iterations
				fmt.Println("ch2 closed")
				continue
			}
			fmt.Println("Received from ch2:", val)
		}
	}
	fmt.Println("Both channels drained.")
}

func main() {
	ch1 := make(chan int, 2)
	ch2 := make(chan int, 2)
	ch1 <- 1
	ch1 <- 2
	ch2 <- 10
	close(ch1)
	close(ch2)
	consumeTwo(ch1, ch2)
}
```
