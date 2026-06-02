# Solutions: Maps & Internals

Below are the step-by-step solutions and explanations for the Topic 1.3 exercises.

---

## 🧠 Conceptual Questions

### Solution 1: Map Lookup Walkthrough

When you call `v := m["my-key"]`, the Go runtime executes the following operations under the hood:

1. **The Hash Seed (`hash0`):**
   Go computes the hash value of `"my-key"` using the hash function determined by the key type (e.g., AES hash). The `hash0` seed is a random integer generated when the map is initialized. It is mixed into the hashing function to ensure that the hash values of keys are randomized for each run of the application. This prevents **Hash Collision Attacks** (where an attacker crafts input keys that all resolve to the same bucket index, degrading lookup performance from $O(1)$ to $O(N)$).
2. **Bucket Selection:**
   Go takes the lower-order bits of the computed hash. If the map has $2^B$ buckets, Go inspects the last $B$ bits of the hash. This numerical value is used directly as the index to access the correct bucket inside the `hmap.buckets` array.
3. **Tophash Matching:**
   Go takes the top 8 bits of the same hash value (called the `tophash`). Inside the selected bucket (`bmap`), there is an array of eight 1-byte elements: `tophash [8]uint8`. Go scans this array to find any index where the stored `tophash` matches the key's `tophash`. This byte-matching check is extremely fast and can be optimized by CPU vector instructions.
4. **Key Equality Check:**
   If a matching `tophash` is found at index `i`, Go calculates the memory offset to access the actual key stored at index `i` of the bucket's keys array. It performs a strict equality comparison (`storedKey == "my-key"`). 
   * If the keys are equal, Go retrieves the value at index `i` from the values block and returns it.
   * If the keys are not equal (a hash collision occurred where the top 8 bits matched but the full keys differed), Go continues scanning the remaining `tophash` slots.
5. **Overflow Buckets:**
   If Go finishes scanning all 8 slots in the current bucket and the key has not been found, it reads the bucket's `overflow` pointer. If it is not `nil`, Go traverses down the linked list to the next overflow bucket and repeats steps 3 and 4. If the overflow pointer is `nil`, the key is not in the map, and Go returns the zero value for the value type.

---

### Solution 2: Sorting Map Iteration Output

#### Code Implementation:
```go
package main

import (
	"fmt"
	"sort"
)

func GetSortedKeys(m map[string]int) []string {
	// Pre-allocate slice with the exact length of the map to prevent resizing
	keys := make([]string, 0, len(m))
	
	// Collect keys
	for k := range m {
		keys = append(keys, k)
	}
	
	// Sort keys alphabetically
	sort.Strings(keys)
	
	return keys
}

func main() {
	scores := map[string]int{
		"Charlie": 85,
		"Alice":   95,
		"Bob":     90,
	}

	sortedKeys := GetSortedKeys(scores)
	for _, key := range sortedKeys {
		fmt.Printf("%s: %d\n", key, scores[key])
	}
}
```

#### Complexity Analysis:
* **Time Complexity:**
  1. Extracting the keys from the map takes $O(N)$ time as it visits every key exactly once.
  2. Sorting the slice of $N$ string keys takes $O(N \log N)$ comparison operations on average.
  3. Total Time Complexity is **$O(N \log N)$**.
* **Space Complexity:**
  We allocate a slice of strings to hold the keys of the map. This requires **$O(N)$** auxiliary space in memory.
* **Comparison to Basic Iteration:**
  A direct map iteration using `for range` runs in **$O(N)$ time** and **$O(1)$ auxiliary space**. Sorting map output introduces extra CPU overhead ($O(N \log N)$) and heap allocation overhead ($O(N)$ space).

---

## 🛠️ Practical Problems

### Solution 3: Structs as Map Keys

* **`KeyA` (Valid):**
  Fields are `string` and `int`. Both types are comparable, meaning the struct can be compared using `==`.
* **`KeyB` (Invalid):**
  Contains `Tags []string`. In Go, slices are not comparable (they do not support `==`). Therefore, any struct containing a slice cannot be a map key.
* **`KeyC` (Invalid):**
  Contains `Metadata map[string]interface{}`. Maps are not comparable. Therefore, `KeyC` is invalid.
* **`KeyD` (Valid):**
  Contains `Data [5]int`. Although arrays are sometimes confused with slices, **arrays in Go are comparable** if their element types are comparable. The size `[5]` is part of the type, and it will compare each element sequentially.
* **`KeyE` (Invalid):**
  Contains `Filter func(int) bool`. Functions are not comparable, so they cannot be used inside map keys.

---

### Solution 4: Concurrent Map: `sync.RWMutex` vs. `sync.Map`

#### 1. When `sync.Map` Out-Performs standard map + Mutex:
`sync.Map` is a highly specialized concurrent map optimized for two specific scenarios:
* **Read-Heavy / Append-Only workloads:** When the entry for a given key is only written once but read many times (e.g., caches that only grow or configurations read frequently).
* **Disjoint Key Spaces:** When multiple goroutines read, write, and overwrite entries for disjoint sets of keys (e.g., session stores where each thread accesses a unique session ID).
Under these scenarios, `sync.Map` avoids lock contention by separating updates into a lock-free "read" map and a locked "dirty" map, using atomic CPU instructions.

#### 2. Why standard map + `sync.RWMutex` is preferred for general use:
* **Type Safety:** `sync.Map` works with `interface{}` (or `any` in Go 1.18+). It does not support generics directly, meaning you lose compile-time type safety and must perform type assertions when retrieving values.
* **Heap Allocations:** Every insert/update on a `sync.Map` wraps values in an `interface{}`, which forces variables to escape to the heap, causing significant garbage collection pressure.
* **Write-Heavy Performance:** If the workload is write-heavy or has constant updates to existing keys, `sync.Map` repeatedly falls back to locking, making it slower and more CPU-intensive than a standard map protected by a simple `sync.RWMutex`.

---

## 💼 Scenario-Based Questions

### Solution 5: Websocket Client Map Memory Leak

#### 1. Why Memory is Not Released:
When you delete keys using `delete(ActiveConnections, uuid)`, Go decreases the `count` field of the `hmap` struct and clears the key-value references in the bucket. However, **Go does not shrink the `buckets` pointer array**.
If 50,000 clients connect, the map resizes to contain thousands of buckets. When you delete 49,990 clients, the map still references the same massive bucket array in memory. Even though the bucket contents are zeroed out (releasing the `*Client` structs to the garbage collector), the bucket structures themselves are held. The memory used for the map header and buckets remains allocated at its peak.

#### 2. Consolidation/Shrink Implementation:
To free this memory, you must copy the remaining active keys into a new map that is sized exactly to the remaining element count, and discard the old map:

```go
func ConsolidateConnections() {
	// 1. Check if the map is empty or small to avoid unnecessary copies
	if len(ActiveConnections) == 0 {
		ActiveConnections = make(map[string]*Client)
		return
	}
	
	// 2. Allocate a new map with the exact capacity needed
	newMap := make(map[string]*Client, len(ActiveConnections))
	
	// 3. Copy the remaining active connections
	for uuid, client := range ActiveConnections {
		newMap[uuid] = client
	}
	
	// 4. Swap the reference. The old map and its buckets are now dereferenced
	// and can be fully garbage collected.
	ActiveConnections = newMap
}
```
In production websocket servers, this consolidation function is typically run inside a background goroutine (a worker) at regular intervals (e.g., every 15 minutes) or when active connection count falls below a certain percentage of peak capacity.
