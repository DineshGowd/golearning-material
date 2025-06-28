# Go Learning - Hard Level

## 1. Advanced Concurrency Patterns

### Worker Pool Pattern
```go
type Job struct {
    ID   int
    Data string
}

type Result struct {
    JobID  int
    Output string
    Error  error
}

func workerPool(jobs <-chan Job, results chan<- Result, workers int) {
    var wg sync.WaitGroup
    
    // Start workers
    for i := 0; i < workers; i++ {
        wg.Add(1)
        go func(workerID int) {
            defer wg.Done()
            for job := range jobs {
                // Process job
                output := processJob(job)
                results <- Result{
                    JobID:  job.ID,
                    Output: output,
                }
            }
        }(i)
    }
    
    // Wait for all workers to finish
    wg.Wait()
    close(results)
}

// Usage
jobs := make(chan Job, 100)
results := make(chan Result, 100)

go workerPool(jobs, results, 5)

// Send jobs
go func() {
    for i := 0; i < 50; i++ {
        jobs <- Job{ID: i, Data: fmt.Sprintf("job-%d", i)}
    }
    close(jobs)
}()

// Collect results
for result := range results {
    fmt.Printf("Job %d: %s\n", result.JobID, result.Output)
}
```

### Fan-In/Fan-Out Pattern
```go
// Fan-out: distribute work
func fanOut(in <-chan int, workers int) []<-chan int {
    channels := make([]<-chan int, workers)
    
    for i := 0; i < workers; i++ {
        ch := make(chan int)
        channels[i] = ch
        
        go func(output chan int) {
            for val := range in {
                output <- val * val // Process
            }
            close(output)
        }(ch)
    }
    
    return channels
}

// Fan-in: merge results
func fanIn(channels ...<-chan int) <-chan int {
    out := make(chan int)
    var wg sync.WaitGroup
    
    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan int) {
            defer wg.Done()
            for val := range c {
                out <- val
            }
        }(ch)
    }
    
    go func() {
        wg.Wait()
        close(out)
    }()
    
    return out
}
```

### Pipeline Pattern
```go
// Pipeline stages
func generate(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        for _, n := range nums {
            out <- n
        }
        close(out)
    }()
    return out
}

func square(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            out <- n * n
        }
        close(out)
    }()
    return out
}

func filter(in <-chan int, predicate func(int) bool) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            if predicate(n) {
                out <- n
            }
        }
        close(out)
    }()
    return out
}

// Usage
numbers := generate(1, 2, 3, 4, 5)
squared := square(numbers)
evens := filter(squared, func(n int) bool { return n%2 == 0 })

for n := range evens {
    fmt.Println(n) // 4, 16
}
```

## 2. Sync Package Advanced Usage

```go
import "sync"

// Mutex for safe concurrent access
type SafeCounter struct {
    mu    sync.Mutex
    value int
}

func (c *SafeCounter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.value++
}

func (c *SafeCounter) Value() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.value
}

// RWMutex for read-heavy workloads
type Cache struct {
    mu    sync.RWMutex
    data  map[string]string
}

func (c *Cache) Get(key string) (string, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    val, ok := c.data[key]
    return val, ok
}

func (c *Cache) Set(key, value string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.data[key] = value
}

// Once for one-time initialization
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

// Pool for reusable objects
var bufferPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

func processWithPool(data []byte) {
    buf := bufferPool.Get().(*bytes.Buffer)
    defer func() {
        buf.Reset()
        bufferPool.Put(buf)
    }()
    
    buf.Write(data)
    // Process buffer
}

// Cond for complex synchronization
type Queue struct {
    items []string
    cond  *sync.Cond
}

func NewQueue() *Queue {
    return &Queue{
        cond: sync.NewCond(&sync.Mutex{}),
    }
}

func (q *Queue) Put(item string) {
    q.cond.L.Lock()
    defer q.cond.L.Unlock()
    
    q.items = append(q.items, item)
    q.cond.Signal() // Wake one waiter
}

func (q *Queue) Get() string {
    q.cond.L.Lock()
    defer q.cond.L.Unlock()
    
    for len(q.items) == 0 {
        q.cond.Wait() // Release lock and wait
    }
    
    item := q.items[0]
    q.items = q.items[1:]
    return item
}
```

## 3. Context Package

