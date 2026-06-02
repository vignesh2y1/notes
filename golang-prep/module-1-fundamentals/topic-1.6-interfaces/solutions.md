# Solutions: Interfaces & Implicit Implementation

Below are the step-by-step solutions, code answers, and architectural explanations for the Topic 1.6 exercises.

---

## 🧠 Conceptual Questions

### Solution 1: Decoupling eface vs. iface

1. **Differences in Representation:**
   * **`eface` (Empty Interface):** Used for variables of type `any` (or `interface{}`). Since it defines zero methods, it only needs to track two things: the pointer to the concrete type information (`_type`) and the pointer to the actual data (`data`).
   * **`iface` (Non-Empty Interface):** Used when the interface defines one or more method signatures. It contains the data pointer (`data`) and a pointer to an interface table (`tab` of type `*itab`).
2. **The `itab` and Runtime Dispatch:**
   * The `itab` acts as a translation layer. It stores the metadata of the interface and concrete type, alongside an array of function pointers (`fun`) pointing to the concrete type's implementations.
   * When you invoke `i.Read()`, Go does not scan method names at runtime. Instead, the compiler maps `Read()` to a fixed index (e.g., index `0` of the `fun` array) in the `itab`. The runtime simply dereferences that index to call the function directly, passing the data pointer as the first argument (receiver).
   * Since `eface` defines no methods, it does not require a method dispatch table. Hence, it has no `itab`, only a direct type descriptor pointer `_type`.

---

### Solution 2: The Logic Behind Implicit Implementation

1. **Consumer-Defined Interfaces:**
   In explicit languages like Java, if a third-party library defines `type Client struct`, and you want to put it behind an interface, you cannot do so without writing wrapper adapter classes because you cannot modify the third-party library source code to append `implements MyInterface`.
   In Go, because implementation is implicit, you can declare:
   ```go
   type Fetcher interface {
       FetchData() (string, error)
   }
   ```
   If the third-party structure has a method `FetchData() (string, error)`, it **automatically** implements your interface. You do not need to modify the library code or write wrapper adapters.
2. **Dependency Inversion Principle:**
   Instead of the producer defining what the consumer can do (explicit interfaces defined by libraries), Go allows the **consumer** to define the exact, small interfaces it needs to do its job. This completely decouples packages and simplifies mocking, allowing package owners to only depend on their local definitions rather than large external dependencies.

---

## 🛠️ Practical Problems

### Solution 3: Method Set Compiler Error

#### The Error:
```
cannot use num (variable of type Number) as Setter value in assignment:
Number does not implement Setter (method Set has pointer receiver)
```

#### Why it fails:
The method `Set` is declared on `*Number` (a pointer receiver):
```go
func (n *Number) Set(val int)
```
In Go, the method set of a **value type** (`Number`) does not include methods declared with pointer receivers. Only the **pointer type** (`*Number`) contains the pointer receiver methods.
Because `num` is a value type, it does not possess the `Set` method and does not implement the `Setter` interface.

#### The Fix:
Assign the address of `num` (which is of type `*Number`) to the interface variable:
```go
func main() {
	var s Setter
	num := Number{value: 10}
	s = &num // Change: Assign pointer instead of value
	s.Set(20)
	fmt.Println(num.value) // Output: 20
}
```

---

### Solution 4: Multi-type Switch Parser

```go
package main

import (
	"fmt"
)

// ExtractText returns string representations of various input types.
func ExtractText(val any) string {
	if val == nil {
		return "nil"
	}

	switch v := val.(type) {
	case string:
		return v
	case []byte:
		return string(v)
	case error:
		return v.Error()
	case fmt.Stringer:
		return v.String()
	default:
		return "unknown type"
	}
}

// Mock Stringer struct for testing
type User struct {
	Name string
}

func (u User) String() string {
	return "User: " + u.Name
}

func main() {
	fmt.Println(ExtractText("Hello String"))             // Output: Hello String
	fmt.Println(ExtractText([]byte("Hello Byte Slice"))) // Output: Hello Byte Slice
	fmt.Println(ExtractText(fmt.Errorf("db error")))     // Output: db error
	fmt.Println(ExtractText(User{Name: "Alice"}))        // Output: User: Alice
	fmt.Println(ExtractText(12345))                      // Output: unknown type
	fmt.Println(ExtractText(nil))                        // Output: nil
}
```

---

## 💼 Scenario-Based Questions

### Solution 5: Interface-Driven Repository Mocking

#### 1. Models and Interface:
```go
package main

import "errors"

type User struct {
	ID    int
	Email string
}

type UserRepository interface {
	GetByID(id int) (*User, error)
	Save(u *User) error
}
```

#### 2. Service Implementation:
```go
type UserService struct {
	repo UserRepository
}

func NewUserService(r UserRepository) *UserService {
	return &UserService{repo: r}
}

func (s *UserService) Register(u *User) error {
	if u.Email == "" {
		return errors.New("invalid email")
	}
	// Check if user already exists
	return s.repo.Save(u)
}
```

#### 3. Mock Repository Implementation:
```go
type MockUserRepository struct {
	SaveFunc       func(u *User) error
	GetByIDFunc    func(id int) (*User, error)
	CapturedSaved  []*User
	CapturedGetIDs []int
}

func (m *MockUserRepository) Save(u *User) error {
	m.CapturedSaved = append(m.CapturedSaved, u) // Capture argument
	if m.SaveFunc != nil {
		return m.SaveFunc(u)
	}
	return nil
}

func (m *MockUserRepository) GetByID(id int) (*User, error) {
	m.CapturedGetIDs = append(m.CapturedGetIDs, id) // Capture argument
	if m.GetByIDFunc != nil {
		return m.GetByIDFunc(id)
	}
	return nil, nil
}
```

#### 4. Mock-Based Test Runner:
```go
func TestUserService_Signup() {
	// Setup the mock with custom duplicate error behavior
	mockRepo := &MockUserRepository{
		SaveFunc: func(u *User) error {
			return errors.New("duplicate email error")
		},
	}
	
	service := NewUserService(mockRepo)
	newUser := &User{ID: 1, Email: "duplicate@example.com"}
	
	err := service.Register(newUser)
	
	// Assertions
	if err == nil {
		panic("expected error but got nil")
	}
	
	if err.Error() != "duplicate email error" {
		panic("expected 'duplicate email error' but got: " + err.Error())
	}
	
	if len(mockRepo.CapturedSaved) != 1 {
		panic("expected 1 call to Save()")
	}
	
	if mockRepo.CapturedSaved[0].Email != "duplicate@example.com" {
		panic("Save() was called with wrong email")
	}
	
	println("TestUserService_Signup passed successfully!")
}

func main() {
	TestUserService_Signup()
}
```
Using this interface mocking pattern, the `UserService` can be fully unit tested without requiring any database network connections.
