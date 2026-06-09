# Exercises: Goroutines & The Scheduler (Intro)

Test your understanding of Go's concurrency model, memory representation of goroutines versus OS threads, context switching overheads, and preemption. Do not look at the solutions file or execute code until you have attempted to answer all questions.

---

## 🧠 Conceptual Questions

### Question 1: Memory Scaling Math
A server has 16GB of RAM. 
1. If you run a Java application that maps each thread to a standard OS thread with a default stack size of 1MB, what is the theoretical maximum number of threads you can create before running out of memory (assuming 2GB is reserved for OS and runtime)?
2. If you run a Go application where each goroutine starts with a 2KB stack size, what is the theoretical maximum number of goroutines you can create under the same memory constraints?
3. What mechanism prevents the Go application from running out of stack space if a goroutine suddenly requires more than 2KB of memory?

---

### Question 2: Thread vs. Goroutine Context Switching
Explain:
1. Why does an OS thread context switch require flushing or updating the Translation Lookaside Buffer (TLB)?
2. Why does a Goroutine context switch avoiding TLB flushes result in higher CPU instruction execution speed?
3. Which system layer manages the registers when switching a Goroutine, and which layer manages them when switching an OS Thread?

---

## 🛠️ Practical Problems

### Question 3: Dynamic Stack Growth Demonstration
Write a Go program containing a recursive function that calls itself a specific depth (e.g., 1000 times) and uses a stack variable to store states. 
* Detail what happens to the goroutine's memory location during execution.
* How can you print the address of a local variable inside the recursive calls to verify whether the stack has grown and moved to a new memory block?

---

### Question 4: Writing a Concurrency Limiter (Semaphore)
Write a Go program that simulates processing 50 high-latency tasks (each taking 100ms).
1. The program must run these tasks concurrently using goroutines.
2. It must restrict execution to at most **5 concurrent tasks** at any given moment.
3. Use a buffered channel as a semaphore to enforce this concurrency boundary.
4. Ensure the program waits for all 50 tasks to finish before exiting `main`.

---

## 💼 Scenario-Based Questions

### Question 5: The "Silent Leak" HTTP Client
A developer writes the following handler to fetch user profiles from an external microservice:

```go
package main

import (
	"context"
	"net/http"
	"time"
)

func fetchUserProfile() string {
	ch := make(chan string) // Unbuffered channel

	go func() {
		// Simulate network call
		time.Sleep(2 * time.Second)
		ch <- "User Profile Data"
	}()

	select {
	case res := <-ch:
		return res
	case <-time.After(500 * time.Millisecond):
		return "Default Guest"
	}
}
```

1. Explain the bug in the code above when the simulated network call takes longer than 500ms.
2. What happens to the spawned goroutine? Does it terminate or linger in memory?
3. How does this affect the application memory footprint under high traffic?
4. Write the corrected version of the `fetchUserProfile` function.

---

### Question 6: The Un-preemptible Spin-Lock
Suppose you compile a Go program using **Go 1.13** (which does not support asynchronous preemption). One of your goroutines executes the following code:

```go
func spin() {
	for {
		// Spin-lock waiting for a global boolean variable
		if ready {
			break
		}
	}
}
```

If you run this code on a machine with `GOMAXPROCS=1`, explain:
1. What happens to other goroutines in the system? Do they get execution time?
2. How would the behavior change if you compiled the exact same code with **Go 1.14+**? Explain the runtime mechanics involved.
