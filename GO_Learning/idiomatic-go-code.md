# Idiomatic Go Code Cheat Sheet

## 1. Go Philosophy & Proverbs

### Core Philosophy
- **Simplicity over cleverness**
- **Clarity over brevity**
- **Composition over inheritance**
- **Concurrency over parallelism**
- **Errors are values**
- **Don't communicate by sharing memory; share memory by communicating**

### Go Proverbs (by Rob Pike)
```go
// Don't communicate by sharing memory, share memory by communicating
// Concurrency is not parallelism
// Channels orchestrate; mutexes serialize
// The bigger the interface, the weaker the abstraction
// Make the zero value useful
// interface{} says nothing
// Gofmt's style is no one's favorite, yet gofmt is everyone's favorite
// A little copying is better than a little dependency
// Syscall must always be guarded with build tags
// Cgo must always be guarded with build tags
// Cgo is not Go
// With the unsafe package there are no guarantees
// Clear is better than clever
// Reflection is never clear
// Errors are values
// Don't just check errors, handle them gracefully
// Design the architecture, name the components, document the details
// Documentation is for users
// Don't panic
```

## 2. Naming Conventions

### Package Names
```go
// Good: short, lowercase, no underscores
package http
package json
package user

// Bad
package userManager      // No camelCase
package user_manager     // No underscores
package UserManager      // No capitals
```

### Variable Names
```go
// Short names for short-lived variables
for i := 0; i < 10; i++ {
    // i is perfectly fine here
}

// Descriptive names for package-level variables
var defaultTimeout = 30 * time.Second
var maxRetries = 3

// Acronyms should be all caps
var userID string    // not userId
var httpClient *http.Client  // not httpClient
var xmlData []byte   // not xmlData

// Interface parameter names can be short
func (c *Client) Get(ctx context.Context, id string) (*User, error)

// Receiver names should be short (1-2 letters)
func (c *Client) Connect() error { }    // Good
func (client *Client) Connect() error { } // Too verbose
```

### Function Names
```go
// Exported functions start with capital
func ParseJSON(data []byte) error { }

// Unexported functions start with lowercase
func parseJSON(data []byte) error { }

// Getters don't use Get prefix
type Person struct {
    name string
}

// Good
func (p *Person) Name() string {
    return p.name
}

// Bad
func (p *Person) GetName() string {
    return p.name
}

// But setters do use Set
func (p *Person) SetName(name string) {
    p.name = name
}
```

### Interface Names
```go
// Single-method interfaces end with -er
type Reader interface {
    Read([]byte) (int, error)
}

type Stringer interface {
    String() string
}

// Don't use I prefix
type IReader interface { } // Bad
type Reader interface { }  // Good
```

## 3. Code Organization

### Package Structure
```go
// Standard project layout
myproject/
├── cmd/
│   └── myapp/
│       └── main.go
├── internal/
│   ├── config/
│   ├── handler/
│   └── service/
├── pkg/
│   └── client/
├── go.mod
├── go.sum
└── README.md
```

### Import Organization
```go
import (
    // Standard library
    "context"
    "fmt"
    "time"
    
    // Third-party packages
    "github.com/gorilla/mux"
    "github.com/pkg/errors"
    
    // Local packages
    "github.com/myuser/myproject/internal/config"
    "github.com/myuser/myproject/pkg/client"
)
```

### File Organization
```go
// Order within a file:
// 1. Package clause
package main

// 2. Import declarations
import (
    "fmt"
    "log"
)

// 3. Constants
const defaultPort = 8080

// 4. Variables
var (
    version = "1.0.0"
    build   = "unknown"
)

// 5. Types
type Server struct {
    port int
}

// 6. Constructor/New functions
func NewServer(port int) *Server {
    return &Server{port: port}
}

// 7. Methods (grouped by receiver)
func (s *Server) Start() error {
    // Implementation
}

// 8. Functions
func main() {
    // Implementation
}
```

## 4. Error Handling

