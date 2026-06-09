# Exercises: Select Statement

Test your understanding of the `select` statement's scheduling logic, non-blocking channels, timeouts, dynamic nil-channel state handling, and memory leakage bugs.

---

## 🧠 Conceptual Questions

### Question 1: Selection Bias & Randomization
Consider the following program:

```go
package main

import "fmt"

func main() {
	ch1 := make(chan string, 1)
	ch2 := make(chan string, 1)

	ch1 <- "apple"
	ch2 <- "banana"

	select {
	case v1 := <-ch1:
		fmt.Println("Value:", v1)
	case v2 := <-ch2:
		fmt.Println("Value:", v2)
	}
}
```

1. If you run this program 100 times, what will the output distribution look like?
2. What is the technical mechanism inside the Go runtime that causes this distribution?
3. Why was this mechanism chosen by the Go runtime designers instead of evaluating cases top-to-bottom?

---

### Question 2: Execution Timing inside Case Statements
Explain the compilation/execution behavior of the following snippet. Does `heavyComputation()` run before or after the select evaluates if `dataChan` is ready? What are the performance implications?

```go
select {
case dataChan <- heavyComputation():
	fmt.Println("Data pushed")
case <-time.After(1 * time.Second):
	fmt.Println("Timed out")
}
```

---

## 🛠️ Practical Problems

### Question 3: Non-Blocking Logger
Write a Go struct `AsyncLogger` that:
1. Has a `Log(msg string)` method.
2. Internally sends logs to a buffered channel of size 10.
3. A background worker goroutine reads from the channel and prints the logs to console.
4. **Requirement:** If the buffered channel is full, the `Log` method must **not block** the caller; it should discard the log message and increment a counter of dropped messages.

---

### Question 4: The Heartbeat Timeout Pattern
Write a function `RunWorker(heartbeat <-chan struct{}, cancel <-chan struct{})` that:
1. Performs work indefinitely in a loop.
2. If a heartbeat is received on `heartbeat`, it prints "Heartbeat received" and resets its idle limit.
3. If no heartbeat is received for **3 seconds**, it prints "Worker timed out: No heartbeat" and returns.
4. If a signal is received on `cancel`, it prints "Worker stopped" and returns.
5. **Requirement:** Ensure your timer is safely cleaned up (stopped/reset) to prevent memory allocation leaks.

---

## 💼 Scenario-Based Questions

### Question 5: The Nil Channel Spin Lock
A developer wants to implement a toggleable worker using channels. They write this code:

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	var workChan chan int // uninitialized, nil
	stopChan := make(chan struct{})

	go func() {
		time.Sleep(2 * time.Second)
		close(stopChan)
	}()

	for {
		select {
		case val := <-workChan:
			fmt.Println("Processing:", val)
		case <-stopChan:
			fmt.Println("Stopped")
			return
		default:
			// Loop immediately, doing nothing
		}
	}
}
```

1. Explain the CPU usage characteristics of this program during the first 2 seconds.
2. Does the program block? If not, why?
3. What is the bug here, and how would you redesign it to prevent high CPU utilization?

---

### Question 6: Memory Leak Audit
Your production microservice runs out of memory every 4 hours. You run a heap profile and find that 90% of heap allocations are held by `time.Timer` structs. 
Here is the code processing stream events:

```go
func processEvents(eventStream <-chan Event) {
	for {
		select {
		case ev := <-eventStream:
			handleEvent(ev)
		case <-time.After(5 * time.Minute):
			log.Println("No events for 5 minutes, shutting down stream.")
			return
		}
	}
}
```
* The stream receives 5,000 events per second.
* Explain exactly why this function causes a rapid memory leak.
* Write the corrected code.