```go
import (
    "context"
    "time"
)

// Context with timeout
func doWorkWithTimeout(ctx context.Context) error {
    ctx, cancel := context.WithTimeout(ctx, 2*time.Second)
    defer cancel()
    
    select {
    case <-time.After(3 * time.Second):
        return errors.New("work took too long")
    case <-ctx.Done():
        return ctx.Err()
    }
}

// Context with cancellation
func processWithCancel() {
    ctx, cancel := context.WithCancel(context.Background())
    
    go func() {
        time.Sleep(2 * time.Second)
        cancel() // Cancel all operations
    }()
    
    doWork(ctx)
}

// Context with values
type key string

const userIDKey key = "userID"

func withUserID(ctx context.Context, userID string) context.Context {
    return context.WithValue(ctx, userIDKey, userID)
}

func getUserID(ctx context.Context) (string, bool) {
    userID, ok := ctx.Value(userIDKey).(string)
    return userID, ok
}

// HTTP request with context
func makeRequest(ctx context.Context, url string) error {
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return err
    }
    
    client := &http.Client{}
    resp, err := client.Do(req)
    if err != nil {
        return err
    }
    defer resp.Body.Close()
    
    return nil
}

// Database operations with context
func queryDB(ctx context.Context, query string) error {
    conn, err := db.Conn(ctx)
    if err != nil {
        return err
    }
    defer conn.Close()
    
    rows, err := conn.QueryContext(ctx, query)
    if err != nil {
        return err
    }
    defer rows.Close()
    
    // Process rows
    return nil
}
```

## 4. Reflection

```go
import "reflect"

// Basic reflection
func examineType(i interface{}) {
    t := reflect.TypeOf(i)
    v := reflect.ValueOf(i)
    
    fmt.Printf("Type: %v\n", t)
    fmt.Printf("Kind: %v\n", t.Kind())
    fmt.Printf("Value: %v\n", v)
}

// Struct field inspection
type User struct {
    Name  string `json:"name" validate:"required"`
    Age   int    `json:"age" validate:"min=0,max=150"`
    Email string `json:"email" validate:"email"`
}

func inspectStruct(s interface{}) {
    t := reflect.TypeOf(s)
    
    for i := 0; i < t.NumField(); i++ {
        field := t.Field(i)
        fmt.Printf("Field: %s, Type: %s, Tag: %s\n",
            field.Name,
            field.Type,
            field.Tag.Get("validate"))
    }
}

// Dynamic method calls
func callMethod(obj interface{}, methodName string, args ...interface{}) {
    v := reflect.ValueOf(obj)
    method := v.MethodByName(methodName)
    
    if !method.IsValid() {
        panic("method not found")
    }
    
    // Convert args to reflect.Value
    inputs := make([]reflect.Value, len(args))
    for i, arg := range args {
        inputs[i] = reflect.ValueOf(arg)
    }
    
    results := method.Call(inputs)
    // Process results
}

// Generic unmarshaling
func unmarshalToStruct(data []byte, result interface{}) error {
    rv := reflect.ValueOf(result)
    if rv.Kind() != reflect.Ptr || rv.IsNil() {
        return errors.New("result must be a non-nil pointer")
    }
    
    // Custom unmarshaling logic
    return json.Unmarshal(data, result)
}

// Creating instances dynamically
func createInstance(t reflect.Type) interface{} {
    if t.Kind() == reflect.Ptr {
        t = t.Elem()
    }
    return reflect.New(t).Interface()
}
```

## 5. Generics (Go 1.18+)

```go
// Generic function
func Min[T constraints.Ordered](a, b T) T {
    if a < b {
        return a
    }
    return b
}

// Generic type
type Stack[T any] struct {
    items []T
}

func (s *Stack[T]) Push(item T) {
    s.items = append(s.items, item)
}

func (s *Stack[T]) Pop() (T, bool) {
    var zero T
    if len(s.items) == 0 {
        return zero, false
    }
    item := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1]
    return item, true
}

// Custom constraints
type Number interface {
    ~int | ~int64 | ~float64
}

func Sum[T Number](values []T) T {
    var sum T
    for _, v := range values {
        sum += v
    }
    return sum
}

// Generic interfaces
type Container[T any] interface {
    Add(T)
    Get(int) (T, bool)
    Size() int
}

// Type inference
func Map[T, U any](slice []T, fn func(T) U) []U {
    result := make([]U, len(slice))
    for i, v := range slice {
        result[i] = fn(v)
    }
    return result
}

// Usage without explicit type
doubled := Map([]int{1, 2, 3}, func(x int) int { return x * 2 })

// Complex generic example
type Result[T any] struct {
    Value T
    Error error
}

func (r Result[T]) IsOK() bool {
    return r.Error == nil
}

func (r Result[T]) Unwrap() T {
    if r.Error != nil {
        panic(r.Error)
    }
    return r.Value
}

func Try[T any](fn func() (T, error)) Result[T] {
    value, err := fn()
    return Result[T]{Value: value, Error: err}
}
```

