# Solutions: Variables, Constants, and Type System

Below are the step-by-step explanations, fixes, and analysis for the Topic 1.1 exercises.

---

## 🧠 Conceptual Questions

### Solution 1: Zero Value vs. Short Declaration

#### Differences:
* `var count int` declares a variable named `count` of type `int` and initializes it to its **zero value** (`0`). No expression evaluation occurs.
* `count := 0` declares a variable named `count`, infers its type to be `int` based on the right-hand side constant value `0`, and assigns that value.

#### Memory Allocation:
At the machine level, both allocate the exact same amount of space (typically 8 bytes on a 64-bit architecture) and initialize it to all zeros. The difference is solely in compile-time type deduction and code intent.

#### Professional Best Practices:
* **Use `var`** when declaring a variable that will be populated later (e.g., through an API call or JSON decoding) or when you explicitly want the zero value.
  ```go
  var user User // Zero-initialized, ready to be filled by json.Unmarshal
  ```
* **Use `:=`** inside functions when you are declaring and immediately initializing to a **non-zero** value or the result of a function call.
  ```go
  totalPrice := calculateTotal(items)
  ```

---

### Solution 2: Variable Shadowing & Panic Propagation

#### The Bug:
Inside the `if` block in `setupConnection()`, the line:
```go
ConnectionString, err := fetchProductionConnectionString()
```
uses the short declaration operator `:=`. Because `err` is a new variable in this scope, Go compiles this successfully. However, this line does **not** reassign the global variable `ConnectionString`. Instead, it declares a **new local variable** named `ConnectionString` that exists only within the scope of the `if` block.

#### Runtime Behavior:
1. `setupConnection` enters the `if` block.
2. The local `ConnectionString` is created and assigned `"prod_database_url"`.
3. The print statement output will be: `"Inside setup: Connected to prod_database_url"`.
4. The `if` block ends. The local `ConnectionString` goes out of scope and is eligible for garbage collection.
5. The function returns `nil`.
6. `main()` prints `"Main: Connected to default_local_dev"`. The global variable was never updated.

#### The Fix:
Pre-declare the `err` variable so that you can use the assignment operator (`=`) instead of the declaration operator (`:=`).

```go
func setupConnection() error {
	if ConnectionString == "default_local_dev" {
		var err error // Pre-declare error
		// Use assignment (=) instead of declaration (:=)
		ConnectionString, err = fetchProductionConnectionString() 
		if err != nil {
			return err
		}
		fmt.Println("Inside setup: Connected to", ConnectionString)
	}
	return nil
}
```

---

### Solution 3: Constant Resolution & Function Calls

#### Why it Fails to Compile:
Go constants are resolved and evaluated **entirely at compile time**. The compiler must be able to compute the exact value of the constant before generating the executable binary.
`math.Max(10, 20)` is a function call. In Go, functions are executed at **runtime**, even if their inputs are hardcoded constants. Because the compiler cannot execute arbitrary functions during compilation, assigning a runtime function output to a `const` is forbidden.

#### The Fix:
If you need to evaluate values at runtime, you must use a variable (`var`):
```go
var MaxValue = math.Max(10, 20)
```

Alternatively, if the values are compile-time constants, you can use comparison operators directly in your constant expression (or use the built-in compiler-friendly constant arithmetic):
```go
const MaxValue = 20 // Manual evaluation
```

---

## 🛠️ Practical Problems

### Solution 4: Bitwise Shifts and `iota`

#### Code Implementation:
```go
package main

import "fmt"

type ByteSize uint64

const (
	B  ByteSize = 1 << (10 * iota) // 1 << (10 * 0) = 1 (1 Byte)
	KB                             // 1 << (10 * 1) = 1024 (1 KB)
	MB                             // 1 << (10 * 2) = 1,048,576 (1 MB)
	GB                             // 1 << (10 * 3) = 1,073,741,824 (1 GB)
	TB                             // 1 << (10 * 4) = 1,099,511,627,776 (1 TB)
	PB                             // 1 << (10 * 5) = 1,125,899,906,842,624 (1 PB)
)

func main() {
	fmt.Printf("1 KB = %d bytes\n", KB)
	fmt.Printf("1 MB = %d bytes\n", MB)
	fmt.Printf("1 GB = %d bytes\n", GB)
}
```

#### How it Works:
1. On the first line, `iota` has the value `0`. `1 << (10 * 0)` results in `1 << 0`, which is `1`.
2. For subsequent constants, the expression `1 << (10 * iota)` is implicitly repeated.
3. For `KB`, `iota` is `1`. `1 << 10` shifts the bit `1` ten places to the left, which yields `1024`.
4. For `MB`, `iota` is `2`. `1 << 20` yields `1,048,576`.
5. This scales up cleanly, showcasing how `iota` simplifies complex scale patterns.

---

### Solution 5: Explicit Conversions & Calculation Precision

#### The Bug:
In Go, division between two integers always results in an integer, truncating any fractional remainder.
```go
totalSum / count // 145 / 4 = 36 (integer division)
```
Even though the resulting value is wrapped in `float64()`, the precision was already lost before the conversion occurred. It converts the integer `36` to `36.0`.

#### The Fix:
You must convert at least one of the integers to `float64` **before** performing the division. This triggers Go's promotion rule, which promotes the other operand to `float64` and performs floating-point division.

```go
package main

import "fmt"

func main() {
	totalSum := 145
	count := 4

	// Convert operands first, then divide
	var average float64 = float64(totalSum) / float64(count)
	fmt.Println("Average:", average) // Output: Average: 36.25
}
```

---

## 💼 Scenario-Based Questions

### Solution 6: The "Config Loader" Nil Pointer Panic

#### 1. Why it compiles successfully:
```go
GlobalConfig, err := parseConfigFromFile()
```
This line uses the short declaration operator `:=`. Because `err` is a new variable in the local scope, Go allows this block. It declares a new local variable `GlobalConfig` (shadowing the global pointer) and a new local `err`. To prevent the "unused variable" compile-time error, `_ = GlobalConfig` is used, so the code compiles perfectly.

#### 2. Why it panics at runtime:
The global `GlobalConfig` variable remains `nil` because the parsed config was stored in the shadowed local `GlobalConfig` variable which died when the `LoadConfig()` function returned.
When `main()` later calls `GlobalConfig.Port`, it attempts to dereference a `nil` pointer. In Go, dereferencing a `nil` pointer causes a runtime panic.

#### 3. How to fix it:
Pre-declare the `err` variable to ensure you use assignment (`=`) instead of short declaration (`:=`), thereby targeting the package-level global variable.

```go
func LoadConfig() error {
	if GlobalConfig == nil {
		var err error // Pre-declare error variable
		
		// Use assignment (=) to mutate the global variable
		GlobalConfig, err = parseConfigFromFile()
		if err != nil {
			return err
		}
	}
	return nil
}
```
Alternatively, declare a local configuration variable with a different name and assign it to the global variable at the end:
```go
func LoadConfig() error {
	if GlobalConfig == nil {
		cfg, err := parseConfigFromFile()
		if err != nil {
			return err
		}
		GlobalConfig = cfg // Assign to global pointer
	}
	return nil
}
```
