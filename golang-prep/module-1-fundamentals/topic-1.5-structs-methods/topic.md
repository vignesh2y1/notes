# Topic 1.5: Structs, Methods & Receivers

Go does not have classes or inheritance. Instead, it relies on **Structs** for data encapsulation, **Methods** for behavior, and **Composition (Embedding)** for code reuse. A professional developer must understand how structs are laid out in memory (including padding) and when to use value vs. pointer receivers.

---

## 1. Concepts & Theory

### 1.1 Struct Memory Alignment & Padding
A struct is a contiguous sequence of fields in memory. However, to optimize CPU memory access, Go aligns struct fields based on the size of the computer's word size (typically 8 bytes on 64-bit systems).

#### Memory Alignment Rules:
* Types have an **alignment guarantee** (e.g., a `uint64` must be stored at a memory address that is a multiple of 8; a `uint16` must be at a multiple of 2).
* To satisfy these guarantees, the compiler inserts **padding bytes** (empty space) between fields.

#### Visualizing Padding:
Consider these two structs containing the exact same fields, but in different orders:

```go
type PoorlyAligned struct {
	A int8  // 1 byte
	B int64 // 8 bytes (needs 8-byte alignment)
	C int8  // 1 byte
}
```

* **Memory Layout for `PoorlyAligned` (Total: 24 bytes):**
  * `A` (1 byte) + 7 padding bytes (to align `B` at multiple of 8)
  * `B` (8 bytes)
  * `C` (1 byte) + 7 padding bytes (to align the entire struct size to a multiple of 8)

```go
type WellAligned struct {
	B int64 // 8 bytes
	A int8  // 1 byte
	C int8  // 1 byte
}
```

* **Memory Layout for `WellAligned` (Total: 16 bytes):**
  * `B` (8 bytes)
  * `A` (1 byte)
  * `C` (1 byte) + 6 padding bytes (to pad the total struct size to a multiple of 8)

By ordering fields from **largest to smallest**, we saved 8 bytes of memory per struct instance.

---

### 1.2 Value vs. Pointer Receivers
A method is a function with a special receiver argument:

```go
func (s MyStruct) ValueMethod()   {} // Value Receiver
func (s *MyStruct) PointerMethod() {} // Pointer Receiver
```

#### The Differences:
1. **Mutation:**
   * A **Value Receiver** receives a copy of the struct. Any changes made inside the method mutate the copy and are lost when the method returns.
   * A **Pointer Receiver** receives a pointer to the struct. Changes made inside mutate the original struct instance.
2. **Method Sets & Interfaces (Preview):**
   * If a method is declared with a pointer receiver, it is only in the method set of pointer types (`*MyStruct`). It is **not** in the method set of value types (`MyStruct`).
3. **Copy Overhead:**
   * For large structs, value receivers copy the entire struct, which can degrade performance.

#### Selecting a Receiver Type:
* Use a **pointer receiver** if:
  * The method needs to mutate the receiver.
  * The struct is large (copying is expensive).
  * The struct contains synchronization primitives (like `sync.Mutex` or `sync.WaitGroup`), which must **never** be copied.
* Use a **value receiver** if:
  * The struct is small, immutable, or a basic type.
  * You want to guarantee that the method cannot modify the original state.

---

### 1.3 Struct Embedding (Composition over Inheritance)
Go does not support class inheritance. Instead, it uses **Composition** through anonymous struct embedding.

```go
type Person struct {
    Name string
}

type Developer struct {
    Person // Embedded/Anonymous field
    Level  string
}
```

#### Key Properties of Embedding:
* **Method Promotion:** Fields and methods of the embedded struct `Person` are promoted to the parent struct `Developer`. You can write `dev.Name` and call `dev.Speak()` directly.
* **Name Shadowing:** If the parent struct declares a field/method with the same name as the embedded struct, the parent's field shadows the embedded one. However, the embedded field can still be accessed explicitly: `dev.Person.Name`.

---

## 2. Edge Cases & Limitations

### 2.1 The Pointer Receiver Nil Check
You can call a pointer method on a `nil` pointer. Go does not automatically throw a null pointer exception on the method call itself; the panic only occurs when you attempt to dereference the pointer inside the method.

```go
var u *User // nil
u.PrintName() // Compiles and executes! If PrintName accesses u.Name, it panics there.
```

---

## 3. Real-World Usage & Best Practices

### 3.1 Struct Ordering Optimization
In systems handling millions of memory-resident objects (e.g., caches, database indexing engines), optimization of field alignment saves megabytes of RAM and improves CPU cache-line hits.

---

## 4. Code Examples: Good vs. Bad

### ❌ Bad: Value Receiver Attempting to Mutate State
```go
package main

import "fmt"

type Counter struct {
	Count int
}

// Value receiver makes a copy of Counter
func (c Counter) Increment() {
	c.Count++ 
}

func main() {
	cnt := Counter{Count: 0}
	cnt.Increment()
	fmt.Println("Count is:", cnt.Count) // Output: Count is: 0 (No change!)
}
```

###  Good: Pointer Receiver for Mutating State
```go
package main

import "fmt"

type Counter struct {
	Count int
}

// Pointer receiver mutates the original struct
func (c *Counter) Increment() {
	c.Count++
}

func main() {
	cnt := Counter{Count: 0}
	cnt.Increment() // Go implicitly converts this to (&cnt).Increment()
	fmt.Println("Count is:", cnt.Count) // Output: Count is: 1 (Correct!)
}
```

---

### ❌ Bad: Copying Mutex via Value Receivers
```go
package main

import "sync"

type ThreadSafeCache struct {
	mu   sync.Mutex
	data map[string]string
}

// Value receiver copies the mutex, which breaks synchronization guarantees!
func (c ThreadSafeCache) Get(key string) string {
	c.mu.Lock()
	defer c.mu.Unlock()
	return c.data[key]
}
```

###  Good: Pointer Receiver to Prevent Mutex Copying
```go
package main

import "sync"

type ThreadSafeCache struct {
	mu   sync.Mutex
	data map[string]string
}

// Pointer receiver ensures we refer to the original mutex instance
func (c *ThreadSafeCache) Get(key string) string {
	c.mu.Lock()
	defer c.mu.Unlock()
	return c.data[key]
}
```
*Note: Go's linter (`go vet`) will warn you if you attempt to copy values containing locks.*
