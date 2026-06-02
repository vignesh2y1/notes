# Topic 1.2: Slices & Arrays under the Hood

Understanding the distinction between arrays and slices, and the internal representation of slices, is critical for writing high-performance, bug-free Go code. Slice allocation and backing array sharing are among the most common sources of bugs and performance bottlenecks in Go.

---

## 1. Concepts & Theory

### 1.1 Arrays: Fixed-Size Value Types
In Go, an **array** is a numbered sequence of elements of a single type with a **fixed length** defined at compile time.

```go
var arr [5]int // An array of 5 integers, initialized to zero
```

#### Key Characteristics of Arrays:
1. **Size is part of the Type:** `[5]int` and `[10]int` are completely distinct, incompatible types. You cannot assign one to the other.
2. **Value Semantics:** When you assign an array to another variable or pass it to a function, **Go copies the entire array** and its elements. This can be extremely expensive for large arrays.
3. **Contiguous Memory:** Arrays are allocated as a single, contiguous block of memory.

---

### 1.2 Slices: Dynamic Views of Arrays
A **slice** is a lightweight data structure that provides a dynamic, flexible window into a backing array. Unlike arrays, the size of a slice is not fixed.

#### The Slice Header (Internal Structure)
Under the hood, a slice is represented as a struct defined in `runtime/slice.go`:

```go
type slice struct {
    array unsafe.Pointer // Pointer to the first element of the backing array
    len   int            // Current length of the slice
    cap   int            // Capacity of the backing array from the start pointer
}
```

* **Pointer (`array`):** Points to the start of the slice within the backing array. It does not have to point to the first element of the array; it can point to any index.
* **Length (`len`):** The number of elements currently in the slice. Accessing indices beyond `len-1` triggers a runtime panic, even if they are within `cap`.
* **Capacity (`cap`):** The maximum number of elements the slice can hold without allocating a new backing array. It is the number of elements in the backing array starting from the slice's pointer.

```
       +---------------------------------------------+
Array: |  0  |  1  |  2  |  3  |  4  |  5  |  6  |  7  |
       +---------------------------------------------+
                     ^
                     | (pointer)
               +-------------+
Slice:         | len: 3      |
               | cap: 6      |
               +-------------+
```

---

### 1.3 Creating Slices
1. **Slice Literal:** `s := []int{1, 2, 3}` (Go automatically creates a backing array under the hood).
2. **From an Array/Slice:** `s := arr[1:4]` (Shares the backing array of `arr`).
3. **Using `make`:** `s := make([]int, length, capacity)`.

---

### 1.4 The Mechanics of `append()` & Capacity Growth
The built-in `append()` function adds elements to the end of a slice.

#### What happens during `append()`:
1. **If `len < cap`:**
   * The new element is written directly to the next index in the backing array.
   * `len` is incremented.
   * The updated slice header is returned. Any other slice sharing this backing array will see the change if they read that memory.
2. **If `len == cap`:**
   * Go allocates a **new, larger backing array** in memory.
   * Go copies the existing elements from the old array to the new one.
   * The new element is appended to the new array.
   * The slice's pointer is updated to point to the new array, and `cap` is increased.

#### How Slice Capacity Grows (Go 1.18+):
To optimize allocation performance, Go increases capacity in jumps rather than one element at a time:
* If the requested capacity is more than double the current capacity, Go allocates the requested capacity.
* If the current capacity is **less than 256**, Go doubles it (`newcap = cap * 2`).
* If the current capacity is **256 or more**, Go uses a formula that transitions slowly towards `1.25x` growth:
  $$\text{newcap} = \text{oldcap} + \frac{\text{oldcap} + 3 \times 256}{4}$$
  This prevents large allocations from wasting too much memory while keeping allocations infrequent.

---

### 1.5 Backing Array Sharing & Mutation Gotchas
When you slice an existing slice (e.g., `b := a[1:3]`), they share the **same backing array**. Modifying elements in `b` will mutate `a`.

```go
a := []int{1, 2, 3, 4}
b := a[1:3] // b = [2, 3]
b[0] = 99   // Modifies the backing array
fmt.Println(a) // Output: [1, 99, 3, 4]
```

However, if you append to `b` and it exceeds `b`'s capacity, `b` will get a **new backing array**. At that point, `a` and `b` are decoupled, and modifications to `b` will no longer affect `a`.

