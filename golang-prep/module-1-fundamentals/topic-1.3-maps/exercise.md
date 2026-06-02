# Exercises: Maps & Internals

Test your understanding of Go map internals (hmap/bmap), key compatibility rules, concurrent safety, and memory management.

---

## 🧠 Conceptual Questions

### Question 1: Map Lookup Walkthrough
Assume you look up a value: `v := m["my-key"]`.
Explain the internal steps Go performs:
1. What role does the hash seed (`hash0`) play?
2. How does Go determine which bucket in `hmap.buckets` contains the key?
3. How is `tophash` used to speed up lookup inside the bucket?
4. What happens if the key matches a tophash value but is not equal to the key at that index?
5. How are overflow buckets resolved?

---

### Question 2: Sorting Map Iteration Output
Since Go explicitly randomizes map iteration, write a function `GetSortedKeys(m map[string]int) []string` that returns the keys of the map sorted alphabetically. 
Then, explain the memory and CPU complexity ($O(N)$ notation) of this sorting operation compared to basic map iteration.

---

## 🛠️ Practical Problems

### Question 3: Structs as Map Keys
In Go, structs can be used as map keys if all their fields are comparable. Identify which of the following struct definitions are valid map keys. For those that are invalid, explain why:

```go
type KeyA struct {
	Name string
	Age  int
}

type KeyB struct {
	ID   int
	Tags []string
}

type KeyC struct {
	Metadata map[string]interface{}
}

type KeyD struct {
	Data [5]int
}

type KeyE struct {
	Filter func(int) bool
}
```

---

### Question 4: Concurrent Map: `sync.RWMutex` vs. `sync.Map`
Go provides `sync.Map` for concurrent map operations.
1. What are the two specific use cases where `sync.Map` out-performs a standard map protected by a `sync.RWMutex`?
2. For general use cases (e.g., standard write-heavy cache), why is a standard map wrapped in a `sync.RWMutex` preferred over `sync.Map`?

---

## 💼 Scenario-Based Questions

### Question 5: Websocket Client Map Memory Leak
You are building a real-time websocket server. When a client connects, they are assigned a string UUID, and a reference is stored in a global map:
```go
type Client struct {
	ConnID string
	Buffer []byte // holds messages
}

var ActiveConnections = make(map[string]*Client)
```
During a marketing event, 50,000 users connect simultaneously, and are registered in the map. After the event, all but 10 users disconnect, and you call `delete(ActiveConnections, uuid)` for each disconnected user.
However, monitoring shows that the process memory usage remains extremely high, even though `len(ActiveConnections)` is now `10`.

1. Explain exactly why the memory has not been released.
2. Write a cleanup function `ConsolidateConnections` that resolves this memory leak and recovers memory.
