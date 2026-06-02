# Solutions: Pointers & Memory Allocation

Below are the step-by-step solutions, benchmark code, and optimization walkthroughs for the Topic 1.4 exercises.

---

## 🧠 Conceptual Questions

### Solution 1: Escape Analysis Fundamentals

1. **Definition & Timeline:**
   Escape analysis is a static compiler check performed **at compile-time** (not runtime). The compiler inspects the AST (Abstract Syntax Tree) to trace the lifetime and flow of all variables.
2. **Advantages over Manual Allocation:**
   In languages like C/C++, developers must manually manage memory layout using `new`/`malloc` (heap) and local stack allocations. This is highly error-prone:
   * Returning a pointer to a stack-allocated variable results in a **dangling pointer** (reading corrupted memory or crashing).
   * Forgetting to free heap memory causes a **memory leak**.
   * Go's escape analysis completely eliminates dangling pointers by promoting variables to the heap if their pointer outlives the function. It reduces developer cognitive overhead while preserving safety.
3. **Performance Cost Comparison:**
   * **Stack Allocation ($O(1)$):** Extremely cheap. It translates to a single CPU instruction (subtraction of the stack pointer register). When the function returns, the register is added back, reclaiming memory instantly.
   * **Heap Allocation ($O(\text{Search})$):** Highly expensive. Go must search its memory allocator structures (mspan/mcache) for a free block, lock resources if needed, track boundaries, and eventually run the garbage collector to sweep the memory, consuming CPU cycles and increasing response latency.

---

### Solution 2: The Interface Escape Trap

When you call `fmt.Println(x)`, the argument `x` escapes to the heap because:

1. **Boxing into Interfaces:**
   `fmt.Println` is defined as `func Println(a ...any)`. The type `any` is an alias for `interface{}`.
2. **Interface Representation:**
   To represent a primitive type like `int` as an interface, Go wraps it in a struct containing a pointer to the type descriptor and a pointer to the value. 
3. **Escaping the Pointer:**
   Because interfaces are dynamic and handled at runtime, the compiler cannot prove where the value pointed to by the interface will go once passed into `fmt.Println`'s internals. To be safe, the compiler must copy the value of `x` to the heap and pass that heap address to the interface wrapper.

---

## 🛠️ Practical Problems

### Solution 3: Predict the Escapes

* **`a` (Stack):**
  `createTokenValue()` returns a `Token` struct value. The compiler copies the value from `createTokenValue`'s stack frame directly to `main`'s stack frame. The variable never escapes.
* **`b` (Heap):**
  `createTokenPointer()` returns a pointer `*Token` pointing to local variable `t`. Because this pointer is returned and assigned in `main()`, it outlives `createTokenPointer`'s stack frame. Thus, `t` escapes to the heap.
* **`c` (Stack):**
  The string literal "Hello" is defined. Its address is stored in pointer `d`.
* **`d` (Stack):**
  `d` is a pointer pointing to `c`. Since both `d` and `c` are only used within `main()` (they do not leave `main()`'s frame), they stay on the stack.
* **`e` (Stack):**
  `e` is a slice allocated via `make([]int, 100)`. Because its size is small (100 integers = 800 bytes) and constant, and it does not escape `main()`, the compiler allocates its backing array on the stack. (If it were `make([]int, 1000000)`, it would escape to the heap due to size constraints).

---

### Solution 4: Designing Benchmark for Copying Tradeoffs

#### 1. Benchmark Code (`main_test.go`):
```go
package main

import "testing"

type BigStruct struct {
	Data [1000]int // 8000 bytes (8KB)
}

//go:noinline
func passValue(b BigStruct) int {
	return b.Data[0]
}

//go:noinline
func passPointer(b *BigStruct) int {
	return b.Data[0]
}

func BenchmarkPassValue(b *testing.B) {
	var big BigStruct
	for i := 0; i < b.N; i++ {
		_ = passValue(big)
	}
}

func BenchmarkPassPointer(b *testing.B) {
	var big BigStruct
	for i := 0; i < b.N; i++ {
		_ = passPointer(&big)
	}
}
```
*(Note: `//go:noinline` tells the compiler not to inline the functions, forcing an actual call/copy comparison).*

#### 2. Slower Pointer Conditions:
Passing by pointer can become slower than passing by value when the pointer forces a variable to be allocated on the heap inside a loop. 
If we allocate a struct, pass it by value (stack copy), it has **zero allocation cost**. If we pass by pointer, and that pointer escapes to the heap, we incur a heap allocation cost on every loop iteration. The cost of a heap allocation (e.g. 50-100ns) is vastly greater than the cost of copying an 8KB stack memory block (e.g. 2-5ns).

---

## 💼 Scenario-Based Questions

### Solution 5: The Coordinate Parser GC Spike

#### 1. The Leak analysis:
```go
c := parseSingle(val) // returns Coordinate (value)
coords[i] = &c        // Takes address of local variable c
```
Taking the address `&c` of a local block variable and storing it in a slice `coords` which is returned from the function forces `c` to escape to the heap.
* Go must allocate `Coordinate` on the heap for every element in the slice.
* If the input slice `raw` has a length of $N$, this function performs **$N$ heap allocations for the coordinates** plus **1 heap allocation for the `coords` slice itself**, leading to $N+1$ allocations. This causes massive garbage collection spikes under high load.

#### 2. Optimized Implementation:
To solve this, return a slice of raw structures (`[]Coordinate`) instead of pointers (`[]*Coordinate`). This groups all coordinates in a single contiguous memory block:

```go
type Coordinate struct {
	Lat, Lng float64
}

func parseCoordinates(raw []string) []Coordinate {
	// Allocate a single contiguous slice of values. 
	// This performs EXACTLY 1 heap allocation for the entire batch.
	coords := make([]Coordinate, len(raw))
	
	for i, val := range raw {
		// Directly write the coordinate into the slice memory.
		// Since we write by value, it is written directly into the pre-allocated backing array.
		coords[i] = parseSingle(val) 
	}
	
	return coords
}

func parseSingle(val string) Coordinate {
	return Coordinate{Lat: 12.34, Lng: 56.78}
}
```
#### Performance Gains:
* **Before:** $N+1$ allocations. If $N = 1000$, we had 1001 heap allocations.
* **After:** Exactly **1 allocation**.
* This reduces Garbage Collection overhead by over 99%, resolving the CPU usage spikes.
