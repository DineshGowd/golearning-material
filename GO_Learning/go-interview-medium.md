# Go Interview - Medium Level

## 1. Interfaces Deep Dive

### Q: What are interfaces in Go and how do they work?
**A:** Interfaces define a set of method signatures. Types implicitly satisfy interfaces by implementing their methods.

```go
type Writer interface {
    Write([]byte) (int, error)
}

// Any type with Write method satisfies Writer
type FileWriter struct {
    filename string
}

func (fw FileWriter) Write(data []byte) (int, error) {
    // Implementation
    return len(data), nil
}

// Implicit implementation - no "implements" keyword
var w Writer = FileWriter{"test.txt"}
```

### Q: What is an empty interface?
**A:**
```go
// interface{} can hold values of any type
var any interface{}
any = 42
any = "hello"
any = []int{1, 2, 3}

// Type assertion to get concrete value
if str, ok := any.(string); ok {
    fmt.Println("String:", str)
}

// Type switch
switch v := any.(type) {
case int:
    fmt.Println("Integer:", v)
case string:
    fmt.Println("String:", v)
default:
    fmt.Println("Unknown type")
}
```

### Q: Explain interface composition
**A:**
```go
type Reader interface {
    Read([]byte) (int, error)
}

type Writer interface {
    Write([]byte) (int, error)
}

type Closer interface {
    Close() error
}

// Interface embedding
type ReadWriter interface {
    Reader
    Writer
}

type ReadWriteCloser interface {
    Reader
    Writer
    Closer
}

// Or inline
type ReadWriteCloser interface {
    Read([]byte) (int, error)
    Write([]byte) (int, error)
    Close() error
}
```

### Q: Common interface pitfalls?
**A:**
```go
// 1. Nil interface vs nil pointer
var p *Person = nil
var i interface{} = p
fmt.Println(i == nil)  // false! Interface has type info

// 2. Comparing interfaces
type MyInt int
var a interface{} = MyInt(5)
var b interface{} = 5
fmt.Println(a == b)  // false! Different types

// 3. Interface pollution
// Bad: Too many single-method interfaces
type Getter interface { Get() string }
type Setter interface { Set(string) }
type Deleter interface { Delete() }

// Good: Cohesive interface
type Storage interface {
    Get() string
    Set(string)
    Delete()
}
```

## 2. Goroutines and Concurrency

### Q: What are goroutines?
**A:** Goroutines are lightweight threads managed by the Go runtime. They're much cheaper than OS threads.

```go
// Starting a goroutine
go func() {
    fmt.Println("Running concurrently")
}()

// With parameters
go func(msg string) {
    fmt.Println(msg)
}("Hello")

// Common mistake - loop variable capture
for i := 0; i < 5; i++ {
    go func(n int) {
        fmt.Println(n)  // Use parameter, not i
    }(i)
}
```

### Q: How do you synchronize goroutines?
**A:**
```go
// 1. WaitGroup
var wg sync.WaitGroup

for i := 0; i < 5; i++ {
    wg.Add(1)
    go func(id int) {
        defer wg.Done()
        // Do work
        fmt.Printf("Worker %d done\n", id)
    }(i)
}
wg.Wait() // Block until all done

// 2. Channels for synchronization
done := make(chan bool)
go func() {
    // Do work
    done <- true
}()
<-done // Wait for completion

// 3. Mutex for shared data
type Counter struct {
    mu    sync.Mutex
    value int
}

func (c *Counter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.value++
}
```

### Q: Explain goroutine leaks
**A:** Goroutines that never terminate cause memory leaks.

```go
// Leak example - goroutine blocks forever
func leak() {
    ch := make(chan int)
    go func() {
        val := <-ch  // Blocks forever if channel not closed
        fmt.Println(val)
    }()
    // Function returns, channel never written to
}

// Fix: Ensure goroutines can exit
func noLeak() {
    ch := make(chan int)
    done := make(chan bool)
    
    go func() {
        select {
        case val := <-ch:
            fmt.Println(val)
        case <-done:
            return
        }
    }()
    
    close(done) // Signal goroutine to exit
}
```

