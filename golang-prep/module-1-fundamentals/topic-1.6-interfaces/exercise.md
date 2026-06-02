# Exercises: Interfaces & Implicit Implementation

Test your understanding of Go's implicit interfaces, `iface` vs. `eface` memory representations, pointer receivers in method sets, type assertions, and mocking design patterns.

---

## 🧠 Conceptual Questions

### Question 1: Decoupling eface vs. iface
1. What is the difference between `eface` and `iface` in the Go runtime?
2. Why does the compiler generate an `itab` for an `iface`, and how is it used at runtime to look up and invoke methods? Does `eface` have an `itab`?

---

### Question 2: The Logic Behind Implicit Implementation
In Java, you must write `class MyClass implements Database`. In Go, you just define methods.
1. Explain how Go's implicit implementation allows you to write interfaces in your own package for third-party structs that you do not own (e.g., standard library types or external dependencies).
2. Why is this useful for implementing the dependency inversion principle?

---

## 🛠️ Practical Problems

### Question 3: Method Set Compiler Error
Explain why the following code fails to compile. What is the compiler error, and how do you modify the code in `main` to resolve it?
```go
package main

import "fmt"

type Setter interface {
	Set(val int)
}

type Number struct {
	value int
}

func (n *Number) Set(val int) {
	n.value = val
}

func main() {
	var s Setter
	num := Number{value: 10}
	s = num // Compilation Error here
	s.Set(20)
	fmt.Println(num.value)
}
```

---

### Question 4: Multi-type Switch Parser
Write a function `ExtractText(val any) string` that inspects the type of `val` using a type switch:
* If `val` is a `string`, return the string.
* If `val` is a `[]byte`, convert it to a string and return it.
* If `val` implements the `error` interface, return its error message (`err.Error()`).
* If `val` implements `fmt.Stringer`, return its string representation (`val.String()`).
* If `val` is `nil`, return the string `"nil"`.
* For any other type, return `"unknown type"`.

Include the import block and mock helper structs if needed to test your implementation.

---

## 💼 Scenario-Based Questions

### Question 5: Interface-Driven Repository Mocking
You are building a User Signup Service. The service depends on a database repository to save users. To write reliable unit tests, you need to mock the database.

1. Define a `UserRepository` interface with two methods:
   * `GetByID(id int) (*User, error)`
   * `Save(u *User) error`
2. Create a `UserService` struct that accepts this interface in its constructor:
   ```go
   type UserService struct {
       repo UserRepository
   }
   ```
3. Implement a `MockUserRepository` struct that allows tests to stub return values and inspect which arguments were passed.
4. Write a simple mock-based test function `TestUserService_Signup` that demonstrates:
   * Injecting the mock repository into `UserService`.
   * Asserting that a signup fails if the repository returns a duplicate email error.