### Error Checking
```go
// Always check errors immediately
data, err := ioutil.ReadFile("file.txt")
if err != nil {
    return fmt.Errorf("failed to read file: %w", err)
}

// Don't use blank identifier for errors
data, _ := ioutil.ReadFile("file.txt") // Bad!

// Exception: When you truly don't care
_ = os.Remove(tempFile) // Cleanup, OK to ignore
```

### Error Types
```go
// Sentinel errors
var (
    ErrNotFound   = errors.New("not found")
    ErrInvalid    = errors.New("invalid input")
    ErrPermission = errors.New("permission denied")
)

// Custom error types
type ValidationError struct {
    Field string
    Err   error
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation failed on field %s: %v", e.Field, e.Err)
}

// Error wrapping (Go 1.13+)
if err != nil {
    return fmt.Errorf("failed to process user %s: %w", userID, err)
}

// Error checking
if errors.Is(err, ErrNotFound) {
    // Handle not found
}

var valErr *ValidationError
if errors.As(err, &valErr) {
    // Handle validation error
}
```

### Error Handling Patterns
```go
// Return early on error
func ProcessFile(filename string) error {
    file, err := os.Open(filename)
    if err != nil {
        return fmt.Errorf("failed to open file: %w", err)
    }
    defer file.Close()
    
    data, err := readData(file)
    if err != nil {
        return fmt.Errorf("failed to read data: %w", err)
    }
    
    return processData(data)
}

// Error handling in loops
for _, item := range items {
    if err := process(item); err != nil {
        // Log and continue, or return based on requirements
        log.Printf("failed to process item %v: %v", item, err)
        continue
    }
}
```

## 5. Interfaces

### Interface Design
```go
// Small interfaces are better
type Reader interface {
    Read([]byte) (int, error)
}

type Writer interface {
    Write([]byte) (int, error)
}

// Compose when needed
type ReadWriter interface {
    Reader
    Writer
}

// Don't create interfaces prematurely
// Bad: Creating interface before you need it
type UserInterface interface {
    GetUser(id string) (*User, error)
    CreateUser(user *User) error
    UpdateUser(user *User) error
    DeleteUser(id string) error
}

// Good: Start with concrete type, extract interface when needed
type UserService struct {
    db *sql.DB
}

func (s *UserService) GetUser(id string) (*User, error) {
    // Implementation
}
```

### Accept Interfaces, Return Structs
```go
// Good: Accept interface
func SaveData(w io.Writer, data []byte) error {
    _, err := w.Write(data)
    return err
}

// Good: Return concrete type
func NewBuffer() *bytes.Buffer {
    return &bytes.Buffer{}
}

// Avoid returning interfaces unless necessary
func NewWriter() io.Writer { // Usually unnecessary
    return &bytes.Buffer{}
}
```

## 6. Concurrency

### Goroutines
```go
// Always handle goroutine lifecycle
func Worker(ctx context.Context, jobs <-chan Job) {
    for {
        select {
        case job, ok := <-jobs:
            if !ok {
                return // Channel closed
            }
            process(job)
        case <-ctx.Done():
            return // Context cancelled
        }
    }
}

// Don't forget to wait
var wg sync.WaitGroup
for i := 0; i < workers; i++ {
    wg.Add(1)
    go func(id int) {
        defer wg.Done()
        worker(id)
    }(i)
}
wg.Wait()

// Pass parameters explicitly
for i := 0; i < 10; i++ {
    go func(n int) {
        fmt.Println(n)
    }(i) // Pass i as parameter
}
```

### Channels
```go
// Make channel direction clear
func send(ch chan<- int) {
    ch <- 42
}

func receive(ch <-chan int) {
    val := <-ch
    fmt.Println(val)
}

// Close channels from sender
func producer(ch chan<- int) {
    defer close(ch) // Only sender should close
    for i := 0; i < 10; i++ {
        ch <- i
    }
}

// Check if channel is closed
val, ok := <-ch
if !ok {
    // Channel is closed
}

// Use select for non-blocking operations
select {
case val := <-ch:
    // Got value
case <-time.After(1 * time.Second):
    // Timeout
default:
    // Non-blocking
}
```