## 3. Channels

### Q: What are channels and how do they work?
**A:** Channels are typed conduits for communication between goroutines.

```go
// Unbuffered channel
ch := make(chan int)

// Buffered channel
buffered := make(chan string, 3)

// Send and receive
ch <- 42     // Send (blocks if no receiver)
value := <-ch // Receive (blocks if no sender)

// Close channel
close(ch)

// Check if channel is closed
val, ok := <-ch
if !ok {
    fmt.Println("Channel closed")
}

// Range over channel
for val := range ch {
    fmt.Println(val)
}
```

### Q: Buffered vs Unbuffered channels?
**A:**
```go
// Unbuffered - synchronous
unbuffered := make(chan int)
// Sender blocks until receiver ready

// Buffered - asynchronous up to buffer size
buffered := make(chan int, 3)
buffered <- 1  // Doesn't block
buffered <- 2  // Doesn't block
buffered <- 3  // Doesn't block
buffered <- 4  // Blocks! Buffer full

// Use cases:
// - Unbuffered: Synchronization
// - Buffered: Performance, decoupling producers/consumers
```

### Q: What is channel direction?
**A:**
```go
// Bidirectional channel
var ch chan int

// Send-only channel
var sendOnly chan<- int

// Receive-only channel
var recvOnly <-chan int

// Function with directional channels
func send(ch chan<- int) {
    ch <- 42
    // <-ch  // Compile error!
}

func receive(ch <-chan int) {
    val := <-ch
    // ch <- 42  // Compile error!
}
```

### Q: Explain select statement
**A:**
```go
// Select enables non-blocking channel operations
select {
case val := <-ch1:
    fmt.Println("Received from ch1:", val)
case val := <-ch2:
    fmt.Println("Received from ch2:", val)
case ch3 <- 42:
    fmt.Println("Sent to ch3")
default:
    fmt.Println("No channels ready")
}

// Timeout pattern
select {
case result := <-ch:
    fmt.Println("Got result:", result)
case <-time.After(1 * time.Second):
    fmt.Println("Timeout!")
}

// Non-blocking send/receive
select {
case ch <- value:
    // Sent successfully
default:
    // Channel full/not ready
}
```

## 4. Methods and Receivers

### Q: Value receiver vs Pointer receiver?
**A:**
```go
type Person struct {
    Name string
    Age  int
}

// Value receiver - gets copy
func (p Person) String() string {
    return fmt.Sprintf("%s (%d)", p.Name, p.Age)
}

// Pointer receiver - can modify
func (p *Person) Birthday() {
    p.Age++
}

// When to use which?
// - Pointer receiver: Need to modify, large struct, consistency
// - Value receiver: Immutable, small struct, concurrency-safe

// Consistency rule: If any method has pointer receiver,
// all should have pointer receiver
```

### Q: Can you add methods to built-in types?
**A:**
```go
// Cannot add methods to built-in types directly
// func (s string) Reverse() string { } // Error!

// Solution: Create new type
type MyString string

func (s MyString) Reverse() string {
    runes := []rune(s)
    for i, j := 0, len(runes)-1; i < j; i, j = i+1, j-1 {
        runes[i], runes[j] = runes[j], runes[i]
    }
    return string(runes)
}

// Usage
s := MyString("hello")
fmt.Println(s.Reverse()) // olleh
```

## 5. Error Handling Patterns

### Q: How do you create custom errors?
**A:**
```go
// 1. Simple error
err := errors.New("something went wrong")

// 2. Formatted error
err := fmt.Errorf("failed to process %s: %w", filename, originalErr)

// 3. Custom error type
type ValidationError struct {
    Field string
    Value interface{}
}

func (e ValidationError) Error() string {
    return fmt.Sprintf("validation failed for %s: %v", e.Field, e.Value)
}

// 4. Error with behavior
type TemporaryError interface {
    error
    Temporary() bool
}

// Check error type
if err, ok := err.(TemporaryError); ok && err.Temporary() {
    // Retry operation
}
```

