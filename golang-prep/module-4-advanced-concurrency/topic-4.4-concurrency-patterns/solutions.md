# Solutions: Advanced Concurrency Patterns

Here are the step-by-step solutions, architectural analyses, and Go code implementations for the Topic 4.4 exercises.

---

## 🧠 Conceptual Questions

### Solution 1: Pipeline Stage Concurrency

#### 1. Status of Goroutines on Single Read:
* The Client requests 1 item.
* Stage B (Doubling) receives the item, processes it, and yields it to the Client. Stage B then blocks waiting for a second item from Stage A.
* Stage A (Squaring) processes the second item and attempts to send it to Stage B. Because the channel is unbuffered and Stage B is blocked (since the Client is no longer reading), the Stage A goroutine blocks on the send operation: `out <- val`.
* The Generator processes the third item and attempts to send it to Stage A. Because Stage A is blocked, the Generator goroutine blocks on its send operation as well.
* **Summary:** Stage B is blocked waiting for input; Stage A and Generator are blocked attempting to write output.

#### 2. Performance Gains:
* Placing operations in stages introduces performance gains because **different elements are processed at different stages concurrently**.
* While Stage B is doubling element $N$, Stage A is squaring element $N+1$, and Generator is fetching element $N+2$.
* This allows the workload to be distributed across multiple CPU cores, resulting in horizontal pipeline parallelism (assembly line speedup).

---

### Solution 2: The Fan-In Close Requirement

#### 1. Why we cannot close in the forwarder:
* The forwarder helper reads from a single input channel and writes to the shared output channel.
* Since multiple forwarders (one for each input channel) write to the **same** output channel, closing the output channel inside any individual forwarder would cause a panic.
* If Forwarder 1 finishes first and closes the output channel, Forwarder 2 will panic with `panic: send on closed channel` when it attempts to write its next item.

#### 2. Why sync.WaitGroup is required:
* A `sync.WaitGroup` tracks the completion of all forwarder goroutines.
* The coordinator goroutine waits until **all** forwarders have finished reading their respective input channels (meaning no more values will ever be written to the output channel) before invoking `close(out)`.
* Closing the channel too early results in a runtime panic for any remaining active writer threads.

---

## 🛠️ Practical Problems

### Solution 3: Even Number Multiplier Pipeline

This code implements the three stages, incorporating context checks in all select blocks to prevent leaks when the client exits early.

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func Generate(ctx context.Context, nums []int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for _, n := range nums {
			select {
			case out <- n:
			case <-ctx.Done():
				return
			}
		}
	}()
	return out
}

func FilterEven(ctx context.Context, in <-chan int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for n := range in {
			if n%2 == 0 {
				select {
				case out <- n:
				case <-ctx.Done():
					return
				}
			}
		}
	}()
	return out
}

func Multiplier(ctx context.Context, in <-chan int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for n := range in {
			select {
			case out <- n * 10:
			case <-ctx.Done():
				return
			}
		}
	}()
	return out
}

func main() {
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	nums := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}

	gen := Generate(ctx, nums)
	even := FilterEven(ctx, gen)
	results := Multiplier(ctx, even)

	// Read 3 items and stop
	for i := 0; i < 3; i++ {
		val, ok := <-results
		if !ok {
			break
		}
		fmt.Println("Result:", val) // Prints: 20, 40, 60
	}

	// Cancel context to terminate upstream stages immediately
	cancel()
	time.Sleep(10 * time.Millisecond) // Give workers time to clean up
}
```

---

### Solution 4: Token Bucket Implementation

We use a buffered channel to hold tokens and a ticker to regenerate them.

```go
package main

import (
	"fmt"
	"time"
)

type TokenBucket struct {
	tokens  chan struct{}
	stop    chan struct{}
}