## 6. Advanced Testing

```go
import (
    "testing"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/mock"
)

// Table-driven tests
func TestCalculate(t *testing.T) {
    tests := []struct {
        name     string
        input    int
        expected int
        wantErr  bool
    }{
        {"positive", 5, 25, false},
        {"zero", 0, 0, false},
        {"negative", -5, 25, false},
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result, err := Calculate(tt.input)
            if tt.wantErr {
                assert.Error(t, err)
            } else {
                assert.NoError(t, err)
                assert.Equal(t, tt.expected, result)
            }
        })
    }
}

// Benchmarks
func BenchmarkProcess(b *testing.B) {
    data := generateTestData()
    b.ResetTimer()
    
    for i := 0; i < b.N; i++ {
        Process(data)
    }
}

// Sub-benchmarks
func BenchmarkSizes(b *testing.B) {
    sizes := []int{10, 100, 1000, 10000}
    
    for _, size := range sizes {
        b.Run(fmt.Sprintf("size-%d", size), func(b *testing.B) {
            data := make([]int, size)
            b.ResetTimer()
            
            for i := 0; i < b.N; i++ {
                ProcessSlice(data)
            }
        })
    }
}

// Mocking
type MockDatabase struct {
    mock.Mock
}

func (m *MockDatabase) Get(key string) (string, error) {
    args := m.Called(key)
    return args.String(0), args.Error(1)
}

func TestService(t *testing.T) {
    mockDB := new(MockDatabase)
    mockDB.On("Get", "user:123").Return("John Doe", nil)
    
    service := NewService(mockDB)
    name, err := service.GetUserName("123")
    
    assert.NoError(t, err)
    assert.Equal(t, "John Doe", name)
    mockDB.AssertExpectations(t)
}

// Fuzzing (Go 1.18+)
func FuzzParse(f *testing.F) {
    // Add seed corpus
    f.Add("valid input")
    f.Add("123")
    f.Add("")
    
    f.Fuzz(func(t *testing.T, input string) {
        result, err := Parse(input)
        if err != nil {
            return // Invalid input is ok
        }
        
        // Verify properties
        assert.NotNil(t, result)
        serialized := result.String()
        reparsed, err := Parse(serialized)
        assert.NoError(t, err)
        assert.Equal(t, result, reparsed)
    })
}
```

## 7. Performance Optimization

```go
// CPU Profiling
import _ "net/http/pprof"

func init() {
    go func() {
        log.Println(http.ListenAndServe("localhost:6060", nil))
    }()
}

// Memory optimization
// Use sync.Pool for frequently allocated objects
var bufPool = sync.Pool{
    New: func() interface{} {
        return make([]byte, 1024)
    },
}

// Avoid allocations in hot paths
type Buffer struct {
    buf []byte
    off int
}

func (b *Buffer) Write(p []byte) (int, error) {
    // Grow buffer if needed (avoid frequent allocations)
    if len(b.buf)-b.off < len(p) {
        newSize := len(b.buf) * 2
        if newSize < b.off+len(p) {
            newSize = b.off + len(p)
        }
        newBuf := make([]byte, newSize)
        copy(newBuf, b.buf[:b.off])
        b.buf = newBuf
    }
    
    copy(b.buf[b.off:], p)
    b.off += len(p)
    return len(p), nil
}

// String building optimization
func buildString(parts []string) string {
    var b strings.Builder
    b.Grow(calculateSize(parts)) // Pre-allocate
    
    for _, part := range parts {
        b.WriteString(part)
    }
    return b.String()
}

// Avoid interface conversions in loops
func sumInts(values []interface{}) int {
    sum := 0
    for _, v := range values {
        if i, ok := v.(int); ok { // This allocates
            sum += i
        }
    }
    return sum
}

// Better: use concrete types
func sumIntsOptimized(values []int) int {
    sum := 0
    for _, v := range values {
        sum += v
    }
    return sum
}

// Concurrent processing for CPU-bound tasks
func processParallel(items []Item) []Result {
    numCPU := runtime.NumCPU()
    runtime.GOMAXPROCS(numCPU)
    
    ch := make(chan Result, len(items))
    sem := make(chan struct{}, numCPU)
    
    var wg sync.WaitGroup
    for _, item := range items {
        wg.Add(1)
        sem <- struct{}{} // Limit concurrency
        
        go func(it Item) {
            defer wg.Done()
            defer func() { <-sem }()
            
            ch <- processItem(it)
        }(item)
    }
    
    go func() {
        wg.Wait()
        close(ch)
    }()
    
    results := make([]Result, 0, len(items))
    for r := range ch {
        results = append(results, r)
    }
    return results
}
```

