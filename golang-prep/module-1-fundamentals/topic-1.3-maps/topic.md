# Topic 1.3: Maps & Internals

Go maps are built-in hash tables. As a 2-year Go developer, you are expected to understand how they function under the hood (buckets, tophash, evacuation), their concurrency limitations, key constraints, and memory retention behavior.

---

## 1. Concepts & Theory

### 1.1 Inside the Go Map: `hmap` and `bmap`
Under the hood, a Go map is a pointer to an `hmap` struct (defined in `runtime/map.go`). 

```go
type hmap struct {
    count     int            // Number of elements (returned by len())
    flags     uint8          // State flags (e.g., map is being written to)
    B         uint8          // Logarithm of number of buckets (hash table has 2^B buckets)
    noverflow uint16         // Approximate number of overflow buckets
    hash0     uint32         // Hash seed (randomized on map creation to prevent collision attacks)
    buckets   unsafe.Pointer // Pointer to array of 2^B buckets
    oldbuckets unsafe.Pointer // Pointer to previous bucket array (used during resizing)
    nevacuate  uintptr        // Evacuation progress counter
    // ... other fields
}
```

#### Bucket Structure (`bmap`):
The `buckets` pointer points to an array of `bmap` structs (commonly called **buckets**). Each bucket holds up to **8 key-value pairs**:

```go
type bmap struct {
    tophash [8]uint8 // Top 8 bits of the hash for each key in this bucket
    // Followed by: 8 keys (contiguous in memory)
    // Followed by: 8 values (contiguous in memory)
    // Followed by: Overflow bucket pointer (if any)
}
```

* **Why Contiguous Keys then Values?**
  Instead of grouping key1-value1, key2-value2, Go groups all 8 keys together followed by all 8 values. This layout avoids padding byte waste due to memory alignment constraints (e.g., if you have `map[int64]int8`).
* **Tophash:** 
  For each key, Go hashes it. The top 8 bits of the hash are stored in the `tophash` array. During lookups, Go compares `tophash` values first. This is a fast CPU register check that avoids expensive full key comparisons if the hash bits don't match.

```
+-------------------------------------------------------+
|                    bmap (Bucket)                      |
+-------------------------------------------------------+
| tophash: [H1, H2, H3, H4, H5, H6, H7, H8]             |
+-------------------------------------------------------+
| keys:    [K1, K2, K3, K4, K5, K6, K7, K8]             |
+-------------------------------------------------------+
| values:  [V1, V2, V3, V4, V5, V6, V7, V8]             |
+-------------------------------------------------------+
| overflow pointer: *bmap                               |
+-------------------------------------------------------+
```

---

### 1.2 Map Resizing & Incremental Evacuation
When a map grows too crowded, Go resizes it to keep lookups fast ($O(1)$).

#### The Resizing Triggers:
1. **Load Factor Exceeded:** The load factor represents the average number of items per bucket. In Go, the threshold is **6.5**. If $\frac{\text{count}}{\text{buckets}} > 6.5$, a growth is triggered.
2. **Too Many Overflow Buckets:** If there are too many overflow buckets, Go triggers a **same-size growth** to clean up the memory layout and consolidate sparse buckets.

#### Incremental Evacuation:
Allocating a new double-sized bucket array and copying all elements at once would cause a massive latency spike (a "stop-the-world" style delay). To prevent this, **Go resizes maps incrementally**.
* During growth, Go allocates a new bucket array (`buckets`) and moves the old pointer to `oldbuckets`.
* Evacuation occurs one bucket at a time, triggered dynamically whenever a write (`map[key] = val`) or delete (`delete(m, key)`) is executed on the map.
* Reads will check `oldbuckets` if the target bucket has not yet been evacuated.

---

### 1.3 Key Constraints: Comparable Types Only
In Go, map keys must be **comparable**. This means the key type must support the `==` and `!=` operators so Go can check for equality during lookup collisions.

* **Comparable Keys:** Integers, floats, booleans, strings, pointers, channels, interfaces, and structs (if all fields inside the struct are comparable).
* **Non-Comparable Keys:** Slices, maps, and functions. Attempts to use these as map keys result in a compiler error.

---

### 1.4 Nil Maps vs. Empty Maps
* **Nil Map (`var m map[string]int`):** 
  * Points to `nil`.
  * **Reading is safe:** `val := m["key"]` returns the zero value (`0`).
  * **Writing panics:** `m["key"] = 1` triggers a `panic: assignment to entry in nil map`.
