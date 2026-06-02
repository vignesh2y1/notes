# Topic 1.1: Variables, Constants, and Type System

In Go, simplicity and safety are the core tenets. This topic covers the details of variable declarations, constants, compilation semantics, and Go's static type system, with a focus on memory layout and production-level gotchas.

---

## 1. Concepts & Theory

### 1.1 Variable Declaration: `var` vs `:=`

Go provides two main ways to declare variables:

1. **The `var` Keyword:** Used for explicit declarations, particularly when you want to initialize a variable to its **zero value** or declare a variable at the package level (global scope).
2. **Short Variable Declaration (`:=`):** Used inside functions for concise declaration and initialization. It infers the type automatically.

#### Rules and Scope:

* **Package-Level Scope:** Only `var` is allowed at the package level. The `:=` operator cannot be used outside functions.
* **Block Scope:** Variables declared inside `{}` are block-scoped and cannot be accessed outside.
* **Re-declaration with `:=`:** The `:=` operator can re-declare variables in the same block *only if* at least one new variable is being declared on the left-hand side. Otherwise, it causes a compilation error.

### 1.2 Zero Values: Memory Safety First

Unlike languages like C/C++, where uninitialized variables contain whatever garbage was previously in that memory location, Go guarantees memory safety by automatically initializing every declared variable to its **Zero Value**.

| Type                                                    | Zero Value                                              |
| :------------------------------------------------------ | :------------------------------------------------------ |
| `int`, `float`, `byte`, `rune`                  | `0` / `0.0`                                         |
| `string`                                              | `""` (empty string)                                   |
| `bool`                                                | `false`                                               |
| Pointers, Slices, Maps, Channels, Interfaces, Functions | `nil`                                                 |
| Structs                                                 | All fields recursively initialized to their zero values |

#### Memory / System-Level Explanation:

When Go allocates memory for a variable (either on the stack or the heap), it zeros out the allocated memory block. This prevents security vulnerabilities related to reading uninitialized memory.

---

### 1.3 Go's Strict Type System & Explicit Conversions

Go is **statically typed** and does **not** perform any implicit type conversions (coercion), even if the conversion is safe (e.g., from `int32` to `int64`).

```go
var a int32 = 42
var b int64 = a // Compiler Error: cannot use a (type int32) as type int64 in assignment
```

To assign `a` to `b`, you must perform an **explicit conversion**:

```go
var b int64 = int64(a)
```

#### Why No Implicit Conversion?

Implicit conversions are a frequent source of bugs in languages like C++ and Java (e.g., precision loss, overflow, unexpected comparison behavior). Go forces developers to be explicit about type size conversions, making code readability and correctness paramount.

---

### 1.4 Constants: Typed vs. Untyped

Constants in Go are declared using the `const` keyword. They are values that are known at compile-time and cannot be changed at runtime.

#### The Magic of Untyped Constants:

In Go, constants can be **untyped**. An untyped constant does not have a fixed size or limit in the way variable types do. They exist in a high-precision numeric space (at least 256 bits for numeric values).

```go
const Pi = 3.14159265358979323846 // Untyped float constant
```

Because `Pi` is untyped, it can be assigned to any compatible floating-point or complex type without explicit conversion:

```go
var f32 float32 = Pi // Allowed
var f64 float64 = Pi // Allowed
```

If `Pi` were declared as `const Pi float64 = 3.14...`, assigning it to `float32` would trigger a compiler error.

#### Compile-Time Restrictions:

Constant values must be resolvable at compile time. This means you **cannot** assign the result of a runtime function call to a constant.

```go
const myTime = time.Now() // Compiler Error: time.Now() is not a constant expression
```

---

### 1.5 The `iota` Constant Generator

The keyword `iota` is a predefined identifier used to simplify definitions of incrementing numbers in constant blocks. It starts at `0` and increments by `1` for each subsequent constant line in a `const` block.

