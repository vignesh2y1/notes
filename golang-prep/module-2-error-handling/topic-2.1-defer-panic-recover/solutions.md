# Solutions: Defer, Panic, and Recover

Below are the step-by-step explanations, fixes, and analysis for the Topic 2.1 exercises.

---

## 🧠 Conceptual Questions

### Solution 1: Defer Argument Evaluation Timing

#### Output:
```
D: 4
C: 3
B: 4
A: 1
```

#### Step-by-Step Explanation:

1. `x = 1` → `defer fmt.Println("A:", x)` — `x` is evaluated **immediately** as an argument. The value `1` is captured. This is Rule 1: arguments are evaluated at defer time.

2. `x = 2` → `defer func() { fmt.Println("B:", x) }()` — This is a **closure** with no arguments. The closure captures the *variable* `x` by reference, not its value. It will read whatever `x` is when it executes.

3. `x = 3` → `defer func(val int) { fmt.Println("C:", val) }(x)` — `x` is passed as an **argument** to the anonymous function. The value `3` is captured immediately.

4. `x = 4` → `fmt.Println("D:", 4)` — This runs immediately. Prints `D: 4`.

5. Function exits. Deferred calls execute in **LIFO** order:
   - **C: 3** — Captured argument value `3`.
   - **B: 4** — Closure reads current `x`, which is `4`.
   - **A: 1** — Captured argument value `1`.

#### Key Takeaway:
- Direct arguments to `defer` → captured at defer time.
- Closures → read the variable at execution time (by reference).
- Closures with arguments → captured at defer time (by value).

---

### Solution 2: Named Return Value Modification

#### Output:
```
Result: 0 Error: recovered: runtime error: integer divide by zero
```

#### Execution Trace:

1. `safeDivide(10, 0)` is called. Named return values are initialized: `result = 0`, `err = nil`.
2. The deferred recovery function is registered.
3. `result = a / b` → `10 / 0` triggers a **runtime panic**: `runtime error: integer divide by zero`.
4. Normal execution stops. The `return result, nil` line is **never reached**.
5. The deferred function runs:
   - `recover()` catches the panic and returns the panic value.
   - `r` is not nil, so `err` is set to `fmt.Errorf("recovered: %v", r)`.
   - `result` was never modified from its zero value `0`.
6. The function returns with named values: `result = 0`, `err = "recovered: ..."`.

#### Why This Works:
The deferred closure has access to the named return values `result` and `err` by reference. When `recover()` succeeds, the function returns normally with whatever values are in the named returns — making this pattern perfect for converting panics into errors.

---

### Solution 3: Panic Propagation Across Goroutines

#### Why It Crashes:
**Each goroutine has its own independent call stack.** A `recover()` can only catch panics on the **same** call stack where it is deferred.

The `defer`/`recover` in `main()` operates on `main`'s goroutine stack. The `go riskyWork()` call creates a **new goroutine** with its own stack. When `riskyWork()` panics, the panic unwinds `riskyWork()`'s stack. There are no deferred recovery functions on that stack, so the panic reaches the top of the goroutine and **crashes the entire program**.

#### The Rule:
> A goroutine's panic can only be recovered by a `recover()` called within a deferred function **on that same goroutine's stack**.

#### The Fix:
Each goroutine must include its own recovery:
```go
go func() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("Worker recovered:", r)
        }
    }()
    riskyWork()
}()
```

---

## 🛠️ Practical Problems

### Solution 4: Defer Execution Order with Panic

#### Output:
```
alpha: start
beta: start
beta: defer 1
alpha: defer 2
alpha: defer 1
main: recovered: beta panicked!
```

#### Step-by-Step Trace:

