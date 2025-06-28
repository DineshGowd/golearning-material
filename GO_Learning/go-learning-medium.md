# Go Learning - Medium Level

## 1. Maps (Dictionaries/Hash Tables)

```go
// Map declaration
var m map[string]int

// Map initialization
m = make(map[string]int)
m["apple"] = 5
m["banana"] = 3

// Map literal
scores := map[string]int{
    "Alice": 95,
    "Bob":   87,
    "Carol": 92,
}

// Accessing values
fmt.Println(scores["Alice"]) // 95

// Check if key exists
value, exists := scores["David"]
if exists {
    fmt.Println("David's score:", value)
} else {
    fmt.Println("David not found")
}

// Delete from map
delete(scores, "Bob")

// Iterate over map
for key, value := range scores {
    fmt.Printf("%s: %d\n", key, value)
}

// Map of slices
groups := map[string][]string{
    "fruits":     {"apple", "banana", "orange"},
    "vegetables": {"carrot", "lettuce", "tomato"},
}

// Nested maps
users := map[string]map[string]string{
    "user1": {
        "name":  "John",
        "email": "john@example.com",
    },
    "user2": {
        "name":  "Jane",
        "email": "jane@example.com",
    },
}
```

## 2. Structs

```go
// Struct definition
type Person struct {
    Name    string
    Age     int
    Email   string
    Active  bool
}

// Creating struct instances
// Method 1: Using field names
p1 := Person{
    Name:   "John Doe",
    Age:    30,
    Email:  "john@example.com",
    Active: true,
}

// Method 2: Positional (not recommended)
p2 := Person{"Jane Doe", 25, "jane@example.com", true}

// Method 3: Zero value
var p3 Person // All fields get zero values

// Accessing fields
fmt.Println(p1.Name)
p1.Age = 31

// Struct embedding (composition)
type Address struct {
    Street  string
    City    string
    Country string
}

type Employee struct {
    Person  // Embedded struct
    Address // Embedded struct
    ID      int
    Salary  float64
}

emp := Employee{
    Person: Person{
        Name: "Bob Smith",
        Age:  35,
    },
    Address: Address{
        City:    "New York",
        Country: "USA",
    },
    ID:     12345,
    Salary: 75000,
}

// Access embedded fields directly
fmt.Println(emp.Name)    // From Person
fmt.Println(emp.City)    // From Address
fmt.Println(emp.ID)      // From Employee

// Anonymous structs
point := struct {
    X, Y int
}{10, 20}
```

## 3. Pointers

```go
// Pointer basics
var x int = 42
var p *int = &x  // p points to x

fmt.Println(x)   // 42
fmt.Println(&x)  // Address of x
fmt.Println(p)   // Address of x
fmt.Println(*p)  // 42 (dereferencing)

// Modifying through pointer
*p = 100
fmt.Println(x)   // 100

// Pointer to struct
person := &Person{Name: "Alice", Age: 30}
person.Age = 31  // Go automatically dereferences

// Function with pointer parameter
func increment(x *int) {
    *x++
}

num := 10
increment(&num)
fmt.Println(num) // 11

// New function
ptr := new(int)  // *int pointing to 0
*ptr = 42

// Comparing pointers
var a, b int = 10, 10
ptr1, ptr2 := &a, &b
fmt.Println(ptr1 == ptr2) // false (different addresses)
fmt.Println(*ptr1 == *ptr2) // true (same values)

// Nil pointers
var nilPtr *int
if nilPtr == nil {
    fmt.Println("Pointer is nil")
}
```

## 4. Methods

```go
// Methods on structs
type Rectangle struct {
    Width  float64
    Height float64
}

// Value receiver
func (r Rectangle) Area() float64 {
    return r.Width * r.Height
}

// Pointer receiver (can modify)
func (r *Rectangle) Scale(factor float64) {
    r.Width *= factor
    r.Height *= factor
}

// Method with same name for different types
type Circle struct {
    Radius float64
}

func (c Circle) Area() float64 {
    return 3.14159 * c.Radius * c.Radius
}

// Using methods
rect := Rectangle{Width: 10, Height: 5}
fmt.Println(rect.Area()) // 50

rect.Scale(2)
fmt.Println(rect.Area()) // 200

// Methods on non-struct types
type MyInt int

func (m MyInt) IsPositive() bool {
    return m > 0
}

num := MyInt(42)
fmt.Println(num.IsPositive()) // true
```

## 5. Interfaces

