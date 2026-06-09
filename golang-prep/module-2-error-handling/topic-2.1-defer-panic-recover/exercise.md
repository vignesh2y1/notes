# Exercises: Defer, Panic, and Recover

Test your understanding of defer execution order, argument evaluation timing, panic propagation, recover mechanics, and production cleanup patterns.

---

## 🧠 Conceptual Questions

### Question 1: Defer Argument Evaluation Timing
Predict the exact output of this program. Explain which values are captured at defer-time vs. execution-time.

```go
package main

import "fmt"

func main() {
	x := 1
	defer fmt.Println("A:", x)

	x = 2
	defer func() {
		fmt.Println("B:", x)
	}()

	x = 3
	defer func(val int) {
		fmt.Println("C:", val)
	}(x)

	x = 4
	fmt.Println("D:", x)
}
```

---

### Question 2: Named Return Value Modification
What does this function return when called as `result, err := safeDivide(10, 0)`? Trace through the entire execution flow including the panic, defer, and recover steps.

```go
package main

import "fmt"

func safeDivide(a, b int) (result int, err error) {
	defer func() {
		if r := recover(); r != nil {
			err = fmt.Errorf("recovered: %v", r)
		}
	}()

	result = a / b
	return result, nil
}

func main() {
	result, err := safeDivide(10, 0)
	fmt.Println("Result:", result, "Error:", err)
}
```

---

### Question 3: Panic Propagation Across Goroutines
Explain why the following program **crashes** despite having a `recover()` in `main`. What fundamental rule about goroutines and panics does this violate?

```go
package main

import (
	"fmt"
	"time"
)

func riskyWork() {
	panic("worker exploded")
}

func main() {
	defer func() {
		if r := recover(); r != nil {
			fmt.Println("Main recovered:", r)
		}
	}()

	go riskyWork()

	time.Sleep(1 * time.Second)
	fmt.Println("Main completed")
}
```

---

## 🛠️ Practical Problems

### Question 4: Defer Execution Order with Panic
Predict the exact output of this program. Pay careful attention to the LIFO ordering and which deferred calls execute.

```go
package main

import "fmt"

func alpha() {
	fmt.Println("alpha: start")
	defer fmt.Println("alpha: defer 1")
	defer fmt.Println("alpha: defer 2")
	beta()
	defer fmt.Println("alpha: defer 3") // Does this run?
	fmt.Println("alpha: end")
}

func beta() {
	fmt.Println("beta: start")
	defer fmt.Println("beta: defer 1")
	panic("beta panicked!")
	defer fmt.Println("beta: defer 2") // Does this run?
}

func main() {
	defer func() {
		if r := recover(); r != nil {
			fmt.Println("main: recovered:", r)
		}
	}()

	alpha()
	fmt.Println("main: after alpha") // Does this run?
}
```

---

### Question 5: Fix the Goroutine Panic Handler
The following server spawns goroutines to process jobs. If any goroutine panics, the entire server crashes. Fix the code so that:
1. Each goroutine's panic is caught individually.
2. The panic value and job ID are logged.
3. Other goroutines continue running unaffected.

```go
package main

import (
	"fmt"
	"sync"
)

func processJob(id int) {
	if id == 3 {
		panic(fmt.Sprintf("job %d encountered critical failure", id))
	}
	fmt.Printf("Job %d completed successfully\n", id)
}

func main() {
	var wg sync.WaitGroup

	for i := 1; i <= 5; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			processJob(id)
		}(i)
	}

	wg.Wait()
	fmt.Println("All jobs processed")
}
```

---

### Question 6: The Defer-in-Loop File Descriptor Leak
You are processing 10,000 CSV files. Each file needs to be opened, parsed, and closed. The following code crashes with `too many open files` after ~1000 files.

```go
func processAllCSVs(paths []string) error {
	for _, path := range paths {
		f, err := os.Open(path)
		if err != nil {
			return fmt.Errorf("open %s: %w", path, err)
		}
		defer f.Close()

		if err := parseCSV(f); err != nil {
			return fmt.Errorf("parse %s: %w", path, err)
		}
	}
	return nil
}
```

1. Explain the root cause of the file descriptor exhaustion.
2. Rewrite the function to properly close each file after processing.

---

## 💼 Scenario-Based Questions

### Question 7: Database Transaction Safety Pattern
You are building a banking service that transfers money between accounts. The transfer involves two SQL statements (debit and credit). If either fails, the transaction must be rolled back.

Write a function `TransferMoney(db *sql.DB, fromID, toID int, amount float64) error` that:
1. Begins a database transaction.
2. Executes both the debit and credit queries.
3. Uses `defer` with named return values to automatically rollback on error OR commit on success.
4. Handles the case where `tx.Commit()` itself could return an error.

You may assume these helper SQL statements:
```sql
UPDATE accounts SET balance = balance - $1 WHERE id = $2
UPDATE accounts SET balance = balance + $1 WHERE id = $2
```

---

### Question 8: The Nested Recover Trap
A junior developer wrote this panic handler and claims it works. Explain why it does NOT actually recover from the panic. Then fix it.

```go
package main

import "fmt"

func handlePanic() {
	if r := recover(); r != nil {
		fmt.Println("Caught:", r)
	}
}

func riskyOperation() {
	defer func() {
		handlePanic() // "Delegating" recovery to a helper
	}()

	panic("critical error")
}

func main() {
	riskyOperation()
	fmt.Println("Program continues")
}
```