### Q: What is error wrapping?
**A:**
```go
// Wrap errors with context
func processFile(name string) error {
    file, err := os.Open(name)
    if err != nil {
        return fmt.Errorf("processFile: %w", err)
    }
    defer file.Close()
    
    // Process...
    return nil
}

// Unwrap errors
err := processFile("data.txt")
if errors.Is(err, os.ErrNotExist) {
    fmt.Println("File doesn't exist")
}

// Extract specific error type
var pathErr *os.PathError
if errors.As(err, &pathErr) {
    fmt.Println("Path error:", pathErr.Path)
}
```

## 6. Common Concurrency Patterns

### Q: Implement a worker pool
**A:**
```go
func workerPool(jobs <-chan int, results chan<- int, workers int) {
    var wg sync.WaitGroup
    
    for i := 0; i < workers; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for job := range jobs {
                results <- job * job // Process job
            }
        }(i)
    }
    
    wg.Wait()
    close(results)
}

// Usage
jobs := make(chan int, 100)
results := make(chan int, 100)

// Start workers
go workerPool(jobs, results, 5)

// Send jobs
go func() {
    for i := 1; i <= 50; i++ {
        jobs <- i
    }
    close(jobs)
}()

// Collect results
for result := range results {
    fmt.Println(result)
}
```

### Q: Implement rate limiting
**A:**
```go
// Using time.Ticker
func rateLimiter(requests <-chan int) {
    ticker := time.NewTicker(100 * time.Millisecond)
    defer ticker.Stop()
    
    for req := range requests {
        <-ticker.C // Wait for tick
        fmt.Printf("Processing request %d\n", req)
    }
}

// Token bucket algorithm
type RateLimiter struct {
    tokens   chan struct{}
    interval time.Duration
}

func NewRateLimiter(rate int, interval time.Duration) *RateLimiter {
    rl := &RateLimiter{
        tokens:   make(chan struct{}, rate),
        interval: interval,
    }
    
    // Fill bucket
    for i := 0; i < rate; i++ {
        rl.tokens <- struct{}{}
    }
    
    // Refill tokens
    go func() {
        ticker := time.NewTicker(interval / time.Duration(rate))
        for range ticker.C {
            select {
            case rl.tokens <- struct{}{}:
            default: // Bucket full
            }
        }
    }()
    
    return rl
}

func (rl *RateLimiter) Allow() bool {
    select {
    case <-rl.tokens:
        return true
    default:
        return false
    }
}
```

## 7. Package and Module Questions

### Q: Explain Go modules
**A:**
```go
// go.mod file
module github.com/user/project

go 1.21

require (
    github.com/gorilla/mux v1.8.0
    github.com/stretchr/testify v1.7.0
)

// Commands
go mod init github.com/user/project  // Initialize module
go get github.com/pkg/errors         // Add dependency
go mod tidy                          // Clean up
go mod vendor                        // Vendor dependencies
```

### Q: How does package initialization work?
**A:**
```go
package mypackage

import "fmt"

// Package-level variables initialized first
var config = loadConfig()

// init functions run after variables
func init() {
    fmt.Println("Package initializing...")
    // Setup code
}

// Multiple init functions allowed
func init() {
    // Run in declaration order
}

// Initialization order:
// 1. Imported packages
// 2. Package-level variables
// 3. init functions
// 4. main function (if applicable)
```

## 8. Testing

### Q: How do you write tests in Go?
**A:**
```go
// math_test.go
package math

import "testing"

func TestAdd(t *testing.T) {
    result := Add(2, 3)
    expected := 5
    
    if result != expected {
        t.Errorf("Add(2, 3) = %d; want %d", result, expected)
    }
}

// Table-driven tests
func TestCalculate(t *testing.T) {
    tests := []struct {
        name     string
        input    int
        expected int
    }{
        {"positive", 5, 25},
        {"negative", -5, 25},
        {"zero", 0, 0},
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result := Calculate(tt.input)
            if result != tt.expected {
                t.Errorf("Calculate(%d) = %d; want %d", 
                    tt.input, result, tt.expected)
            }
        })
    }
}
```