```go
// Interface definition
type Shape interface {
    Area() float64
    Perimeter() float64
}

// Implementing interface
type Rectangle struct {
    Width, Height float64
}

func (r Rectangle) Area() float64 {
    return r.Width * r.Height
}

func (r Rectangle) Perimeter() float64 {
    return 2 * (r.Width + r.Height)
}

type Circle struct {
    Radius float64
}

func (c Circle) Area() float64 {
    return 3.14159 * c.Radius * c.Radius
}

func (c Circle) Perimeter() float64 {
    return 2 * 3.14159 * c.Radius
}

// Using interfaces
func PrintShapeInfo(s Shape) {
    fmt.Printf("Area: %.2f\n", s.Area())
    fmt.Printf("Perimeter: %.2f\n", s.Perimeter())
}

rect := Rectangle{Width: 10, Height: 5}
circle := Circle{Radius: 7}

PrintShapeInfo(rect)
PrintShapeInfo(circle)

// Empty interface
var any interface{} // Can hold any type
any = 42
any = "hello"
any = true

// Type assertion
var i interface{} = "hello"
s, ok := i.(string)
if ok {
    fmt.Println(s) // hello
}

// Type switch
func describe(i interface{}) {
    switch v := i.(type) {
    case int:
        fmt.Printf("Integer: %d\n", v)
    case string:
        fmt.Printf("String: %s\n", v)
    case bool:
        fmt.Printf("Boolean: %t\n", v)
    default:
        fmt.Printf("Unknown type: %T\n", v)
    }
}
```

## 6. Error Handling

```go
import (
    "errors"
    "fmt"
)

// Function returning error
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("division by zero")
    }
    return a / b, nil
}

// Using the function
result, err := divide(10, 0)
if err != nil {
    fmt.Println("Error:", err)
} else {
    fmt.Println("Result:", result)
}

// Custom error types
type ValidationError struct {
    Field string
    Value string
}

func (e ValidationError) Error() string {
    return fmt.Sprintf("validation failed for field %s: invalid value %s", 
                      e.Field, e.Value)
}

// Using custom errors
func validateAge(age int) error {
    if age < 0 {
        return ValidationError{
            Field: "age",
            Value: fmt.Sprintf("%d", age),
        }
    }
    return nil
}

// fmt.Errorf for formatted errors
func processFile(filename string) error {
    if filename == "" {
        return fmt.Errorf("invalid filename: %s", filename)
    }
    return nil
}

// Error wrapping (Go 1.13+)
func doSomething() error {
    err := doStep1()
    if err != nil {
        return fmt.Errorf("step 1 failed: %w", err)
    }
    return nil
}
```

## 7. Defer, Panic, and Recover

```go
// Defer - executes after function returns
func readFile() {
    file := openFile("data.txt")
    defer file.Close() // Will run after function returns
    
    // Process file
    // Even if error occurs, file.Close() will be called
}

// Multiple defers - LIFO order
func multiple() {
    defer fmt.Println("First")
    defer fmt.Println("Second")
    defer fmt.Println("Third")
    // Output: Third, Second, First
}

// Defer with anonymous function
func deferExample() {
    name := "initial"
    defer func() {
        fmt.Println(name) // Captures current value
    }()
    name = "changed"
    // Output: changed
}

// Panic and recover
func riskyOperation() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("Recovered from:", r)
        }
    }()
    
    panic("something went wrong")
    // Code after panic won't execute
}

// Practical defer usage
func writeData(data []byte) error {
    file, err := os.Create("output.txt")
    if err != nil {
        return err
    }
    defer file.Close()
    
    _, err = file.Write(data)
    return err // file.Close() will be called
}
```

## 8. Goroutines (Basic Concurrency)

```go
import (
    "fmt"
    "time"
)

// Basic goroutine
func printNumbers() {
    for i := 1; i <= 5; i++ {
        fmt.Println(i)
        time.Sleep(100 * time.Millisecond)
    }
}

func main() {
    go printNumbers() // Run concurrently
    
    // Main continues
    fmt.Println("Main function")
    time.Sleep(1 * time.Second) // Wait for goroutine
}

// Anonymous goroutine
go func() {
    fmt.Println("Anonymous goroutine")
}()

// Goroutine with parameters
go func(msg string) {
    fmt.Println(msg)
}("Hello from goroutine")

// Multiple goroutines
for i := 0; i < 5; i++ {
    go func(id int) {
        fmt.Printf("Goroutine %d\n", id)
    }(i) // Pass i as parameter
}
```

## 9. Channels (Basic)

