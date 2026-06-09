# Exercises: Advanced Concurrency Patterns

Test your understanding of Go's advanced concurrency compositions, concurrent pipelines, Fan-In/Fan-Out mechanics, rate limiters, worker pools, and memory leak vulnerabilities.

---

## 🧠 Conceptual Questions

### Question 1: Pipeline Stage Concurrency
A pipeline consists of three stages: Generator -> Stage A (Squaring) -> Stage B (Doubling) -> Client.
1. If Stage A and Stage B are both unbuffered, and the Client reads only 1 item, explain the status of the goroutines in Stage A and Generator.
2. Even if Stage A processes elements sequentially (one by one), why does placing it in a pipeline stage introduce performance gains on a multi-core machine?

---

### Question 2: The Fan-In Close Requirement
In the Fan-In implementation:
```go
func merge(channels ...<-chan int) <-chan int {
    // ...
}
```
1. Why can't we simply close the output channel inside the forwarder helper? (e.g. `defer close(out)` inside the helper function that reads from a single channel).
2. Why is a `sync.WaitGroup` used to synchronize the closure of the output channel? What occurs if we close the channel too early?

---

## 🛠️ Practical Problems

### Question 3: Even Number Multiplier Pipeline
Write a Go program implementing a concurrent pipeline with a `context.Context` cancellation parameter:
1. **Stage 1 (Generator):** Reads a slice of integers and sends them to the output channel.
2. **Stage 2 (FilterEven):** Discards any odd numbers, forwarding only even numbers.
3. **Stage 3 (Multiplier):** Multiplies the received even number by 10.
4. **Client:** Reads 3 items from the Multiplier stage, prints them, and then cancels the pipeline.
5. Verify that no goroutines leak.

---

### Question 4: Token Bucket Implementation
Implement a rate limiter struct `TokenBucket`:
1. Struct signature:
   ```go
   type TokenBucket struct {
       tokens chan struct{}
   }
   ```
2. Write a constructor `NewTokenBucket(capacity int, refillRate time.Duration)` that launches a background ticker to replenish the bucket.
3. Write `Allow() bool` which performs a non-blocking check to consume a token. If a token is available, return `true`. If empty, return `false`.
4. Ensure the background routine stops cleanly when the bucket is closed.

---

## 💼 Scenario-Based Questions

### Question 5: The Event Pipeline Staged Hang
You manage an event logging microservice. It uses a three-stage pipeline to parse, enrich, and write events to Elasticsearch.
During a network split where Elasticsearch becomes unreachable, the writing stage blocks.
Within minutes:
* The system runs out of memory (OOM).
* Thread allocations increase.
* Running `runtime.NumGoroutine()` shows tens of thousands of blocked goroutines.

1. Trace the propagation of the block upstream. Explain why the enrichment stage and parser stage block.
2. What happens to incoming events?
3. How would you design a pipeline stage cancellation and backpressure system using Context to prevent memory exhaustion?

---

### Question 6: Rate-Limited Batch Fetcher
You need to write a batch client that calls an external API 100 times.
* The external API enforces a strict rate limit of **5 requests per second**.
* If you exceed this, you receive a 423 Locked / 429 Too Many Requests response and your IP is banned.
* To optimize execution, you want to use **3 concurrent workers** executing the API calls.

Explain and write a Go program that:
1. Limits the total execution rate of the workers to 5 requests/sec using a ticker rate limiter.
2. Employs 3 workers that consume job tasks from a queue.
3. Coordinates the execution to guarantee the rate limit is never breached, even with multiple parallel workers.
