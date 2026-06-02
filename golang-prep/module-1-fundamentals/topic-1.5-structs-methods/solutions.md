# Solutions: Structs, Methods & Receivers

Below are the step-by-step answers and explanations for the Topic 1.5 exercises.

---

## 🧠 Conceptual Questions

### Solution 1: Struct Alignment Math

#### 1. Naive `OrderInfo` Calculation (24 Bytes):
* `Active` (bool) = 1 byte.
* `OrderID` (int64) = 8 bytes. Since it is a 64-bit int, it must be aligned to an 8-byte boundary. The compiler inserts **7 padding bytes** after `Active` to align `OrderID` at byte offset 8.
* `OrderID` takes 8 bytes (occupying offsets 8 to 15).
* `Cancelled` (bool) = 1 byte (occupying offset 16).
* **End padding:** The total struct size must be a multiple of the largest field alignment guarantee (8 bytes for `int64`). The compiler inserts **7 padding bytes** at the end of the struct to bring the total size up to 24 bytes.

```
Offsets:  0      1 - 7       8 - 15      16     17 - 23
Layout:  |Active| Padding  |  OrderID  |Cancel|  Padding  | (24 Bytes total)
```

#### 2. Optimized `OrderInfo` (16 Bytes):
By reordering the fields from largest to smallest (or grouping smaller fields together):
```go
type OrderInfo struct {
	OrderID   int64 // 8 bytes (offset 0-7)
	Active    bool  // 1 byte  (offset 8)
	Cancelled bool  // 1 byte  (offset 9)
}
```
* `OrderID` takes 8 bytes.
* `Active` (1 byte) and `Cancelled` (1 byte) occupy offsets 8 and 9.
* **End padding:** The compiler inserts **6 padding bytes** at the end to align the total size to the next multiple of 8 (16 bytes).

```
Offsets:  0 - 7       8      9      10 - 15
Layout:  |  OrderID  |Active|Cancel|  Padding  | (16 Bytes total)
```
We successfully saved 8 bytes (33% size reduction).

---

### Solution 2: Embedded Method promotion and Shadowing

1. **Prediction & Reasoning:**
   `m.DoWork()` prints `"Manager doing managing work"`.
   This is because the `Manager` struct defines its own method `DoWork()`. Even though the embedded `Worker`'s `DoWork()` method is promoted to `Manager`, the outer struct's method **shadows** (overrides) the promoted one.
2. **Calling the Embedded Method:**
   To call the shadowed method, reference the promoted field explicitly:
   ```go
   m.Worker.DoWork()
   ```

---

## 🛠️ Practical Problems

### Solution 3: The Copied Lock Vulnerability

#### Why the Code is Dangerous:
1. **Pass-by-Value Mutex Copy:**
   The function `updateCounter(c SafeCounter)` takes `SafeCounter` by value. When invoked, Go copies the entire struct `c`, including its `sync.Mutex` field `mu`.
2. **Broken Synchronization:**
   Because each goroutine receives a copy of the mutex, they are each locking and unlocking a **different mutex instance**. The lock does not coordinate access to a shared resource.
3. **Data Loss:**
   Because the struct `c` itself is copied, the incremented `c.value++` only modifies the local copy. Once `updateCounter` returns, the updated value is lost.

#### The Fix:
Pass the struct by pointer to ensure all goroutines share the same mutex and mutate the same state:

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

type SafeCounter struct {
	mu    sync.Mutex
	value int
}

// Pass by pointer (*SafeCounter)
func updateCounter(c *SafeCounter) {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.value++
}

func main() {
	counter := SafeCounter{value: 0}
	for i := 0; i < 10; i++ {
		// Pass the pointer (&counter)
		go updateCounter(&counter) 
	}
	time.Sleep(100 * time.Millisecond)
	fmt.Println("Final Counter Value:", counter.value) // Output: 10
}
```

---

### Solution 4: Predict the Nil Receiver Behavior

#### Prediction:
The program compiles and executes successfully, outputting:
```
User name: Anonymous
```
It does **not** panic.

#### Mechanics:
In Go, a method call `node.GetName()` is syntactic sugar for a function call where the receiver is passed as the first argument:
```go
(*UserNode).GetName(node)
```
Even though `node` is `nil`, Go calls the function normally. The receiver pointer `u` inside `GetName` receives the value `nil`.
The function check `if u == nil` evaluates to `true` and returns `"Anonymous"`. The application only panics if the method attempts to access a field of a nil pointer (e.g., `u.Username`), which triggers a segment fault at runtime.

---

## 💼 Scenario-Based Questions

### Solution 5: Order Book Memory Compression

#### 1. Naive `Order` Layout & Calculation (40 Bytes):
* `IsBuyOrder` (bool): 1 byte (offset 0)
* **Padding:** 7 bytes (to align `OrderID` to offset 8)
* `OrderID` (uint64): 8 bytes (offset 8-15)
* `IsMarket` (bool): 1 byte (offset 16)
* **Padding:** 7 bytes (to align `Price` to offset 24)
* `Price` (float64): 8 bytes (offset 24-31)
* `Quantity` (uint32): 4 bytes (offset 32-35)
* **Padding:** 4 bytes at the end (to pad total struct to multiple of 8)
* **Total size:** 1 + 7 (pad) + 8 + 1 + 7 (pad) + 8 + 4 + 4 (pad) = **40 bytes**.

#### 2. Optimized Struct Code:
Order fields from largest to smallest size:
```go
type Order struct {
	OrderID    uint64  // 8 bytes (offset 0-7)
	Price      float64 // 8 bytes (offset 8-15)
	Quantity   uint32  // 4 bytes (offset 16-19)
	IsBuyOrder bool    // 1 byte  (offset 20)
	IsMarket   bool    // 1 byte  (offset 21)
	// 2 bytes padding at the end to align to multiple of 8 (offset 22-23)
}
```

#### 3. Savings Calculation:
* **Optimized Size:** 8 + 8 + 4 + 1 + 1 + 2 (pad) = **24 bytes**.
* **Bytes Saved Per Order:** 40 bytes - 24 bytes = **16 bytes** (40% space reduction).
* **Total RAM Saved for 10M Orders:**
  $$16\text{ bytes} \times 10,000,000 = 160,000,000\text{ bytes} \approx 160\text{ MB}$$
This is a massive optimization for a latency-sensitive service, keeping more data inside CPU L1/L2 caches and improving cache locality.
