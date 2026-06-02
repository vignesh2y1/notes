# Exercises: Slices & Arrays under the Hood

Test your understanding of array memory allocation, slice headers, capacity growth, shared backing arrays, and memory optimization.

---

## 🧠 Conceptual Questions

### Question 1: Shared Backing Array Side-Effects
Predict the output of the following code. Explain step-by-step how the slice headers of `s`, `s1`, and `s2` behave, and why the final values print the way they do.
```go
package main

import "fmt"

func main() {
	s := make([]int, 0, 5)
	
	s1 := append(s, 1, 2)
	s2 := append(s, 3, 4)
	
	fmt.Println("s1:", s1)
	fmt.Println("s2:", s2)
	fmt.Println("s (len):", len(s))
}
```

---

### Question 2: Nil vs. Empty Slices
Explain the internal difference between a **nil slice** and an **empty slice**:
```go
var sliceA []int          // nil slice
sliceB := []int{}         // empty slice
sliceC := make([]int, 0)  // empty slice
```
1. How do their slice headers compare in memory?
2. How do they behave when passed to `len()` and `cap()`?
3. How do they behave when serialized using `json.Marshal`?

---

## 🛠️ Practical Problems

### Question 3: Removing an Element from a Slice
You need to remove an element at index `i` from slice `s`.
A common way to do this in Go is:
```go
s = append(s[:i], s[i+1:]...)
```
1. Does this operation mutate the backing array of the original slice `s`?
2. If another slice `other` was sharing the backing array before this operation, what will happen to `other`?
3. Write a function `SafeRemove(s []int, i int) []int` that removes the element at index `i` *without* mutating the backing array of the input slice.

---

### Question 4: Slice Growth Allocation Profiling
Analyze the following function:
```go
func generateSequence(n int) []int {
	var result []int
	for i := 0; i < n; i++ {
		result = append(result, i)
	}
	return result
}
```
1. If `n = 1000`, approximately how many memory allocations (backing array resizing/copying) will occur?
2. Rewrite this function to achieve maximum efficiency (exactly **one** memory allocation for the backing array).

---

## 💼 Scenario-Based Questions

### Question 5: The TCP Packet Log OOM
You are writing a high-throughput network packet analyzer. The system captures packets in chunks of 4KB (`[]byte`). For each packet, you need to extract the 4-byte payload type ID located at the beginning (`packet[0:4]`) and store it in a global map (`map[string][]byte`) for metrics.

Here is the implementation:
```go
var payloadCache = make(map[string][]byte)

func processPacket(packet []byte, packetName string) {
	// Extract payload type ID (first 4 bytes)
	typeID := packet[0:4]
	payloadCache[packetName] = typeID
}
```
Although each stored slice is only 4 bytes, after processing millions of packets, the server runs out of memory (OOM) and crashes.
1. Explain the root cause of this OOM.
2. Provide the corrected `processPacket` function to resolve this leak.