### Q: How do you write benchmarks?
**A:**
```go
func BenchmarkAdd(b *testing.B) {
    for i := 0; i < b.N; i++ {
        Add(2, 3)
    }
}

// Benchmark with different inputs
func BenchmarkSort(b *testing.B) {
    sizes := []int{10, 100, 1000}
    
    for _, size := range sizes {
        b.Run(fmt.Sprintf("size-%d", size), func(b *testing.B) {
            data := generateData(size)
            b.ResetTimer()
            
            for i := 0; i < b.N; i++ {
                Sort(data)
            }
        })
    }
}
```

## 9. Common Interview Coding Problems

### Q: Implement a concurrent safe map
```go
type SafeMap struct {
    mu   sync.RWMutex
    data map[string]interface{}
}

func NewSafeMap() *SafeMap {
    return &SafeMap{
        data: make(map[string]interface{}),
    }
}

func (m *SafeMap) Set(key string, value interface{}) {
    m.mu.Lock()
    defer m.mu.Unlock()
    m.data[key] = value
}

func (m *SafeMap) Get(key string) (interface{}, bool) {
    m.mu.RLock()
    defer m.mu.RUnlock()
    val, ok := m.data[key]
    return val, ok
}

func (m *SafeMap) Delete(key string) {
    m.mu.Lock()
    defer m.mu.Unlock()
    delete(m.data, key)
}
```

### Q: Implement a producer-consumer pattern
```go
func producer(ch chan<- int, done <-chan bool) {
    i := 0
    for {
        select {
        case ch <- i:
            i++
            time.Sleep(100 * time.Millisecond)
        case <-done:
            close(ch)
            return
        }
    }
}

func consumer(id int, ch <-chan int, wg *sync.WaitGroup) {
    defer wg.Done()
    for val := range ch {
        fmt.Printf("Consumer %d: %d\n", id, val)
        time.Sleep(200 * time.Millisecond)
    }
}

func main() {
    ch := make(chan int, 5)
    done := make(chan bool)
    var wg sync.WaitGroup
    
    // Start producer
    go producer(ch, done)
    
    // Start consumers
    for i := 0; i < 3; i++ {
        wg.Add(1)
        go consumer(i, ch, &wg)
    }
    
    // Run for 5 seconds
    time.Sleep(5 * time.Second)
    done <- true
    
    wg.Wait()
}
```

### Q: Implement an LRU cache
```go
type LRUCache struct {
    capacity int
    cache    map[string]*list.Element
    lru      *list.List
    mu       sync.Mutex
}

type entry struct {
    key   string
    value interface{}
}

func NewLRUCache(capacity int) *LRUCache {
    return &LRUCache{
        capacity: capacity,
        cache:    make(map[string]*list.Element),
        lru:      list.New(),
    }
}

func (c *LRUCache) Get(key string) (interface{}, bool) {
    c.mu.Lock()
    defer c.mu.Unlock()
    
    if elem, ok := c.cache[key]; ok {
        c.lru.MoveToFront(elem)
        return elem.Value.(*entry).value, true
    }
    return nil, false
}

func (c *LRUCache) Put(key string, value interface{}) {
    c.mu.Lock()
    defer c.mu.Unlock()
    
    if elem, ok := c.cache[key]; ok {
        c.lru.MoveToFront(elem)
        elem.Value.(*entry).value = value
        return
    }
    
    if c.lru.Len() >= c.capacity {
        oldest := c.lru.Back()
        if oldest != nil {
            c.lru.Remove(oldest)
            delete(c.cache, oldest.Value.(*entry).key)
        }
    }
    
    elem := c.lru.PushFront(&entry{key, value})
    c.cache[key] = elem
}
```

## Best Practices for Medium Level

1. **Understand interface design** - Keep interfaces small and focused
2. **Know concurrency primitives** - When to use channels vs mutexes
3. **Handle errors properly** - Don't just return errors, add context
4. **Write testable code** - Use interfaces for dependencies
5. **Avoid race conditions** - Use `go race` detector
6. **Prevent goroutine leaks** - Always ensure goroutines can exit
7. **Choose right receiver type** - Consistency is key
8. **Use channels for communication** - "Don't communicate by sharing memory; share memory by communicating"