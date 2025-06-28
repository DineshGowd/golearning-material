# Go Interview - Easy Level

## 1. Basic Go Questions

### Q: What is Go and why was it created?
**A:** Go (Golang) is a statically typed, compiled programming language designed at Google in 2007. It was created to address issues with existing languages:
- **Simplicity**: Easy to learn and read
- **Concurrency**: Built-in support for concurrent programming
- **Performance**: Compiled language with garbage collection
- **Fast compilation**: Quick build times
- **Modern tooling**: Built-in formatting, testing, and documentation

### Q: What are the advantages of Go?
**A:**
- Simple and clean syntax
- Fast compilation
- Built-in concurrency (goroutines)
- Automatic memory management (garbage collection)
- Strong standard library
- Cross-platform compilation
- Static typing with type inference
- No inheritance (composition over inheritance)

### Q: What is GOPATH and GOROOT?
**A:**
- **GOROOT**: Directory where Go is installed (e.g., `/usr/local/go`)
- **GOPATH**: Workspace directory for Go projects (deprecated with Go modules)
- With Go modules (go.mod), GOPATH is less important

### Q: What is a package in Go?
**A:**
```go
package main // Package declaration

import "fmt" // Importing other packages

// Package is a collection of Go source files
// main package is special - program entry point
```

## 2. Variables and Data Types

### Q: How do you declare variables in Go?
**A:**
```go
// Method 1: var keyword
var name string = "John"
var age int = 30

// Method 2: Type inference
var city = "New York"

// Method 3: Short declaration (only in functions)
country := "USA"

// Multiple variables
var x, y int = 10, 20
a, b := 5.5, 3.3
```

### Q: What are zero values in Go?
**A:**
```go
var i int        // 0
var f float64    // 0.0
var b bool       // false
var s string     // ""
var p *int       // nil
var sl []int     // nil
var m map[string]int // nil
var ch chan int  // nil
var fn func()    // nil
var iface interface{} // nil
```

### Q: What's the difference between `var` and `:=`?
**A:**
```go
// var - can be used anywhere, explicit type optional
var name string = "John"
var age = 30

// := short declaration - only inside functions
func main() {
    city := "NYC" // OK
}

// city := "NYC" // Error: outside function
```

### Q: Explain type conversion in Go
**A:**
```go
// Go requires explicit type conversion
var i int = 42
var f float64 = float64(i)  // Explicit conversion
var u uint = uint(f)

// string conversion
var s string = strconv.Itoa(i)  // int to string
var n, _ = strconv.Atoi("123")  // string to int
```

## 3. Control Structures

### Q: How does if-else work in Go?
**A:**
```go
// No parentheses needed
if x > 0 {
    fmt.Println("Positive")
} else if x < 0 {
    fmt.Println("Negative")
} else {
    fmt.Println("Zero")
}

// With initialization
if val := compute(); val > 0 {
    fmt.Println(val)
}
// val not accessible here
```

### Q: Explain switch statement in Go
**A:**
```go
// No break needed - no fallthrough by default
switch day {
case "Monday":
    fmt.Println("Start of week")
case "Friday":
    fmt.Println("TGIF")
case "Saturday", "Sunday":
    fmt.Println("Weekend")
default:
    fmt.Println("Midweek")
}

// Switch without condition
switch {
case score >= 90:
    return "A"
case score >= 80:
    return "B"
default:
    return "F"
}
```

### Q: What types of loops exist in Go?
**A:**
```go
// Only 'for' loop exists in Go

// Traditional for
for i := 0; i < 10; i++ {
    fmt.Println(i)
}

// While-style
i := 0
for i < 10 {
    fmt.Println(i)
    i++
}

// Infinite loop
for {
    // break to exit
}

// Range loop
slice := []int{1, 2, 3}
for index, value := range slice {
    fmt.Println(index, value)
}
```

## 4. Functions

### Q: How do you define functions in Go?
**A:**
```go
// Basic function
func greet() {
    fmt.Println("Hello")
}

// With parameters and return
func add(x, y int) int {
    return x + y
}

// Multiple return values
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("division by zero")
    }
    return a / b, nil
}

// Named return values
func split(sum int) (x, y int) {
    x = sum * 4 / 9
    y = sum - x
    return // naked return
}
```

