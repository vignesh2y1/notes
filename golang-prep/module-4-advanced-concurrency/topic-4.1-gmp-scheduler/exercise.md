# Exercises: The GMP Scheduler Model

Test your understanding of the low-level mechanics of Go's GMP scheduler, local vs. global run queues, work-stealing algorithms, netpoller handling, synchronous system calls, and containerized deployment CPU issues.

---

## 🧠 Conceptual Questions

### Question 1: GMP Status Mapping
For each scenario below, describe the state of the entities (**G**, **M**, and **P**):

1. A goroutine is waiting to read from an open channel (no sender is ready).
2. A goroutine is currently executing a CPU-intensive math formula.
3. A goroutine is calling a blocking CGO library function that executes a synchronous C-sleep.
4. A goroutine is calling `net.Dial` to connect to a TCP socket (waiting for network handshake).

---

### Question 2: Work-Stealing Sequence
P1's Local Run Queue (LRQ) has just become empty. 
1. Write down the exact step-by-step sequence the worker thread M1 (paired with P1) executes to find another goroutine to run.
2. Why is the work-stealing check randomized instead of checking Ps in sequential order (P2, P3, etc.)?

---

## 🛠️ Practical Problems

### Question 3: Observing OS Thread Counts
Write a Go program that calls a blocking system call in a loop (e.g., `syscall.Write` or standard synchronous file opening/reading operations) across 1,000 concurrent goroutines.
1. Use the `runtime` package to write a function that prints the current number of active goroutines using `runtime.NumGoroutine()`.
2. How can we check the number of physical OS threads created by the Go process on a macOS terminal?
3. Why does the OS thread count exceed `GOMAXPROCS` in this scenario?

---

### Question 4: Manual GOMAXPROCS Allocation
Write a simple benchmark program in Go that runs a heavy computation (e.g., calculating prime numbers).
1. Configure the benchmark to run with `GOMAXPROCS` set to `1`, `2`, `4`, and `8` programmatically using the `runtime.GOMAXPROCS()` function.
2. Explain the CPU usage characteristics and benchmark performance trends you would expect on a 4-core machine.

---

## 💼 Scenario-Based Questions

### Question 5: The Kubernetes CPU Throttling Mystery
A high-throughput API gateway built in Go is deployed to a Kubernetes cluster. 
* The pod resource limits are set to: `resources.limits.cpu: "2"`.
* The Kubernetes node running the pod has a 64-core Intel CPU.
* The application team reports that the API response latency spikes to over 2 seconds under medium load, and Kubernetes metrics show severe **CPU throttling** (CFS quota throttles).
* Running the application locally on a 4-core machine does not show this latency.

Explain:
1. What value does `GOMAXPROCS` default to inside the pod, and why?
2. How does this default value trigger CPU throttling by the Kubernetes CFS (Completely Fair Scheduler)?
3. How do you resolve this production issue?

---

### Question 6: Anatomy of a Syscall Handoff
A goroutine (G1) executing on thread M1 and logical processor P1 invokes a synchronous file system call: `syscall.Read(fd, buf)`.

Trace the exact behavior of:
1. G1: Where does it go, and what is its status?
2. M1: What does it do? Does it remain active in user-space?
3. P1: What happens to P1 and the other goroutines (G2, G3) queued in P1's Local Run Queue?
4. When the read operation returns from the OS kernel, how do G1 and M1 resume execution?
