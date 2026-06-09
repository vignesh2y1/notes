# Topic 4.4: Advanced Concurrency Patterns

Go's primitives (channels and goroutines) enable the construction of advanced architectural patterns for processing high-throughput data streams. 

This topic covers the implementation details of Generators, Pipelines, Fan-out/Fan-in, Worker Pools, and Rate Limiting, along with leak mitigation strategies in complex stream stages.

---

## 1. Concepts & Theory

### 1.1 The Generator Pattern
A **generator** is a function that returns a read-only channel. Under the hood, it spawns a goroutine to produce data and pushes it to the channel, encapsulating the concurrency logic:

```go
func GenerateRange(start, end int) <-chan int {
    ch := make(chan int)
    go func() {
        defer close(ch)
        for i := start; i < end; i++ {
            ch <- i
        }
    }()
	return ch
}
```

---

### 1.2 The Pipeline Pattern
A **pipeline** is a series of stages connected by channels, where each stage is a group of goroutines running the same function:

```
[Generator] ---> (ch1) ---> [Stage A: Square] ---> (ch2) ---> [Stage B: Double] ---> Client
```

* Each stage receives data from upstream, transforms it, and sends it downstream.
* **Stream Concurrency:** Pipeline stages execute in parallel, optimizing execution times on multicore systems.

---

### 1.3 Fan-Out, Fan-In
When a particular pipeline stage is CPU-intensive, you can distribute the load:

```
                   +---> [Worker 1] ---+
                   |                   |
[Generator] ---> (ch1) -> [Worker 2] ---> (ch2) ---> [Merger / Fan-in] ---> Client
                   |                   |
                   +---> [Worker 3] ---+
```

1. **Fan-Out:** Multiple goroutines read from the **same** channel to process work in parallel (parallel workers).
2. **Fan-In:** A single goroutine reads from **multiple** channels and merges their output into a single channel.

---

### 1.4 Worker Pools
A worker pool maintains a fixed number of goroutines that consume tasks from a shared queue channel, process them, and write outputs. This bounds resource consumption.

---

### 1.5 Rate Limiting
API consumers must respect server rate limits. Go handles this using channels:
* **Ticker Limiting:** Emitting a ticket on a `time.Ticker` channel at regular intervals.
* **Token Bucket:** A buffered channel loaded with "tokens" at a fixed rate. Requests consume a token. This permits handling temporary spikes (bursty traffic) up to the bucket capacity while maintaining an overall rate limit.

---

## 2. Edge Cases & Limitations

### 2.1 The Pipeline Goroutine Leak
A major trap in pipelines occurs when a downstream stage terminates early (e.g., due to an error or hit limit):

```go
// If client only reads 2 values and returns,
// the upstream stage blocks on ch <- value forever, leaking its goroutine.
ch := Square(Generate(10))
fmt.Println(<-ch)
fmt.Println(<-ch)
return
```

* **Mitigation:** Always pass a `context.Context` to all pipeline stages. When the consumer exits, the context is canceled, prompting all upstream stages to abort their writes and terminate cleanly.

---

## 3. Real-World Usage & Best Practices

### 3.1 Design Principles
1. **Channel Ownership:** Every stage function should create and return its own output channel, and close it when done (`defer close(out)`).
2. **Cancellation Propagation:** Every stage must check `ctx.Done()` in its select loops.
3. **Buffer Sizing:** Avoid arbitrary buffer sizes. Use unbuffered channels for strict synchronization, and small buffers (1-100) to absorb minor system jitter.

---

## 4. Code Examples: Good vs. Bad

### ❌ Bad: Leaky Pipeline on Early Termination
If the client exits early, the `Square` and `Generate` goroutines block on channel write lines and leak.

```go
package main

import "fmt"

func Generate(nums ...int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for _, n := range nums {
			out <- n
		}
	}()
	return out
}

func Square(in <-chan int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for n := range in {
			out <- n * n // Blocks here forever if client stops reading!
		}
	}()
	return out
}

func main() {
	in := Generate(1, 2, 3, 4, 5)
	out := Square(in)

	fmt.Println(<-out) // Prints 1
	fmt.Println(<-out) // Prints 4
	// main exits, leaving 2 active goroutines blocked in memory
}
```

### Good: Cancellable Pipeline via Context
By passing a context down the pipeline, we ensure that all stages abort writes and exit cleanly if the client terminates early.

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func Generate(ctx context.Context, nums ...int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for _, n := range nums {
			select {
			case out <- n:
			case <-ctx.Done():
				return // Abort early
			}
		}
	}()
	return out
}

func Square(ctx context.Context, in <-chan int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for n := range in {
			select {
			case out <- n * n:
			case <-ctx.Done():
				return // Abort early
			}
		}
	}()
	return out
}

func main() {
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel() // Cancels context, terminating all stages

	in := Generate(ctx, 1, 2, 3, 4, 5)
	out := Square(ctx, in)

	fmt.Println(<-out) // Prints 1
	fmt.Println(<-out) // Prints 4

	cancel() // Trigger cancellation of upstream stages
	time.Sleep(10 * time.Millisecond) // Give scheduler time to clean up
}
```

### Good: Fan-Out / Fan-In Processing
This example showcases a multi-worker pipeline where results from parallel workers are safely merged back into a single output stream.

```go
package main

import (
	"context"
	"fmt"
	"sync"
	"time"
)

// Worker stage (Fan-Out target)
func worker(ctx context.Context, id int, in <-chan int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for val := range in {
			select {
			case out <- val * 2:
				fmt.Printf("[Worker %d] Processed %d\n", id, val)
				time.Sleep(100 * time.Millisecond) // Simulate CPU heavy work
			case <-ctx.Done():
				return
			}
		}
	}()
	return out
}

// Merger stage (Fan-In orchestrator)
func merge(ctx context.Context, channels ...<-chan int) <-chan int {
	var wg sync.WaitGroup
	out := make(chan int)

	// Forwarder helper
	output := func(c <-chan int) {
		defer wg.Done()
		for n := range c {
			select {
			case out <- n:
			case <-ctx.Done():
				return
			}
		}
	}

	wg.Add(len(channels))
	for _, c := range channels {
		go output(c)
	}

	// Coordinator closes out once all forwarders are done
	go func() {
		wg.Wait()
		close(out)
	}()

	return out
}

func main() {
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	// 1. Generate tasks
	jobs := make(chan int, 10)
	for i := 1; i <= 6; i++ {
		jobs <- i
	}
	close(jobs)

	// 2. Fan-out to 3 workers
	w1 := worker(ctx, 1, jobs)
	w2 := worker(ctx, 2, jobs)
	w3 := worker(ctx, 3, jobs)

	// 3. Fan-in (Merge) results
	results := merge(ctx, w1, w2, w3)

	for res := range results {
		fmt.Println("Result:", res)
	}
}
```