func NewTokenBucket(capacity int, refillRate time.Duration) *TokenBucket {
	tb := &TokenBucket{
		tokens: make(chan struct{}, capacity),
		stop:   make(chan struct{}),
	}

	// Fill the bucket initially
	for i := 0; i < capacity; i++ {
		tb.tokens <- struct{}{}
	}

	// Start background replenisher
	go func() {
		ticker := time.NewTicker(refillRate)
		defer ticker.Stop()
		for {
			select {
			case <-ticker.C:
				select {
				case tb.tokens <- struct{}{}:
					// Replenished token
				default:
					// Bucket full, discard token
				}
			case <-tb.stop:
				return
			}
		}
	}()

	return tb
}

func (tb *TokenBucket) Allow() bool {
	select {
	case <-tb.tokens:
		return true // Token consumed
	default:
		return false // Bucket empty, rate limit exceeded
	}
}

func (tb *TokenBucket) Close() {
	close(tb.stop)
}

func main() {
	limiter := NewTokenBucket(3, 100*time.Millisecond)
	defer limiter.Close()

	// Consume tokens rapidly
	for i := 1; i <= 5; i++ {
		fmt.Printf("Request %d: Allowed? %t\n", i, limiter.Allow())
	}

	time.Sleep(250 * time.Millisecond)
	fmt.Printf("After sleep - Request 6: Allowed? %t\n", limiter.Allow())
}
```

---

## 💼 Scenario-Based Questions

### Solution 5: The Event Pipeline Staged Hang

#### 1. Why Upstream Stages Block:
1. When Elasticsearch becomes unreachable, the Writer stage blocks attempting to write to the database network buffer.
2. The channel connecting the Enrichment stage to the Writer stage fills up (since the Writer is no longer reading).
3. The Enrichment stage blocks on the line `writerChan <- enrichedEvent`.
4. Consequently, the channel connecting the Parser stage to the Enrichment stage fills up.
5. The Parser stage blocks on `enrichmentChan <- parsedEvent`.
6. Finally, the main thread or intake server attempting to queue new events to the Parser channel blocks.

#### 2. Impact on Incoming Events:
Since the entire pipeline is blocked, the API ingestion endpoint hangs. It stops responding to client logging calls, causing clients to block, network sockets to fill, and the microservice to experience OOM errors as queued inputs pile up in memory.

#### 3. Redesign with Context & Backpressure:
To make the pipeline resilient, we:
* Pass a `context.Context` from the HTTP server to the pipeline stages. If a client timeout occurs, the context is canceled, aborting blocked reads/writes.
* Implement a **non-blocking write check** with a `default` case at the API intake level. If the intake buffer is full, return an immediate error response (503 Service Unavailable / 429 Too Many Requests) to the client (backpressure), preventing heap memory overflow.

---

### Solution 6: Rate-Limited Batch Fetcher

To coordinate multiple parallel workers to respect a single rate limit, we route request tokens through a shared channel controlled by a `time.Ticker`.

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func worker(id int, jobs <-chan int, rateLimiter <-chan time.Time, wg *sync.WaitGroup) {
	defer wg.Done()
	for job := range jobs {
		// Wait for rate limiter ticket before executing
		<-rateLimiter

		// Simulate API execution
		fmt.Printf("[Worker %d] Executed API request for job %d at %s\n", 
			id, job, time.Now().Format("15:04:05.000"))
	}
}

func main() {
	const totalJobs = 15
	const numWorkers = 3

	jobs := make(chan int, totalJobs)
	var wg sync.WaitGroup

	// Set up ticker to emit a ticket 5 times per second (every 200ms)
	limiter := time.NewTicker(200 * time.Millisecond)
	defer limiter.Stop()

	// Start workers
	for i := 1; i <= numWorkers; i++ {
		wg.Add(1)
		go worker(i, jobs, limiter.C, &wg)
	}

	// Enqueue jobs
	for i := 1; i <= totalJobs; i++ {
		jobs <- i
	}
	close(jobs)

	wg.Wait()
	fmt.Println("All batch calls completed safely.")
}
```
* Note: This architecture guarantees that even though 3 workers run in parallel, they all read from the same `limiter.C` channel, meaning at most 5 requests are made per second in total.
