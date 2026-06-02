# Exercises: Structs, Methods & Receivers

Test your understanding of struct memory layout, byte alignment, value vs. pointer receivers, method sets, and composition via embedding.

---

## 🧠 Conceptual Questions

### Question 1: Struct Alignment Math
Consider the following struct:
```go
type OrderInfo struct {
	Active    bool  // 1 byte
	OrderID   int64 // 8 bytes
	Cancelled bool  // 1 byte
}
```
1. Calculate the exact byte size of `OrderInfo` on a 64-bit architecture, taking padding into account. Sketch the memory layout block by block.
2. Reorder the fields to minimize memory size. What is the new size, and what is the layout?

---

### Question 2: Embedded Method promotion and Shadowing
Consider the following code:
```go
package main

import "fmt"

type Worker struct{}

func (w Worker) DoWork() {
	fmt.Println("Worker doing basic work")
}

type Manager struct {
	Worker
}

func (m Manager) DoWork() {
	fmt.Println("Manager doing managing work")
}

func main() {
	m := Manager{}
	m.DoWork()
	// How do you call the embedded Worker's DoWork() method on m?
}
```
1. Predict the output of `m.DoWork()`. Why does this output appear?
2. Write the line of code that invokes the embedded `Worker`'s `DoWork` method.

---

## 🛠️ Practical Problems

### Question 3: The Copied Lock Vulnerability
Explain why the following code is dangerous and violates thread safety, even though it uses a mutex. Show how to fix it.
```go
package main

import (
	"sync"
	"time"
)

type SafeCounter struct {
	mu    sync.Mutex
	value int
}

func updateCounter(c SafeCounter) {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.value++
}

func main() {
	counter := SafeCounter{value: 0}
	for i := 0; i < 10; i++ {
		go updateCounter(counter)
	}
	time.Sleep(100 * time.Millisecond)
}
```

---

### Question 4: Predict the Nil Receiver Behavior
Predict the output of the following program. Does it compile? Does it panic? Explain the mechanics of why it behaves the way it does.
```go
package main

import "fmt"

type UserNode struct {
	Username string
}

func (u *UserNode) GetName() string {
	if u == nil {
		return "Anonymous"
	}
	return u.Username
}

func main() {
	var node *UserNode // node is nil
	fmt.Println("User name:", node.GetName())
}
```

---

## 💼 Scenario-Based Questions

### Question 5: Order Book Memory Compression
You are building an in-memory high-frequency crypto trading engine. The engine keeps millions of order structures in memory.
Every order contains:
* `OrderID` (uint64) - 8 bytes
* `IsBuyOrder` (bool) - 1 byte
* `Price` (float64) - 8 bytes
* `Quantity` (uint32) - 4 bytes
* `IsMarket` (bool) - 1 byte

Here is the naive definition:
```go
type Order struct {
	IsBuyOrder bool
	OrderID    uint64
	IsMarket   bool
	Price      float64
	Quantity   uint32
}
```

1. Sketch the memory layout and compute the byte size of the naive `Order` struct.
2. Re-arrange the fields of the struct to achieve optimal memory size. Show your optimized struct code.
3. Compute the exact byte size of the optimized struct. How many bytes did you save per order? If the engine holds 10,000,000 orders in memory, how much RAM is saved in total?