### Synchronization
```go
// Use sync.Once for one-time initialization
var (
    instance *Singleton
    once     sync.Once
)

func GetInstance() *Singleton {
    once.Do(func() {
        instance = &Singleton{}
    })
    return instance
}

// Use mutex appropriately
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

// Use RWMutex for read-heavy workloads
type Cache struct {
    mu   sync.RWMutex
    data map[string]string
}

func (c *Cache) Get(key string) (string, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    val, ok := c.data[key]
    return val, ok
}
```

## 7. Testing

### Test Organization
```go
// Test file naming: *_test.go
// calculator.go -> calculator_test.go

// Test function naming
func TestFunctionName(t *testing.T) { }
func TestTypeName_MethodName(t *testing.T) { }

// Table-driven tests
func TestAdd(t *testing.T) {
    tests := []struct {
        name string
        a, b int
        want int
    }{
        {"positive", 2, 3, 5},
        {"negative", -1, -1, -2},
        {"zero", 0, 0, 0},
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := Add(tt.a, tt.b)
            if got != tt.want {
                t.Errorf("Add(%d, %d) = %d, want %d",
                    tt.a, tt.b, got, tt.want)
            }
        })
    }
}
```

### Test Helpers
```go
// Test helpers should call t.Helper()
func createTestServer(t *testing.T) *Server {
    t.Helper()
    
    srv := NewServer()
    if err := srv.Start(); err != nil {
        t.Fatalf("failed to start server: %v", err)
    }
    
    t.Cleanup(func() {
        srv.Stop()
    })
    
    return srv
}

// Use t.Cleanup for cleanup
func TestWithTempFile(t *testing.T) {
    tmpfile, err := ioutil.TempFile("", "test")
    if err != nil {
        t.Fatal(err)
    }
    t.Cleanup(func() {
        os.Remove(tmpfile.Name())
    })
    
    // Use tmpfile
}
```

## 8. Common Patterns

### Constructor Functions
```go
// Use NewType for constructors
func NewClient(baseURL string) *Client {
    return &Client{
        baseURL:    baseURL,
        httpClient: &http.Client{Timeout: 30 * time.Second},
    }
}

// Options pattern for complex constructors
type Option func(*Server)

func WithPort(port int) Option {
    return func(s *Server) {
        s.port = port
    }
}

func NewServer(opts ...Option) *Server {
    s := &Server{
        port: 8080, // default
    }
    
    for _, opt := range opts {
        opt(s)
    }
    
    return s
}
```

### Zero Values
```go
// Make zero values useful
type Buffer struct {
    buf []byte
}

// Works with zero value
func (b *Buffer) Write(p []byte) (int, error) {
    b.buf = append(b.buf, p...)
    return len(p), nil
}

// Usage without initialization
var buf Buffer
buf.Write([]byte("hello")) // Works!

// sync.Mutex zero value is unlocked
var mu sync.Mutex
mu.Lock() // Works without initialization
```

### Method Receivers
```go
// Use pointer receivers when:
// 1. Method modifies the receiver
func (p *Person) SetName(name string) {
    p.name = name
}

// 2. Receiver is large
func (b *BigStruct) Method() {
    // Avoid copying large struct
}

// 3. Consistency (if one method has pointer receiver, all should)
type Counter struct {
    count int
}

func (c *Counter) Increment() { c.count++ }
func (c *Counter) Value() int { return c.count } // Also pointer

// Use value receivers when:
// 1. Receiver is small and immutable
func (p Point) Distance(q Point) float64 {
    return math.Sqrt(math.Pow(p.X-q.X, 2) + math.Pow(p.Y-q.Y, 2))
}
```

## 9. Performance Tips

### String Building
```go
// Bad: String concatenation in loop
var s string
for i := 0; i < 1000; i++ {
    s += "a" // Creates new string each time
}

// Good: Use strings.Builder
var b strings.Builder
for i := 0; i < 1000; i++ {
    b.WriteString("a")
}
s := b.String()

// For known size, pre-allocate
var b strings.Builder
b.Grow(1000)
```