#### Controlling Capacity with 3-Index Slicing
To prevent appending elements from corrupting the original backing array, use the **three-index slice** syntax:
```go
b := a[low:high:max]
```
This sets the capacity of `b` to `max - low`. If you set `max` equal to `high`, the capacity of the new slice is exactly equal to its length. Any subsequent `append()` will immediately force a new allocation, protecting the original slice `a`.

```go
b := a[1:3:3] // len = 2, cap = 2
```

---

## 2. Edge Cases & Limitations

### 2.1 The Sub-Slice Memory Leak
If you have a massive backing array (e.g., a 10MB file loaded in memory) and you take a tiny slice of it (e.g., `header := data[:10]`), the **entire 10MB backing array remains in memory**.
Even though you only reference 10 bytes, the garbage collector (GC) cannot reclaim the backing array because the `header` slice header holds a pointer to it.

```go
var fileData []byte = readHugeFile() // 10MB
var prefix []byte = fileData[0:10]   // Pointer holds 10MB in memory!
```

---

## 3. Real-World Usage & Best Practices

### 3.1 Pre-Allocating Slice Capacity
If you know the size of the slice beforehand, **always pre-allocate capacity** using `make([]T, 0, expectedSize)`. This eliminates multiple heap allocations and copy operations as the slice grows.

### 3.2 Handing Slice Parameters
When passing a slice to a function, the slice header is passed by value (copied). The backing array pointer is copied, meaning the function can mutate elements of the caller's slice. However, if the function appends to the slice and triggers a reallocation, the updated header (pointer, len, cap) is **not** reflected in the caller's variable.

---

## 4. Code Examples: Good vs. Bad

### ❌ Bad: Sub-Slice Memory Leak
```go
package main

import "runtime"

func getFirstID() []byte {
	// Let's simulate a heavy byte array (e.g., 50MB of logs)
	heavyBytes := make([]byte, 50*1024*1024) 
	heavyBytes[0] = 'I'
	heavyBytes[1] = 'D'
	heavyBytes[2] = '-'
	heavyBytes[3] = '1'

	// Slicing keeps the entire 50MB array allocated in memory
	return heavyBytes[0:4] 
}

func main() {
	id := getFirstID()
	runtime.GC() // Garbage collector cannot clean up the 50MB array!
	_ = id
}
```

###  Good: Copying to Release Memory
```go
package main

import "runtime"

func getFirstID() []byte {
	heavyBytes := make([]byte, 50*1024*1024)
	heavyBytes[0] = 'I'
	heavyBytes[1] = 'D'
	heavyBytes[2] = '-'
	heavyBytes[3] = '1'

	// Create a new slice of exact size and copy elements
	cleanedID := make([]byte, 4)
	copy(cleanedID, heavyBytes[0:4])

	return cleanedID // The 50MB array is dereferenced and can be garbage collected
}

func main() {
	id := getFirstID()
	runtime.GC() // Garbage collector successfully reclaims 50MB!
	_ = id
}
```

---

### ❌ Bad: Shared Backing Array Corrupting Data
Appends to sub-slices can overwrite neighboring elements in the original slice.
```go
package main

import "fmt"

func main() {
	original := []int{10, 20, 30, 40, 50}
	
	// Create a sub-slice
	sub := original[1:3] // [20, 30], capacity is 4 (indices 1 to 4)
	
	// Appending overwrites index 3 (40) of the original slice
	sub = append(sub, 99) 
	
	fmt.Println("Original:", original) // Output: [10, 20, 30, 99, 50] (40 got corrupted!)
}
```

###  Good: 3-Index Slicing for Safety
Setting capacity equal to length forces `append` to allocate a new backing array.
```go
package main

import "fmt"

func main() {
	original := []int{10, 20, 30, 40, 50}
	
	// 3-Index slice sets capacity to high-low (3-1 = 2)
	sub := original[1:3:3] // len = 2, cap = 2
	
	// Appending forces a new backing array allocation
	sub = append(sub, 99) 
	
	fmt.Println("Original:", original) // Output: [10, 20, 30, 40, 50] (Safe!)
	fmt.Println("Sub:", sub)           // Output: [20, 30, 99]
}
```