1. `main()` registers its deferred recovery function.
2. `alpha()` starts → prints `"alpha: start"`.
3. `alpha()` registers `defer "alpha: defer 1"` (stack position 1).
4. `alpha()` registers `defer "alpha: defer 2"` (stack position 2).
5. `alpha()` calls `beta()`.
6. `beta()` starts → prints `"beta: start"`.
7. `beta()` registers `defer "beta: defer 1"`.
8. `panic("beta panicked!")` fires.
9. `beta()` stops. `defer "beta: defer 2"` is **never registered** (code after panic doesn't execute).
10. `beta()`'s deferred stack runs: prints `"beta: defer 1"`.
11. Panic propagates to `alpha()`. Execution **does not continue** in `alpha()`, so `defer "alpha: defer 3"` is **never registered**.
12. `alpha()`'s deferred stack runs in LIFO: prints `"alpha: defer 2"`, then `"alpha: defer 1"`.
13. Panic propagates to `main()`.
14. `main()`'s deferred recovery runs: `recover()` catches the panic, prints `"main: recovered: beta panicked!"`.
15. `"main: after alpha"` is **never printed** because the panic unwinding skipped past it.

#### Key Insights:
- Deferred calls registered **before** the panic execute. Deferred calls in code **after** the panic are never registered.
- After a panic is recovered, the function that recovered it returns normally — but execution does **not** resume at the point of panic.

---

### Solution 5: Fix the Goroutine Panic Handler

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

func safeProcessJob(id int) {
	defer func() {
		if r := recover(); r != nil {
			fmt.Printf("⚠️ Job %d panicked: %v\n", id, r)
		}
	}()
	processJob(id)
}

func main() {
	var wg sync.WaitGroup

	for i := 1; i <= 5; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			safeProcessJob(id) // Each goroutine recovers its own panics
		}(i)
	}

	wg.Wait()
	fmt.Println("All jobs processed")
}
```

#### How It Works:
1. Each goroutine wraps `processJob` inside `safeProcessJob`, which has its own `defer`/`recover`.
2. When job 3 panics, only that goroutine's stack unwinds. `recover()` catches the panic inside the same goroutine.
3. All other goroutines continue executing normally.
4. `wg.Wait()` blocks until all goroutines (including the recovered one) complete.

#### Alternative: Inline recovery without helper function:
```go
go func(id int) {
    defer wg.Done()
    defer func() {
        if r := recover(); r != nil {
            fmt.Printf("⚠️ Job %d panicked: %v\n", id, r)
        }
    }()
    processJob(id)
}(i)
```

---

### Solution 6: The Defer-in-Loop File Descriptor Leak

#### 1. Root Cause:
`defer f.Close()` is inside a `for` loop. All deferred calls belong to the **enclosing function** (`processAllCSVs`), not the loop iteration. This means:
- Iteration 1: opens file1, defers Close → file1 stays open.
- Iteration 2: opens file2, defers Close → file1 AND file2 stay open.
- Iteration 1000: opens file1000 → all 1000 files are still open.
- Eventually the OS limit (often 1024 on Linux) is hit, and `os.Open` returns `too many open files`.

#### 2. Fix — Extract Into Helper Function:
```go
func processAllCSVs(paths []string) error {
	for _, path := range paths {
		if err := processOneCSV(path); err != nil {
			return err
		}
	}
	return nil
}

func processOneCSV(path string) error {
	f, err := os.Open(path)
	if err != nil {
		return fmt.Errorf("open %s: %w", path, err)
	}
	defer f.Close() // Closes when processOneCSV returns (each iteration)

	if err := parseCSV(f); err != nil {
		return fmt.Errorf("parse %s: %w", path, err)
	}
	return nil
}
```

Now each file is opened and closed within the scope of `processOneCSV`. At most 1 file is open at a time.

---

## 💼 Scenario-Based Questions

### Solution 7: Database Transaction Safety Pattern

```go
package main

import (
	"database/sql"
	"fmt"
)

func TransferMoney(db *sql.DB, fromID, toID int, amount float64) (err error) {
	tx, err := db.Begin()
	if err != nil {
		return fmt.Errorf("begin transaction: %w", err)
	}

	// Deferred cleanup: rollback on error, commit on success
	defer func() {
		if err != nil {
			// An error occurred in one of the queries
			if rbErr := tx.Rollback(); rbErr != nil {
				err = fmt.Errorf("rollback failed: %v (original error: %w)", rbErr, err)
			}
		} else {
			// All queries succeeded, attempt commit
			if commitErr := tx.Commit(); commitErr != nil {
				err = fmt.Errorf("commit failed: %w", commitErr)
			}
		}
	}()

	// Debit the source account
	_, err = tx.Exec(
		"UPDATE accounts SET balance = balance - $1 WHERE id = $2",
		amount, fromID,
	)
	if err != nil {
		return fmt.Errorf("debit account %d: %w", fromID, err)
	}

	// Credit the destination account
	_, err = tx.Exec(
		"UPDATE accounts SET balance = balance + $1 WHERE id = $2",
		amount, toID,
	)
	if err != nil {
		return fmt.Errorf("credit account %d: %w", toID, err)
	}

	return nil // err is nil → defer will commit
}
```

#### How It Works:
1. The function uses **named return** `err error`, allowing the deferred closure to inspect and modify the error.
2. If any query sets `err` to non-nil and the function returns, the deferred closure sees `err != nil` and calls `tx.Rollback()`.
3. If all queries succeed, `err` is `nil` when defer runs, so it calls `tx.Commit()`.
4. If `tx.Commit()` itself fails (e.g., serialization conflict), the error is captured and returned.
5. This pattern ensures the transaction is **never left hanging** regardless of early returns, panics, or commit failures.

---

### Solution 8: The Nested Recover Trap

#### Why It Doesn't Work:
The Go specification states that `recover()` only stops a panicking sequence **if it is called directly by a deferred function**. In this code:

```go
defer func() {
    handlePanic() // recover() is inside handlePanic, NOT directly in the deferred function
}()
```

`recover()` is called inside `handlePanic()`, which is a **nested function call** within the deferred function. Since `recover()` is not called directly in the deferred function body, it returns `nil` and has no effect. The panic continues propagating and crashes the program.

#### The Fix:
Call `recover()` directly inside the deferred anonymous function:

```go
func riskyOperation() {
	defer func() {
		if r := recover(); r != nil {
			fmt.Println("Caught:", r)
		}
	}()

	panic("critical error")
}
```

#### Alternative Fix — If You Want Helper Function Reuse:
Pass the recover result as a parameter:
```go
func handlePanic(r interface{}) {
	if r != nil {
		fmt.Println("Caught:", r)
	}
}

func riskyOperation() {
	defer func() {
		handlePanic(recover()) // recover() called directly in deferred func
	}()

	panic("critical error")
}
```

This works because `recover()` is invoked directly within the deferred function body, and its return value is passed to `handlePanic`.