## 8. Advanced Interface Patterns

```go
// Interface composition
type Reader interface {
    Read([]byte) (int, error)
}

type Writer interface {
    Write([]byte) (int, error)
}

type ReadWriter interface {
    Reader
    Writer
}

// Adapter pattern
type FileAdapter struct {
    *os.File
}

func (fa *FileAdapter) ReadString() (string, error) {
    buf := make([]byte, 1024)
    n, err := fa.Read(buf)
    if err != nil {
        return "", err
    }
    return string(buf[:n]), nil
}

// Strategy pattern
type SortStrategy interface {
    Sort([]int) []int
}

type QuickSort struct{}
func (qs QuickSort) Sort(data []int) []int {
    // Implementation
    return data
}

type MergeSort struct{}
func (ms MergeSort) Sort(data []int) []int {
    // Implementation
    return data
}

type Sorter struct {
    strategy SortStrategy
}

func (s *Sorter) SetStrategy(strategy SortStrategy) {
    s.strategy = strategy
}

func (s *Sorter) Sort(data []int) []int {
    return s.strategy.Sort(data)
}

// Functional options pattern
type Server struct {
    host    string
    port    int
    timeout time.Duration
    tls     bool
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
    s := &Server{
        host:    "localhost",
        port:    8080,
        timeout: 30 * time.Second,
    }
    
    for _, opt := range opts {
        opt(s)
    }
    
    return s
}

// Usage
server := NewServer(
    WithHost("example.com"),
    WithPort(443),
    WithTimeout(60*time.Second),
)
```

## 9. Advanced Error Handling

```go
// Error wrapping and unwrapping
type ValidationError struct {
    Field string
    Err   error
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation error on field %s: %v", e.Field, e.Err)
}

func (e *ValidationError) Unwrap() error {
    return e.Err
}

// Sentinel errors
var (
    ErrNotFound     = errors.New("not found")
    ErrUnauthorized = errors.New("unauthorized")
    ErrInternal     = errors.New("internal error")
)

// Error types with behavior
type RetryableError struct {
    Err        error
    RetryAfter time.Duration
}

func (e *RetryableError) Error() string {
    return fmt.Sprintf("retryable error: %v (retry after %v)", e.Err, e.RetryAfter)
}

func (e *RetryableError) IsRetryable() bool {
    return true
}

// Error handling with context
func processWithRetry(ctx context.Context, fn func() error) error {
    var lastErr error
    
    for attempt := 0; attempt < 3; attempt++ {
        if err := ctx.Err(); err != nil {
            return fmt.Errorf("context cancelled: %w", err)
        }
        
        err := fn()
        if err == nil {
            return nil
        }
        
        var retryable *RetryableError
        if errors.As(err, &retryable) {
            select {
            case <-time.After(retryable.RetryAfter):
                continue
            case <-ctx.Done():
                return ctx.Err()
            }
        }
        
        lastErr = err
        if !isRetryable(err) {
            return err
        }
    }
    
    return fmt.Errorf("max retries exceeded: %w", lastErr)
}
```

## 10. Build Tags and Conditional Compilation

```go
// +build linux

package main

// This file only compiles on Linux

// Multiple build tags
// +build linux,amd64 !cgo

// Development vs Production
// +build !prod

func init() {
    // Development-only initialization
    enableDebugMode()
}

// Custom build tags
// go build -tags=feature1,feature2

// +build feature1

func Feature1Enabled() bool {
    return true
}
```

## Advanced Go Patterns Summary

1. **Concurrency**: Master channels, goroutines, and sync primitives
2. **Context**: Use for cancellation, timeouts, and request-scoped values
3. **Reflection**: Powerful but use sparingly, affects performance
4. **Generics**: Type-safe code reuse without interface{}
5. **Testing**: Comprehensive testing including benchmarks and fuzzing
6. **Performance**: Profile first, optimize later
7. **Error Handling**: Rich errors with context and wrapping
8. **Interfaces**: Small, focused interfaces are better

## Best Practices for Production

1. Always handle errors explicitly
2. Use context for cancellation
3. Avoid goroutine leaks
4. Profile before optimizing
5. Write comprehensive tests
6. Use interfaces for abstraction
7. Keep interfaces small
8. Document exported APIs
9. Use linters (golangci-lint)
10. Follow Go idioms and conventions