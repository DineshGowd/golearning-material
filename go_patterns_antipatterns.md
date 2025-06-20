# Go Language Patterns and Anti-Patterns

## Table of Contents
- [Go Patterns](#go-patterns)
- [Go Anti-Patterns](#go-anti-patterns)
- [Best Practices Summary](#best-practices-summary)

---

## Go Patterns

### 1. Interface Segregation Pattern

**Description:** Keep interfaces small and focused on specific behaviors. Go's implicit interface satisfaction makes this particularly powerful.

```go
// Good: Small, focused interfaces
type Reader interface {
    Read([]byte) (int, error)
}

type Writer interface {
    Write([]byte) (int, error)
}

type Closer interface {
    Close() error
}

// Compose when needed
type ReadWriteCloser interface {
    Reader
    Writer
    Closer
}
```

**Benefits:**
- Easier testing and mocking
- Better decoupling
- More flexible code composition

### 2. Error Wrapping Pattern

**Description:** Provide context to errors while preserving the original error information using `fmt.Errorf` with `%w` verb.

```go
package main

import (
    "fmt"
    "errors"
)

func processFile(filename string) error {
    err := openFile(filename)
    if err != nil {
        return fmt.Errorf("failed to process file %s: %w", filename, err)
    }
    return nil
}

func openFile(filename string) error {
    return errors.New("file not found")
}

func main() {
    err := processFile("config.json")
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        
        // Unwrap to check original error
        var originalErr error
        if errors.As(err, &originalErr) {
            fmt.Printf("Original error: %v\n", originalErr)
        }
    }
}
```

### 3. Functional Options Pattern

**Description:** Provide a clean API for optional parameters using variadic functions.

```go
type Server struct {
    host    string
    port    int
    timeout time.Duration
    logger  *log.Logger
}

type Option func(*Server)

func WithHost(host string) Option {
    return func(s *Server) {
        s.host = host
    }
}

func WithPort(port int) Option {
    return func(s *Server) {
        s.port = port
    }
}

func WithTimeout(timeout time.Duration) Option {
    return func(s *Server) {
        s.timeout = timeout
    }
}

func NewServer(opts ...Option) *Server {
    // Default values
    s := &Server{
        host:    "localhost",
        port:    8080,
        timeout: 30 * time.Second,
        logger:  log.New(os.Stdout, "", log.LstdFlags),
    }
    
    // Apply options
    for _, opt := range opts {
        opt(s)
    }
    
    return s
}

// Usage
func main() {
    server := NewServer(
        WithHost("0.0.0.0"),
        WithPort(9090),
        WithTimeout(60*time.Second),
    )
}
```

### 4. Worker Pool Pattern

**Description:** Efficiently manage goroutines for concurrent processing using channels.

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

type Job struct {
    ID   int
    Data string
}

type Result struct {
    Job    Job
    Output string
    Error  error
}

func worker(id int, jobs <-chan Job, results chan<- Result, wg *sync.WaitGroup) {
    defer wg.Done()
    
    for job := range jobs {
        // Simulate work
        time.Sleep(100 * time.Millisecond)
        
        result := Result{
            Job:    job,
            Output: fmt.Sprintf("Worker %d processed job %d: %s", id, job.ID, job.Data),
        }
        
        results <- result
    }
}

func main() {
    const numWorkers = 3
    const numJobs = 10
    
    jobs := make(chan Job, numJobs)
    results := make(chan Result, numJobs)
    
    var wg sync.WaitGroup
    
    // Start workers
    for i := 1; i <= numWorkers; i++ {
        wg.Add(1)
        go worker(i, jobs, results, &wg)
    }
    
    // Send jobs
    for i := 1; i <= numJobs; i++ {
        jobs <- Job{ID: i, Data: fmt.Sprintf("task-%d", i)}
    }
    close(jobs)
    
    // Close results channel when all workers are done
    go func() {
        wg.Wait()
        close(results)
    }()
    
    // Collect results
    for result := range results {
        fmt.Println(result.Output)
    }
}
```

### 5. Context Pattern

**Description:** Use context for cancellation, timeouts, and passing request-scoped values.

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func longRunningOperation(ctx context.Context, id string) error {
    select {
    case <-time.After(2 * time.Second):
        fmt.Printf("Operation %s completed\n", id)
        return nil
    case <-ctx.Done():
        fmt.Printf("Operation %s cancelled: %v\n", id, ctx.Err())
        return ctx.Err()
    }
}

func main() {
    // Context with timeout
    ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
    defer cancel()
    
    err := longRunningOperation(ctx, "task1")
    if err != nil {
        fmt.Printf("Error: %v\n", err)
    }
    
    // Context with cancellation
    ctx2, cancel2 := context.WithCancel(context.Background())
    
    go func() {
        time.Sleep(500 * time.Millisecond)
        cancel2() // Cancel after 500ms
    }()
    
    err = longRunningOperation(ctx2, "task2")
    if err != nil {
        fmt.Printf("Error: %v\n", err)
    }
}
```

### 6. Builder Pattern

**Description:** Create complex objects step by step with method chaining.

```go
type HTTPClient struct {
    baseURL string
    timeout time.Duration
    headers map[string]string
    client  *http.Client
}

type HTTPClientBuilder struct {
    client *HTTPClient
}

func NewHTTPClientBuilder() *HTTPClientBuilder {
    return &HTTPClientBuilder{
        client: &HTTPClient{
            timeout: 30 * time.Second,
            headers: make(map[string]string),
        },
    }
}

func (b *HTTPClientBuilder) BaseURL(url string) *HTTPClientBuilder {
    b.client.baseURL = url
    return b
}

func (b *HTTPClientBuilder) Timeout(timeout time.Duration) *HTTPClientBuilder {
    b.client.timeout = timeout
    return b
}

func (b *HTTPClientBuilder) Header(key, value string) *HTTPClientBuilder {
    b.client.headers[key] = value
    return b
}

func (b *HTTPClientBuilder) Build() *HTTPClient {
    b.client.client = &http.Client{
        Timeout: b.client.timeout,
    }
    return b.client
}

// Usage
func main() {
    client := NewHTTPClientBuilder().
        BaseURL("https://api.example.com").
        Timeout(60 * time.Second).
        Header("User-Agent", "MyApp/1.0").
        Header("Authorization", "Bearer token").
        Build()
}
```

### 7. Pipeline Pattern

**Description:** Process data through a series of stages using channels.

```go
package main

import (
    "fmt"
    "strconv"
    "strings"
)

func generator(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums {
            out <- n
        }
    }()
    return out
}

func square(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            out <- n * n
        }
    }()
    return out
}

func toString(in <-chan int) <-chan string {
    out := make(chan string)
    go func() {
        defer close(out)
        for n := range in {
            out <- strconv.Itoa(n)
        }
    }()
    return out
}

func main() {
    // Pipeline: generate numbers → square them → convert to strings
    numbers := generator(1, 2, 3, 4, 5)
    squared := square(numbers)
    strings := toString(squared)
    
    for s := range strings {
        fmt.Println(s)
    }
}
```

---

## Go Anti-Patterns

### 1. Ignoring Errors

**Anti-Pattern:** Ignoring errors or handling them poorly.

```go
// BAD: Ignoring errors
func badExample() {
    file, _ := os.Open("config.json") // Ignoring error
    defer file.Close()
    // This will panic if file is nil
}

// BAD: Generic error handling
func anotherBadExample() {
    if err != nil {
        log.Fatal("something went wrong")
    }
}
```

**Good Practice:**
```go
// GOOD: Proper error handling
func goodExample() error {
    file, err := os.Open("config.json")
    if err != nil {
        return fmt.Errorf("failed to open config file: %w", err)
    }
    defer file.Close()
    
    // Process file...
    return nil
}

// GOOD: Specific error handling
func handleSpecificErrors(filename string) error {
    file, err := os.Open(filename)
    if err != nil {
        if os.IsNotExist(err) {
            return fmt.Errorf("config file %s does not exist", filename)
        }
        return fmt.Errorf("failed to open config file %s: %w", filename, err)
    }
    defer file.Close()
    
    return nil
}
```

### 2. Goroutine Leaks

**Anti-Pattern:** Starting goroutines without proper cleanup or termination.

```go
// BAD: Goroutine leak
func badGoroutineExample() {
    ch := make(chan int)
    
    go func() {
        for {
            select {
            case val := <-ch:
                fmt.Println(val)
            }
        }
    }() // This goroutine will never terminate
    
    // Function returns but goroutine keeps running
}
```

**Good Practice:**
```go
// GOOD: Proper goroutine management
func goodGoroutineExample(ctx context.Context) {
    ch := make(chan int)
    
    go func() {
        defer close(ch)
        for {
            select {
            case val := <-ch:
                fmt.Println(val)
            case <-ctx.Done():
                fmt.Println("Goroutine terminating")
                return
            }
        }
    }()
}

// GOOD: Using WaitGroup for coordination
func goodWaitGroupExample() {
    var wg sync.WaitGroup
    
    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            fmt.Printf("Worker %d completed\n", id)
        }(i)
    }
    
    wg.Wait() // Wait for all goroutines to complete
}
```

### 3. Inappropriate Use of Channels

**Anti-Pattern:** Using channels when simpler alternatives exist.

```go
// BAD: Overusing channels for simple synchronization
type Counter struct {
    ch chan struct{}
    count int
}

func (c *Counter) Increment() {
    c.ch <- struct{}{}
    c.count++
    <-c.ch
}
```

**Good Practice:**
```go
// GOOD: Use mutex for simple synchronization
type Counter struct {
    mu    sync.Mutex
    count int
}

func (c *Counter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count++
}

func (c *Counter) Value() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.count
}
```

### 4. Premature Interface Abstraction

**Anti-Pattern:** Creating interfaces before they're needed.

```go
// BAD: Unnecessary interface
type UserService interface {
    GetUser(id int) (*User, error)
    CreateUser(user *User) error
    UpdateUser(user *User) error
    DeleteUser(id int) error
}

type DatabaseUserService struct {
    db *sql.DB
}

func (s *DatabaseUserService) GetUser(id int) (*User, error) {
    // Implementation
}

// Only one implementation exists, interface is premature
```

**Good Practice:**
```go
// GOOD: Create interfaces when you have multiple implementations
// or when you need to define behavior contracts

// Start with concrete type
type UserService struct {
    db *sql.DB
}

func (s *UserService) GetUser(id int) (*User, error) {
    // Implementation
}

// Create interface later when needed (e.g., for testing or multiple implementations)
type UserRepository interface {
    GetUser(id int) (*User, error)
}
```

### 5. Improper Resource Management

**Anti-Pattern:** Not properly closing resources or handling cleanup.

```go
// BAD: Resource leaks
func badResourceExample() {
    resp, err := http.Get("https://api.example.com")
    if err != nil {
        return
    }
    // Missing resp.Body.Close() - resource leak!
    
    body, _ := ioutil.ReadAll(resp.Body)
    fmt.Println(string(body))
}

// BAD: Deferred function in loop
func badDeferInLoop() {
    for i := 0; i < 100; i++ {
        file, err := os.Open(fmt.Sprintf("file%d.txt", i))
        if err != nil {
            continue
        }
        defer file.Close() // This will accumulate, files won't close until function returns
        
        // Process file...
    }
}
```

**Good Practice:**
```go
// GOOD: Proper resource management
func goodResourceExample() error {
    resp, err := http.Get("https://api.example.com")
    if err != nil {
        return fmt.Errorf("failed to make request: %w", err)
    }
    defer resp.Body.Close() // Ensure body is closed
    
    if resp.StatusCode != http.StatusOK {
        return fmt.Errorf("unexpected status code: %d", resp.StatusCode)
    }
    
    body, err := ioutil.ReadAll(resp.Body)
    if err != nil {
        return fmt.Errorf("failed to read response body: %w", err)
    }
    
    fmt.Println(string(body))
    return nil
}

// GOOD: Proper defer usage in loops
func goodDeferInLoop() error {
    for i := 0; i < 100; i++ {
        if err := processFile(fmt.Sprintf("file%d.txt", i)); err != nil {
            return err
        }
    }
    return nil
}

func processFile(filename string) error {
    file, err := os.Open(filename)
    if err != nil {
        return err
    }
    defer file.Close() // Properly scoped defer
    
    // Process file...
    return nil
}
```

### 6. Mutex Deadlocks

**Anti-Pattern:** Acquiring multiple locks in different orders leading to deadlocks.

```go
// BAD: Potential deadlock
type Account struct {
    mu      sync.Mutex
    balance int
}

func transfer(from, to *Account, amount int) {
    from.mu.Lock()
    defer from.mu.Unlock()
    
    to.mu.Lock()   // Potential deadlock if another goroutine
    defer to.mu.Unlock()  // locks in reverse order
    
    from.balance -= amount
    to.balance += amount
}
```

**Good Practice:**
```go
// GOOD: Consistent lock ordering
func safeTransfer(from, to *Account, amount int) {
    // Always acquire locks in the same order (e.g., by memory address)
    if uintptr(unsafe.Pointer(from)) < uintptr(unsafe.Pointer(to)) {
        from.mu.Lock()
        defer from.mu.Unlock()
        to.mu.Lock()
        defer to.mu.Unlock()
    } else {
        to.mu.Lock()
        defer to.mu.Unlock()
        from.mu.Lock()
        defer from.mu.Unlock()
    }
    
    from.balance -= amount
    to.balance += amount
}

// BETTER: Use a single lock for the operation
type Bank struct {
    mu       sync.Mutex
    accounts map[string]*Account
}

func (b *Bank) Transfer(fromID, toID string, amount int) error {
    b.mu.Lock()
    defer b.mu.Unlock()
    
    from, ok := b.accounts[fromID]
    if !ok {
        return errors.New("source account not found")
    }
    
    to, ok := b.accounts[toID]
    if !ok {
        return errors.New("destination account not found")
    }
    
    if from.balance < amount {
        return errors.New("insufficient funds")
    }
    
    from.balance -= amount
    to.balance += amount
    return nil
}
```

### 7. Empty Interface Overuse

**Anti-Pattern:** Using `interface{}` instead of specific types or generics.

```go
// BAD: Overusing empty interface
func badProcessData(data interface{}) {
    switch v := data.(type) {
    case string:
        fmt.Println("Processing string:", v)
    case int:
        fmt.Println("Processing int:", v)
    case []interface{}:
        fmt.Println("Processing slice:", v)
    default:
        fmt.Println("Unknown type")
    }
}
```

**Good Practice:**
```go
// GOOD: Use specific types
func processString(data string) {
    fmt.Println("Processing string:", data)
}

func processInt(data int) {
    fmt.Println("Processing int:", data)
}

// GOOD: Use generics (Go 1.18+)
func processGeneric[T any](data T, processor func(T)) {
    processor(data)
}

// GOOD: Use interfaces with defined methods
type Processor interface {
    Process() error
}

func processData(p Processor) error {
    return p.Process()
}
```

---

## Best Practices Summary

### Do's:
1. **Always handle errors explicitly** - Don't ignore them
2. **Use small, focused interfaces** - Keep them simple and composable
3. **Manage goroutines properly** - Always have a way to terminate them
4. **Use context for cancellation** - Pass context through call chains
5. **Close resources properly** - Use defer for cleanup
6. **Use channels for communication** - Mutexes for synchronization
7. **Write self-documenting code** - Clear variable and function names
8. **Use Go's built-in tools** - gofmt, go vet, golint

### Don'ts:
1. **Don't ignore errors** - Handle them appropriately
2. **Don't create goroutine leaks** - Always provide termination mechanism
3. **Don't overuse channels** - Use appropriate synchronization primitives
4. **Don't create premature abstractions** - Interfaces should be discovered, not designed
5. **Don't use empty interfaces unnecessarily** - Prefer specific types
6. **Don't acquire multiple locks carelessly** - Avoid deadlocks
7. **Don't defer in loops** - It can cause resource exhaustion
8. **Don't panic in libraries** - Return errors instead

### Performance Tips:
- Use `strings.Builder` for efficient string concatenation
- Prefer `sync.Pool` for object reuse in high-performance scenarios
- Use buffered channels appropriately to avoid blocking
- Profile your code with `go tool pprof` to identify bottlenecks
- Consider using `sync.Once` for expensive one-time initialization

### Code Organization:
- Keep packages focused and cohesive
- Use meaningful package names
- Avoid circular dependencies
- Write tests alongside your code
- Use Go modules for dependency management