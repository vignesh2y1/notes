# Exercises: Pointers & Memory Allocation

Test your understanding of Go pointers, stack vs. heap allocation, escape analysis rules, and performance tradeoffs.

---

## 🧠 Conceptual Questions

### Question 1: Escape Analysis Fundamentals
1. What is Escape Analysis? Does it run at compile-time or runtime?
2. Why is having a smart compiler perform escape analysis more beneficial than forcing developers to manually allocate memory using `new` (heap) and standard variable declarations (stack)?
3. What is the execution performance cost of a heap allocation compared to a stack allocation in Go?

---

### Question 2: The Interface Escape Trap
Consider the following program:
```go
package main

import "fmt"

func main() {
	x := 42
	fmt.Println(x)
}
```
If you run `go build -gcflags="-m" main.go`, you will see that `x` escapes to the heap.
Explain exactly why a simple local integer variable `x`, which does not outlive `main()`, escapes to the heap when passed to `fmt.Println()`.

---

## 🛠️ Practical Problems

### Question 3: Predict the Escapes
Analyze the following code. For each variable declaration (`a`, `b`, `c`, `d`, `e`), predict whether it allocates on the **stack** or **escapes to the heap**. Explain your reasoning.

```go
package main

type Token struct {
	Value string
}

func main() {
	a := createTokenValue()
	b := createTokenPointer()
	
	c := "Hello"
	d := &c
	
	e := make([]int, 100)
	
	_ = a
	_ = b
	_ = d
	_ = e
}

func createTokenValue() Token {
	t := Token{Value: "active"}
	return t
}

func createTokenPointer() *Token {
	t := Token{Value: "expired"}
	return &t
}
```

---

### Question 4: Designing Benchmark for Copying Tradeoffs
Suppose you are analyzing the performance of passing structs by value vs. pointer:
```go
type BigStruct struct {
	Data [1000]int // 8000 bytes (8KB)
}

type SmallStruct struct {
	A, B int // 16 bytes
}
```
1. Write the code for two benchmark functions in `main_test.go` to test passing `BigStruct` by value vs. by pointer.
2. Under what condition does passing `BigStruct` by pointer become slower than passing by value (hint: think about where they are allocated)?

---

## 💼 Scenario-Based Questions

### Question 5: The Coordinate Parser GC Spike
You are building an IoT data processing pipeline that handles millions of GPS coordinate records per second. You run a pprof CPU and memory profile and notice that `parseCoordinates` is causing a high frequency of GC cycles, consuming 35% of CPU time.

Here is the code:
```go
type Coordinate struct {
	Lat, Lng float64
}

func parseCoordinates(raw []string) []*Coordinate {
	coords := make([]*Coordinate, len(raw))
	for i, val := range raw {
		c := parseSingle(val) // returns Coordinate struct (value)
		coords[i] = &c
	}
	return coords
}

func parseSingle(val string) Coordinate {
	// Parses raw string to Lat and Lng
	return Coordinate{Lat: 12.34, Lng: 56.78}
}
```

1. Explain the specific line in `parseCoordinates` that causes the coordinate data to escape to the heap. How many heap allocations occur for a slice of size `N`?
2. Rewrite the `parseCoordinates` function so that it processes the coordinates with **exactly one** heap allocation for the entire batch, dramatically reducing GC pressure.
