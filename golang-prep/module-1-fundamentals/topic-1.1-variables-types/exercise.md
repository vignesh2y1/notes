# Exercises: Variables, Constants, and Type System

Test your understanding of Go's type system, scoping rules, variable shadowing, constants, and `iota`. Try to answer these questions without looking at the solutions file or running the code first.

---

## 🧠 Conceptual Questions

### Question 1: Zero Value vs. Short Declaration
Explain the difference in meaning, memory initialization, and usage between these two declarations inside a function:
```go
var count int
```
vs.
```go
count := 0
```
When should a professional Go developer prefer one over the other?

---

### Question 2: Variable Shadowing & Panic Propagation
Explain why the following code compiles but exhibits a critical runtime bug. Explain exactly what variable is shadowed and what the behavior will be.
```go
package main

import (
	"errors"
	"fmt"
)

var ConnectionString = "default_local_dev"

func setupConnection() error {
	if ConnectionString == "default_local_dev" {
		ConnectionString, err := fetchProductionConnectionString()
		if err != nil {
			return err
		}
		fmt.Println("Inside setup: Connected to", ConnectionString)
	}
	return nil
}

func fetchProductionConnectionString() (string, error) {
	return "prod_database_url", nil
}

func main() {
	setupConnection()
	fmt.Println("Main: Connected to", ConnectionString)
}
```

---

### Question 3: Constant Resolution & Function Calls
Why does the following code fail to compile? What is the core rule regarding constant values in Go?
```go
package main

import (
	"fmt"
	"math"
)

const MaxValue = math.Max(10, 20)

func main() {
	fmt.Println(MaxValue)
}
```

---

## 🛠️ Practical Problems

### Question 4: Bitwise Shifts and `iota`
Write the constant definition block using a custom type `ByteSize` (backed by `uint64`) and `iota` to represent memory units:
* `B` = 1 Byte
* `KB` = 1024 Bytes
* `MB` = 1024 KB
* `GB` = 1024 MB
* `TB` = 1024 GB
* `PB` = 1024 TB

Use bitwise shift operators (e.g., `1 << (10 * iota)`) or a variation of it to define these. 

---

### Question 5: Explicit Conversions & Calculation Precision
Consider the following calculation for an average value. What is the bug here, and how do you modify the expression using explicit type casting to get the correct floating-point result?
```go
package main

import "fmt"

func main() {
	totalSum := 145 // int
	count := 4      // int

	// We want the average as a float64
	var average float64 = float64(totalSum / count)
	fmt.Println("Average:", average) // Output: Average: 36 (Expected: 36.25)
}
```

---

## 💼 Scenario-Based Questions

### Question 6: The "Config Loader" Nil Pointer Panic
You are building a backend service. You have a global pointer variable to a configuration struct:
```go
type Config struct {
    Port int
    DB   string
}

var GlobalConfig *Config
```
In your setup code, you write the following loader:
```go
func LoadConfig() error {
    if GlobalConfig == nil {
        GlobalConfig, err := parseConfigFromFile()
        if err != nil {
            return err
        }
        _ = GlobalConfig
    }
    return nil
}
```
Explain:
1. Why this code compiles successfully.
2. Why calling `LoadConfig()` followed by accessing `GlobalConfig.Port` in `main()` causes a `nil pointer dereference` panic.
3. How to fix the `LoadConfig` function so that it successfully updates the global variable.