* **Empty Map (`m := make(map[string]int)`):**
  * Properly allocates an `hmap` header.
  * Reading and writing are both safe.

---

### 1.5 Random Map Iteration
When iterating over a map using `for key, val := range m`, **the order is randomized**.
Go does not guarantee stable map ordering. In early Go versions, developers accidentally relied on iteration order, which broke when maps resized or Go versions changed. To prevent this, the runtime explicitly randomizes the starting bucket and offset during iteration setup.

---

### 1.6 Concurrency Limitations
Go maps are **not thread-safe**. 
* Multiple goroutines can **read** a map concurrently.
* If any goroutine is **writing** to a map while others are writing or reading it, the Go runtime crashes instantly with a non-recoverable error:
  `fatal error: concurrent map read and map write`
* Note that this is a **runtime crash**, not a standard panic. It cannot be recovered using `recover()`.

To protect maps, you must use synchronization (like `sync.RWMutex`) or `sync.Map`.

---

## 2. Edge Cases & Limitations

### 2.1 The Map Memory Leak (Memory Retention)
Go maps only grow in memory size; **they never shrink**.
If you add 1,000,000 items to a map, the map allocates thousands of buckets. If you subsequently delete all 1,000,000 items using `delete()`, the map's `count` returns to `0`, but **the allocated buckets remain in memory**. Go does not release the buckets back to the OS or the GC.

---

## 3. Real-World Usage & Best Practices

### 3.1 Pre-Allocating Map Capacity
If you know the approximate number of keys your map will hold, pre-allocate it:
```go
m := make(map[string]int, 10000)
```
This allocates the correct number of buckets upfront, avoiding costly incremental resizes and allocations during runtime.

### 3.2 Solving the Map Memory Leak
If a map undergoes a peak workload where it holds millions of keys, and then shrinks, you must manually discard the old map and copy any remaining elements to a new one to reclaim memory:
```go
func shrinkMap(largeMap map[string]Value) map[string]Value {
    newMap := make(map[string]Value, len(largeMap))
    for k, v := range largeMap {
        newMap[k] = v
    }
    return newMap
}
```

---

## 4. Code Examples: Good vs. Bad

### ❌ Bad: Writing to a Nil Map
```go
package main

import "fmt"

type Cache struct {
	items map[string]string // Nil by default
}

func main() {
	c := Cache{}
	// Panic: assignment to entry in nil map
	c.items["token"] = "xyz" 
	fmt.Println(c.items["token"])
}
```

###  Good: Explicit Map Initialization
```go
package main

import "fmt"

type Cache struct {
	items map[string]string
}

func NewCache() *Cache {
	return &Cache{
		items: make(map[string]string), // Initialize empty map
	}
}

func main() {
	c := NewCache()
	c.items["token"] = "xyz" // Safe
	fmt.Println(c.items["token"])
}
```

---

### ❌ Bad: Unsynchronized Concurrent Map Access
```go
package main

import "time"

func main() {
	m := make(map[int]int)

	// Goroutine 1: Write
	go func() {
		for i := 0; i < 1000; i++ {
			m[i] = i
		}
	}()

	// Goroutine 2: Read
	go func() {
		for i := 0; i < 1000; i++ {
			_ = m[i] // Will crash with concurrent read/write
		}
	}()

	time.Sleep(1 * time.Second)
}
```

###  Good: Thread-Safe Map Wrapper using RWMutex
```go
package main

import (
	"sync"
	"time"
)

type SafeMap struct {
	mu sync.RWMutex
	m  map[int]int
}

func (s *SafeMap) Set(k, v int) {
	s.mu.Lock()         // Lock for writing
	defer s.mu.Unlock()
	s.m[k] = v
}

func (s *SafeMap) Get(k int) (int, bool) {
	s.mu.RLock()         // Read Lock for concurrency
	defer s.mu.RUnlock()
	val, ok := s.m[k]
	return val, ok
}

func main() {
	sm := &SafeMap{m: make(map[int]int)}

	go func() {
		for i := 0; i < 1000; i++ {
			sm.Set(i, i)
		}
	}()

	go func() {
		for i := 0; i < 1000; i++ {
			_, _ = sm.Get(i)
		}
	}()

	time.Sleep(500 * time.Millisecond)
}
```