```go
// Channel creation
ch := make(chan int)

// Buffered channel
buffered := make(chan string, 3)

// Sending and receiving
go func() {
    ch <- 42 // Send
}()
value := <-ch // Receive

// Channel as function parameter
func worker(ch chan int) {
    for i := 0; i < 5; i++ {
        ch <- i
    }
    close(ch) // Close channel
}

func main() {
    ch := make(chan int)
    go worker(ch)
    
    // Range over channel
    for val := range ch {
        fmt.Println(val)
    }
}

// Select statement
ch1 := make(chan string)
ch2 := make(chan string)

go func() {
    time.Sleep(1 * time.Second)
    ch1 <- "one"
}()

go func() {
    time.Sleep(2 * time.Second)
    ch2 <- "two"
}()

for i := 0; i < 2; i++ {
    select {
    case msg1 := <-ch1:
        fmt.Println("Received:", msg1)
    case msg2 := <-ch2:
        fmt.Println("Received:", msg2)
    }
}
```

## 10. Packages and Modules

```go
// Package declaration
package mypackage

// Exported (capital letter)
func PublicFunction() {
    fmt.Println("This is public")
}

// Unexported (lowercase)
func privateFunction() {
    fmt.Println("This is private")
}

// Exported struct
type User struct {
    Name  string    // Exported field
    email string    // Unexported field
}

// Import packages
import (
    "fmt"
    "strings"
    "math/rand"
    
    // Alias
    str "strings"
    
    // Blank import (side effects only)
    _ "image/png"
)

// Module initialization
// go.mod file
module github.com/username/projectname

go 1.21

require (
    github.com/gorilla/mux v1.8.0
)

// Init function (runs automatically)
func init() {
    fmt.Println("Package initialized")
}
```

## 11. Type Assertions and Type Switches

```go
// Basic type assertion
var i interface{} = "hello"

// Safe assertion
s, ok := i.(string)
if ok {
    fmt.Printf("String value: %s\n", s)
}

// Type switch
func processValue(i interface{}) {
    switch v := i.(type) {
    case int:
        fmt.Printf("Integer: %d\n", v*2)
    case string:
        fmt.Printf("String length: %d\n", len(v))
    case []int:
        fmt.Printf("Slice length: %d\n", len(v))
    case nil:
        fmt.Println("Value is nil")
    default:
        fmt.Printf("Unknown type: %T\n", v)
    }
}

// Interface type assertion
type Writer interface {
    Write([]byte) (int, error)
}

func checkWriter(w interface{}) {
    if writer, ok := w.(Writer); ok {
        writer.Write([]byte("test"))
    }
}
```

## 12. JSON Handling

```go
import (
    "encoding/json"
    "fmt"
)

// Struct for JSON
type Person struct {
    Name    string `json:"name"`
    Age     int    `json:"age"`
    Email   string `json:"email,omitempty"`
    Active  bool   `json:"is_active"`
}

// Marshal (struct to JSON)
person := Person{
    Name:   "John Doe",
    Age:    30,
    Active: true,
}

jsonData, err := json.Marshal(person)
if err != nil {
    fmt.Println("Error:", err)
}
fmt.Println(string(jsonData))

// Pretty print
jsonDataPretty, _ := json.MarshalIndent(person, "", "  ")
fmt.Println(string(jsonDataPretty))

// Unmarshal (JSON to struct)
jsonStr := `{"name":"Jane Doe","age":25,"is_active":false}`
var p Person
err = json.Unmarshal([]byte(jsonStr), &p)
if err != nil {
    fmt.Println("Error:", err)
}
fmt.Printf("%+v\n", p)

// Working with unknown JSON structure
var result map[string]interface{}
json.Unmarshal([]byte(jsonStr), &result)

// JSON arrays
jsonArray := `[{"name":"John"},{"name":"Jane"}]`
var people []Person
json.Unmarshal([]byte(jsonArray), &people)
```

## Practice Projects

1. **TODO List Manager**: Use structs, slices, and methods
2. **Bank Account System**: Implement with interfaces and error handling
3. **Concurrent Web Scraper**: Use goroutines and channels
4. **JSON API Client**: Parse and handle JSON responses
5. **Custom Error Types**: Create domain-specific errors
6. **Worker Pool**: Implement using goroutines and channels
7. **Configuration Loader**: Read JSON/YAML config files

## Common Medium-Level Pitfalls

1. **Nil pointer dereference**: Always check pointers before use
2. **Goroutine leaks**: Ensure goroutines can exit
3. **Race conditions**: Use channels or sync package
4. **Not closing channels**: Can cause goroutines to block forever
5. **Interface pollution**: Don't create interfaces until needed
6. **Error handling**: Don't ignore errors with `_`

## Next Steps

Ready for the Hard level? You'll learn:
- Advanced concurrency patterns
- Context package
- Reflection
- Testing strategies
- Performance optimization
- Advanced interface patterns
- Generic types (Go 1.18+)