```go
const (
    A = iota // 0
    B        // 1 (implicitly B = iota)
    C        // 2 (implicitly C = iota)
)
```

`iota` resets to `0` whenever the keyword `const` appears again.

---

## 2. Edge Cases & Limitations

### 2.1 Variable Shadowing (The Silent Bug)

Variable shadowing occurs when a variable declared in an inner scope has the same name as a variable in an outer scope. The inner variable "shadows" the outer one, making it temporarily inaccessible.

This frequently happens when using `:=` alongside error handling inside `if` or `for` blocks.

```go
var user string = "Guest"

if authenticated {
    // BUG: := creates a NEW local variable 'user' inside this block
    user, err := fetchUsername() 
    _ = err
}
// Outer 'user' is still "Guest" here!
```

---

### 2.2 Constant Overflows

While untyped constants can have arbitrary precision during compile-time, they will overflow if they are assigned to a variable type that cannot fit them.

```go
const Huge = 1 << 100 // OK: Untyped constant can hold massive values
var smallInt int = Huge // Compiler Error: constant 1267650600228229401496703205376 overflows int
```

---

## 3. Real-World Usage & Best Practices

### 3.1 State Machines and Enums with `iota`

Go does not have a dedicated `enum` type. Instead, we group constants together and use custom types.
Always define a custom type name for your enum to ensure type-safe function arguments.

```go
type OrderStatus int

const (
    StatusPending OrderStatus = iota
    StatusProcessing
    StatusShipped
    StatusCancelled
)
```

### 3.2 Managing Bitmasks (Bitwise Flags)

`iota` can be combined with bitwise shift operators to build flags (e.g., read, write, execute permissions).

```go
type Permission int

const (
    Read    Permission = 1 << iota // 1 (0001)
    Write                          // 2 (0010)
    Execute                        // 4 (0100)
)
```

---

## 4. Code Examples: Good vs. Bad

### ❌ Bad: Shadowing Outer Variables

```go
package main

import (
	"fmt"
	"log"
)

var DatabaseURL = "localhost:5432" // Global variable

func connect() {
	// Shadowing the global DatabaseURL
	DatabaseURL, err := getDatabaseConfig() // Declares a local DatabaseURL
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println("Connecting to local db:", DatabaseURL)
}

func getDatabaseConfig() (string, error) {
	return "production-db:5432", nil
}

func main() {
	connect()
	// Global DatabaseURL remains unchanged!
	fmt.Println("Global DB URL:", DatabaseURL) 
}
```

### Good: Reassigning Properly Without Shadowing

To avoid shadowing, declare variables explicitly first, or use an helper variable before assigning back to global.

```go
package main

import (
	"fmt"
	"log"
)

var DatabaseURL = "localhost:5432"

func connect() {
	var err error
	// Use = instead of := to modify the global variable
	DatabaseURL, err = getDatabaseConfig() 
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println("Connecting to db:", DatabaseURL)
}

func getDatabaseConfig() (string, error) {
	return "production-db:5432", nil
}

func main() {
	connect()
	fmt.Println("Global DB URL is now updated:", DatabaseURL)
}
```

---

### ❌ Bad: Untyped Constants Without Custom Types (Enums)

Using plain integers for state variables allows callers to pass invalid integers.

```go
package main

import "fmt"

func ProcessOrder(status int) {
	fmt.Println("Processing status:", status)
}

func main() {
	// Caller can pass any arbitrary int, bypassing validation logic
	ProcessOrder(9999) 
}
```

### Good: Type-Safe Enums

```go
package main

import "fmt"

type OrderStatus int

const (
	StatusPending OrderStatus = iota
	StatusProcessing
	StatusCompleted
)

func ProcessOrder(status OrderStatus) {
	// Compiler forces caller to pass OrderStatus, not random integers
	fmt.Println("Processing status:", status)
}

func main() {
	ProcessOrder(StatusCompleted)
	// ProcessOrder(9999) will fail compilation if we try to pass a raw typed variable
}
```
