# Solutions: Slices & Arrays under the Hood

Below are the step-by-step explanations, code solutions, and profiling analysis for the Topic 1.2 exercises.

---

## 🧠 Conceptual Questions

### Solution 1: Shared Backing Array Side-Effects

#### The Output:
```
s1: [3 4]
s2: [3 4]
s (len): 0
```

#### Step-by-step Explanation:
1. `s := make([]int, 0, 5)` allocates a backing array of size 5. The slice header `s` has `pointer -> array[0]`, `len = 0`, and `cap = 5`.
2. `s1 := append(s, 1, 2)`:
   * The `append` function receives a copy of the slice header `s` (length 0).
   * Since there is enough capacity (`len + 2 <= cap` i.e. `0 + 2 <= 5`), `append` writes `1` and `2` into the backing array at indices `0` and `1`.
   * It returns a new slice header `s1` with `pointer -> array[0]`, `len = 2`, `cap = 5`.
3. `s2 := append(s, 3, 4)`:
   * The `append` function receives a copy of the slice header **`s`** (whose length is still **0**).
   * It sees `len = 0` and enough capacity in the backing array.
   * It writes `3` and `4` starting at index `0` of the backing array. **This overwrites the `1` and `2`** previously written by the first append because both operations use the same backing array pointer at index 0.
   * It returns a new slice header `s2` with `pointer -> array[0]`, `len = 2`, `cap = 5`.
4. `s1` and `s2` both share the backing array. The backing array now contains `[3, 4, 0, 0, 0]`.
5. When printed, `s1` read values at indices 0 and 1, producing `[3, 4]`. `s2` does the same, producing `[3, 4]`.
6. `s` itself was passed by value to `append()`, so its own header length was never modified. Thus, `len(s)` remains `0`.

---

### Solution 2: Nil vs. Empty Slices

#### 1. Header Representation in Memory:
* **Nil Slice (`var sliceA []int`):**
  The slice header has `array = nil` (0x0), `len = 0`, `cap = 0`. It does not allocate any memory.
* **Empty Slice (`sliceB := []int{}` or `sliceC := make([]int, 0)`):**
  The slice header has `array = zerobase` (a non-zero address pointing to a special static memory location in the Go runtime used for all zero-size allocations), `len = 0`, `cap = 0`.

#### 2. Behavior with `len()` and `cap()`:
Both `nil` and empty slices behave identically when passed to `len()` and `cap()`. They both return `0`.

#### 3. Behavior with `json.Marshal`:
This is a critical distinction in REST APIs:
* `json.Marshal(sliceA)` (nil slice) yields the JSON value **`null`**.
* `json.Marshal(sliceB)` (empty slice) yields the JSON value **`[]`** (an empty array).

**Production Tip:** Always initialize slice fields in API response structs to empty slices (using `make` or literal initialization) instead of leaving them `nil` if your API contract expects an array structure rather than `null`.

---

## 🛠️ Practical Problems

### Solution 3: Removing an Element from a Slice

#### 1. Mutation of the Backing Array:
Yes, the expression `append(s[:i], s[i+1:]...)` mutates the backing array.
`s[:i]` represents a slice header pointing to the same backing array with length `i`. When we append `s[i+1:]...` to it, it shifts all elements after index `i` one position to the left in the backing array.

#### 2. Impact on Sharing Slices:
If `other` shared the backing array, its elements at and after index `i` will shift. Because `other`'s length has not changed, its last element will now appear duplicated.
For example, if the backing array was `[1, 2, 3, 4]` and we remove index `1` (`2`), the backing array becomes `[1, 3, 4, 4]`.

#### 3. SafeRemove Implementation:
To avoid mutating the original slice, allocate a new backing array and copy the elements:
```go
func SafeRemove(s []int, i int) []int {
	if i < 0 || i >= len(s) {
		// Return a copy of the original slice if index is out of bounds
		res := make([]int, len(s))
		copy(res, s)
		return res
	}
	
	// Pre-allocate the exact size needed
	result := make([]int, len(s)-1)
	
	// Copy elements before index i
	copy(result[:i], s[:i])
	
	// Copy elements after index i
	copy(result[i:], s[i+1:])
	
	return result
}
```

---

### Solution 4: Slice Growth Allocation Profiling

#### 1. Allocation Analysis:
If `n = 1000`, starting with a zero slice causes Go to reallocate the backing array and copy existing elements repeatedly as capacity limit thresholds are crossed.
Specifically, capacity grows as: `0 -> 1 -> 2 -> 4 -> 8 -> 16 -> 32 -> 64 -> 128 -> 256 -> 512 -> 840 -> 1260 ...`
This causes approximately **11 to 12 allocations** and copies.

#### 2. Optimized Function:
To make only **exactly one allocation**, pre-allocate the capacity of the backing array:
```go
func generateSequence(n int) []int {
	// Length is 0, Capacity is pre-allocated to n
	result := make([]int, 0, n) 
	for i := 0; i < n; i++ {
		result = append(result, i)
	}
	return result
}
```

---

## 💼 Scenario-Based Questions

### Solution 5: The TCP Packet Log OOM

#### 1. Root Cause:
The statement `typeID := packet[0:4]` creates a new slice header. Its `pointer` points to the start of the `packet` slice. Crucially, its `cap` represents the capacity of the original 4KB backing array.
Because `typeID` is stored in the global map `payloadCache`, the Go Garbage Collector (GC) sees a live reference to a slice header pointing to that backing array. Even though we only care about 4 bytes, the GC **cannot collect the entire 4KB backing array**.
If we process 1,000,000 packets:
* Expected memory: 1,000,000 * 4 bytes ≈ 4 MB.
* Actual memory held: 1,000,000 * 4 KB ≈ 4 GB.
This results in a major memory leak and an eventual crash (Out of Memory).

#### 2. Corrected Function:
To break the reference to the large backing array, allocate a new, small slice and copy the 4 bytes into it. This allows the large 4KB packet backing array to be garbage collected:

```go
var payloadCache = make(map[string][]byte)

func processPacket(packet []byte, packetName string) {
	// Allocate a new backing array of exactly 4 bytes
	typeID := make([]byte, 4)
	
	// Copy the contents
	copy(typeID, packet[0:4])
	
	// Store the independent slice in the map
	payloadCache[packetName] = typeID
}
```
This keeps our memory footprint to exactly the size of the payload IDs (4 bytes per packet).