### Slice Operations
```go
// Pre-allocate slices when size is known
data := make([]int, 0, 100) // cap: 100
for i := 0; i < 100; i++ {
    data = append(data, i)
}

// Reuse slices
buf := make([]byte, 1024)
for {
    n, err := r.Read(buf)
    if err != nil {
        break
    }
    process(buf[:n])
}
```

### Map Initialization
```go
// Initialize with capacity for better performance
users := make(map[string]*User, 1000)

// Check existence efficiently
if user, ok := users[id]; ok {
    // Use user
}

// Delete from map
delete(users, id) // Safe even if key doesn't exist
```

## 10. Code Smells & Anti-patterns

### Things to Avoid
```go
// Don't use init() unless necessary
func init() {
    // Often makes testing harder
}

// Don't panic in libraries
func LibraryFunc() {
    panic("something went wrong") // Bad!
}

// Don't ignore errors
result, _ := doSomething() // Bad!

// Don't use empty interfaces unnecessarily
func Process(data interface{}) { // Avoid when possible
    // Type assertions everywhere
}

// Don't create huge interfaces
type DoEverything interface {
    Method1()
    Method2()
    // ... 20 more methods
}

// Don't use globals
var globalConfig *Config // Avoid

// Don't return concrete error types from public APIs
func PublicFunc() *MyError { // Bad
    return &MyError{}
}

func PublicFunc() error { // Good
    return &MyError{}
}
```

### Code Review Checklist
```go
// ✓ Are errors handled?
// ✓ Are goroutines properly managed?
// ✓ Are channels closed by sender?
// ✓ Are mutexes used correctly?
// ✓ Is the code testable?
// ✓ Are interfaces small and focused?
// ✓ Are names clear and idiomatic?
// ✓ Is the zero value useful?
// ✓ Is the code formatted with gofmt?
// ✓ Are comments clear and necessary?
```

## 11. Documentation

### Package Documentation
```go
// Package user provides user management functionality.
// 
// This package includes user authentication, authorization,
// and profile management features.
package user

// Always document exported types and functions
// UserService manages user-related operations.
type UserService struct {
    db *sql.DB
}

// NewUserService creates a new instance of UserService.
// It requires a valid database connection.
func NewUserService(db *sql.DB) *UserService {
    return &UserService{db: db}
}

// GetUser retrieves a user by ID.
// It returns ErrNotFound if the user doesn't exist.
func (s *UserService) GetUser(id string) (*User, error) {
    // Implementation
}
```

### Examples in Documentation
```go
// Example functions in test files
func ExampleUserService_GetUser() {
    svc := NewUserService(db)
    user, err := svc.GetUser("123")
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println(user.Name)
    // Output: John Doe
}
```

## 12. Tools & Commands

### Essential Go Commands
```bash
# Format code
go fmt ./...
gofmt -s -w .

# Lint code
golangci-lint run

# Run tests
go test ./...
go test -race ./...
go test -cover ./...

# Benchmarks
go test -bench=.
go test -bench=. -benchmem

# Check for race conditions
go run -race main.go

# Generate documentation
go doc package
go doc package.Function

# Module management
go mod init
go mod tidy
go mod vendor
```

## Summary

Writing idiomatic Go means:
1. **Embrace simplicity** - Don't be clever when simple works
2. **Handle errors explicitly** - No exceptions, errors are values
3. **Use composition** - Embed types and interfaces
4. **Keep interfaces small** - The bigger the interface, the weaker the abstraction
5. **Make zero values useful** - Types should work without initialization when possible
6. **Document public APIs** - Clear, concise documentation
7. **Format consistently** - Use gofmt, no debates
8. **Test thoroughly** - Table-driven tests, benchmarks
9. **Profile before optimizing** - Don't guess, measure
10. **Follow conventions** - When in Rome, do as Gophers do