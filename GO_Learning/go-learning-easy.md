# Go Learning - Easy Level

## 1. Hello World & Basic Structure

```go
package main

import "fmt"

func main() {
    fmt.Println("Hello, World!")
}
```

**Key Points:**
- Every Go file starts with `package` declaration
- `main` package is special - it's the entry point
- `import` brings in other packages
- `func main()` is where execution begins

## 2. Variables & Constants

```go
// Variable declaration methods
var name string = "John"
var age int = 30
var isActive bool = true

// Type inference
var city = "New York"  // Go infers string

// Short declaration (only inside functions)
country := "USA"
score := 100

// Multiple declaration
var x, y int = 10, 20
a, b := 5.5, 3.3

// Constants
const Pi = 3.14159
const (
    StatusOK = 200
    StatusNotFound = 404
)

// Zero values
var i int     // 0
var f float64 // 0.0
var b bool    // false
var s string  // ""
```

## 3. Basic Data Types

```go
// Numeric types
var i int = 42
var i8 int8 = 127
var i16 int16 = 32767
var i32 int32 = 2147483647
var i64 int64 = 9223372036854775807

var ui uint = 42
var ui8 uint8 = 255
var ui16 uint16 = 65535
var ui32 uint32 = 4294967295
var ui64 uint64 = 18446744073709551615

var f32 float32 = 3.14
var f64 float64 = 3.141592653589793

// String and boolean
var str string = "Hello Go"
var flag bool = true

// Type conversion
var x int = 10
var y float64 = float64(x)
var z int = int(y)
```

## 4. Basic Operators

```go
// Arithmetic
a := 10 + 5   // 15
b := 10 - 5   // 5
c := 10 * 5   // 50
d := 10 / 5   // 2
e := 10 % 3   // 1

// Increment/Decrement
i := 1
i++  // i = 2
i--  // i = 1

// Comparison
x := 5
y := 10
fmt.Println(x == y)  // false
fmt.Println(x != y)  // true
fmt.Println(x < y)   // true
fmt.Println(x > y)   // false
fmt.Println(x <= y)  // true
fmt.Println(x >= y)  // false

// Logical
a := true
b := false
fmt.Println(a && b)  // false
fmt.Println(a || b)  // true
fmt.Println(!a)      // false
```

## 5. Control Flow - If/Else

```go
// Basic if
if x > 0 {
    fmt.Println("Positive")
}

// If-else
if age >= 18 {
    fmt.Println("Adult")
} else {
    fmt.Println("Minor")
}

// If-else if-else
if score >= 90 {
    fmt.Println("A")
} else if score >= 80 {
    fmt.Println("B")
} else if score >= 70 {
    fmt.Println("C")
} else {
    fmt.Println("F")
}

// If with initialization
if val := compute(); val > 0 {
    fmt.Println("Positive:", val)
}
// val is not accessible here
```

## 6. Loops

```go
// Basic for loop
for i := 0; i < 5; i++ {
    fmt.Println(i)
}

// While-style loop
i := 0
for i < 5 {
    fmt.Println(i)
    i++
}

// Infinite loop
for {
    fmt.Println("Forever")
    break // Use break to exit
}

// Continue and break
for i := 0; i < 10; i++ {
    if i%2 == 0 {
        continue // Skip even numbers
    }
    if i > 7 {
        break // Exit loop
    }
    fmt.Println(i)
}

// Range over string
for i, ch := range "Hello" {
    fmt.Printf("%d: %c\n", i, ch)
}
```

## 7. Functions

```go
// Basic function
func greet() {
    fmt.Println("Hello!")
}

// Function with parameters
func add(x int, y int) int {
    return x + y
}

// Multiple parameters of same type
func multiply(x, y int) int {
    return x * y
}

// Multiple return values
func divmod(a, b int) (int, int) {
    return a / b, a % b
}

// Named return values
func split(sum int) (x, y int) {
    x = sum * 4 / 9
    y = sum - x
    return // naked return
}

// Using functions
func main() {
    greet()
    
    result := add(3, 5)
    fmt.Println(result) // 8
    
    q, r := divmod(10, 3)
    fmt.Println(q, r) // 3 1
    
    // Ignore a return value
    _, remainder := divmod(10, 3)
    fmt.Println(remainder) // 1
}
```

## 8. Arrays