### Q: What is a variadic function?
**A:**
```go
// Function accepting variable number of arguments
func sum(nums ...int) int {
    total := 0
    for _, num := range nums {
        total += num
    }
    return total
}

// Usage
sum(1, 2, 3)      // 6
sum(1, 2, 3, 4, 5) // 15

// Passing slice to variadic function
numbers := []int{1, 2, 3}
sum(numbers...) // Use ... to expand slice
```

### Q: Can functions be passed as arguments?
**A:**
```go
// Function type
type operation func(int, int) int

// Function accepting function parameter
func calculate(x, y int, op operation) int {
    return op(x, y)
}

// Usage
add := func(a, b int) int { return a + b }
result := calculate(10, 5, add) // 15
```

## 5. Arrays and Slices

### Q: What's the difference between arrays and slices?
**A:**
```go
// Array - fixed size
var arr [5]int // Size is part of type
arr[0] = 10

// Slice - dynamic size
var slice []int
slice = append(slice, 10)

// Key differences:
// 1. Arrays have fixed size, slices are dynamic
// 2. Arrays are values, slices are references
// 3. Array size is part of its type
```

### Q: How do you create slices?
**A:**
```go
// Method 1: Slice literal
slice1 := []int{1, 2, 3}

// Method 2: make function
slice2 := make([]int, 5)       // length 5, capacity 5
slice3 := make([]int, 5, 10)   // length 5, capacity 10

// Method 3: From array
arr := [5]int{1, 2, 3, 4, 5}
slice4 := arr[1:4]  // [2, 3, 4]

// Method 4: Empty slice
var slice5 []int
```

### Q: Explain slice internals
**A:**
```go
// Slice is a struct with three fields:
// - Pointer to underlying array
// - Length (len)
// - Capacity (cap)

slice := make([]int, 3, 5)
fmt.Println(len(slice)) // 3
fmt.Println(cap(slice)) // 5

// Append may create new underlying array
slice = append(slice, 4, 5, 6) // Exceeds capacity
// New array allocated, capacity doubled
```

### Q: Common slice operations?
**A:**
```go
// Append
slice := []int{1, 2, 3}
slice = append(slice, 4)
slice = append(slice, 5, 6, 7)

// Copy
src := []int{1, 2, 3}
dst := make([]int, len(src))
copy(dst, src)

// Slice of slice
sub := slice[1:3]  // [2, 3]
sub = slice[:2]    // [1, 2]
sub = slice[2:]    // [3, 4, 5, 6, 7]
sub = slice[:]     // entire slice
```

## 6. Maps

### Q: How do you work with maps?
**A:**
```go
// Declaration and initialization
var m map[string]int
m = make(map[string]int)

// Map literal
scores := map[string]int{
    "Alice": 90,
    "Bob":   85,
}

// Operations
scores["Charlie"] = 88        // Insert/Update
delete(scores, "Bob")         // Delete
value := scores["Alice"]      // Access

// Check existence
value, exists := scores["David"]
if exists {
    fmt.Println("Found:", value)
}
```

### Q: Are maps thread-safe?
**A:** No, maps are not thread-safe. For concurrent access, you need to use sync.Mutex or sync.RWMutex:
```go
type SafeMap struct {
    mu   sync.RWMutex
    data map[string]int
}

func (m *SafeMap) Get(key string) (int, bool) {
    m.mu.RLock()
    defer m.mu.RUnlock()
    val, ok := m.data[key]
    return val, ok
}
```

## 7. Structs

### Q: How do you define and use structs?
**A:**
```go
// Struct definition
type Person struct {
    Name string
    Age  int
    City string
}

// Creating instances
p1 := Person{Name: "John", Age: 30, City: "NYC"}
p2 := Person{"Jane", 25, "LA"} // Not recommended

var p3 Person // Zero values
p3.Name = "Bob"

// Pointer to struct
p4 := &Person{Name: "Alice", Age: 28}
p4.Age = 29 // Automatic dereferencing
```