```go
// Array declaration
var arr [5]int // [0 0 0 0 0]

// Array initialization
arr2 := [5]int{1, 2, 3, 4, 5}

// Partial initialization
arr3 := [5]int{1, 2} // [1 2 0 0 0]

// Array with inferred length
arr4 := [...]int{1, 2, 3} // [1 2 3]

// Accessing elements
fmt.Println(arr2[0]) // 1
arr2[0] = 10

// Array length
fmt.Println(len(arr2)) // 5

// Iterating over array
for i := 0; i < len(arr2); i++ {
    fmt.Println(arr2[i])
}

// Using range
for index, value := range arr2 {
    fmt.Printf("%d: %d\n", index, value)
}
```

## 9. Slices (Dynamic Arrays)

```go
// Slice declaration
var slice []int

// Slice from array
arr := [5]int{1, 2, 3, 4, 5}
slice = arr[1:4] // [2 3 4]

// Make slice
slice2 := make([]int, 5) // [0 0 0 0 0]
slice3 := make([]int, 5, 10) // length 5, capacity 10

// Slice literal
slice4 := []int{1, 2, 3, 4, 5}

// Append to slice
slice4 = append(slice4, 6)
slice4 = append(slice4, 7, 8, 9)

// Slice operations
fmt.Println(len(slice4)) // length
fmt.Println(cap(slice4)) // capacity

// Slice of slice
subSlice := slice4[2:5]

// Copy slice
dest := make([]int, len(slice4))
copy(dest, slice4)
```

## 10. Basic String Operations

```go
// String declaration
str := "Hello, Go!"

// String length
fmt.Println(len(str)) // 10 (bytes)

// String concatenation
str1 := "Hello"
str2 := "World"
result := str1 + " " + str2

// String comparison
fmt.Println("apple" == "apple") // true
fmt.Println("apple" < "banana") // true

// Accessing characters (bytes)
fmt.Println(str[0]) // 72 (ASCII for 'H')

// String to byte slice
bytes := []byte(str)

// Byte slice to string
newStr := string(bytes)

// Multiline strings
multiline := `This is
a multiline
string`
```

## 11. Basic fmt Package

```go
import "fmt"

// Print functions
fmt.Print("Hello")           // No newline
fmt.Println("Hello")         // With newline
fmt.Printf("Hello %s", name) // Formatted

// Common format verbs
name := "John"
age := 30
pi := 3.14159

fmt.Printf("String: %s\n", name)
fmt.Printf("Integer: %d\n", age)
fmt.Printf("Float: %f\n", pi)
fmt.Printf("Float (2 decimal): %.2f\n", pi)
fmt.Printf("Boolean: %t\n", true)
fmt.Printf("Any value: %v\n", age)
fmt.Printf("Type: %T\n", pi)

// Sprint functions (return string)
str := fmt.Sprintf("Name: %s, Age: %d", name, age)
```

## 12. Switch Statement

```go
// Basic switch
switch day {
case "Monday":
    fmt.Println("Start of work week")
case "Friday":
    fmt.Println("TGIF!")
case "Saturday", "Sunday":
    fmt.Println("Weekend!")
default:
    fmt.Println("Midweek")
}

// Switch with no condition
score := 85
switch {
case score >= 90:
    fmt.Println("A")
case score >= 80:
    fmt.Println("B")
case score >= 70:
    fmt.Println("C")
default:
    fmt.Println("F")
}

// Switch with initialization
switch num := getNumber(); {
case num < 0:
    fmt.Println("Negative")
case num == 0:
    fmt.Println("Zero")
default:
    fmt.Println("Positive")
}
```

## Practice Exercises

1. **Variables**: Create variables of different types and print them
2. **Calculator**: Build a simple calculator with add, subtract, multiply, divide
3. **Grade System**: Use if-else to assign letter grades based on scores
4. **Array Sum**: Calculate the sum of all elements in an array
5. **Slice Operations**: Create a slice, append elements, and find min/max
6. **String Manipulation**: Count vowels in a string
7. **Function Practice**: Write functions for factorial and fibonacci
8. **Loop Patterns**: Print various patterns using nested loops

## Common Beginner Mistakes to Avoid

1. **Unused variables**: Go doesn't allow unused variables
2. **`:=` outside functions**: Short declaration only works inside functions
3. **Array vs Slice confusion**: Arrays have fixed size, slices are dynamic
4. **Not handling errors**: Always check returned errors (we'll cover this in medium level)
5. **Forgetting to use the result of `append`**: `slice = append(slice, item)`

## Next Steps

Once comfortable with these concepts, move to the Medium level covering:
- Maps and Structs
- Pointers
- Methods
- Interfaces
- Error handling
- Goroutines basics
- Packages and modules