### Q: What is struct embedding?
**A:**
```go
type Address struct {
    Street string
    City   string
}

type Person struct {
    Name    string
    Address // Embedded struct
}

p := Person{
    Name: "John",
    Address: Address{
        Street: "123 Main St",
        City:   "NYC",
    },
}

// Access embedded fields directly
fmt.Println(p.City) // NYC
```

## 8. Pointers

### Q: How do pointers work in Go?
**A:**
```go
// & gets address, * dereferences
var x int = 42
var p *int = &x

fmt.Println(x)  // 42
fmt.Println(&x) // 0xc0000140a8 (address)
fmt.Println(p)  // 0xc0000140a8 (address)
fmt.Println(*p) // 42 (value)

// Modify through pointer
*p = 100
fmt.Println(x) // 100
```

### Q: When should you use pointers?
**A:**
1. To modify the original value
2. To avoid copying large structs
3. When implementing methods that modify the receiver
4. For optional/nullable values

```go
// Without pointer - doesn't modify original
func tryModify(p Person) {
    p.Age = 100
}

// With pointer - modifies original
func modify(p *Person) {
    p.Age = 100
}
```

## 9. Basic Error Handling

### Q: How does error handling work in Go?
**A:**
```go
// Functions return error as last value
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("division by zero")
    }
    return a / b, nil
}

// Always check errors
result, err := divide(10, 0)
if err != nil {
    fmt.Println("Error:", err)
    return
}
fmt.Println("Result:", result)
```

### Q: What is the error type?
**A:**
```go
// error is a built-in interface
type error interface {
    Error() string
}

// Creating custom errors
type ValidationError struct {
    Field string
}

func (e ValidationError) Error() string {
    return fmt.Sprintf("validation failed for %s", e.Field)
}
```

## 10. Common Interview Coding Questions

### Q: Reverse a string
```go
func reverseString(s string) string {
    runes := []rune(s)
    for i, j := 0, len(runes)-1; i < j; i, j = i+1, j-1 {
        runes[i], runes[j] = runes[j], runes[i]
    }
    return string(runes)
}
```

### Q: Check if string is palindrome
```go
func isPalindrome(s string) bool {
    s = strings.ToLower(s)
    for i, j := 0, len(s)-1; i < j; i, j = i+1, j-1 {
        if s[i] != s[j] {
            return false
        }
    }
    return true
}
```

### Q: Find duplicates in slice
```go
func findDuplicates(nums []int) []int {
    seen := make(map[int]bool)
    duplicates := []int{}
    
    for _, num := range nums {
        if seen[num] {
            duplicates = append(duplicates, num)
        }
        seen[num] = true
    }
    return duplicates
}
```

### Q: FizzBuzz
```go
func fizzBuzz(n int) {
    for i := 1; i <= n; i++ {
        switch {
        case i%15 == 0:
            fmt.Println("FizzBuzz")
        case i%3 == 0:
            fmt.Println("Fizz")
        case i%5 == 0:
            fmt.Println("Buzz")
        default:
            fmt.Println(i)
        }
    }
}
```

## Common Mistakes to Avoid

1. **Not checking errors**
   ```go
   // Bad
   result, _ := someFunction()
   
   // Good
   result, err := someFunction()
   if err != nil {
       // Handle error
   }
   ```

2. **Range loop variable capture**
   ```go
   // Bad
   for _, v := range values {
       go func() {
           fmt.Println(v) // Captures loop variable
       }()
   }
   
   // Good
   for _, v := range values {
       v := v // Create new variable
       go func() {
           fmt.Println(v)
       }()
   }
   ```

3. **Nil slice vs empty slice**
   ```go
   var s1 []int        // nil slice
   s2 := []int{}       // empty slice
   s3 := make([]int, 0) // empty slice
   
   // All have len() == 0, but s1 is nil
   ```

## Tips for Easy Level Interviews

1. **Know the basics thoroughly** - syntax, data types, control structures
2. **Understand value vs reference types** - arrays vs slices, when to use pointers
3. **Practice common algorithms** - string manipulation, array operations
4. **Always handle errors** - Never ignore error returns
5. **Use proper naming** - Follow Go conventions (camelCase for private, PascalCase for public)
6. **Keep it simple** - Go favors simplicity over cleverness