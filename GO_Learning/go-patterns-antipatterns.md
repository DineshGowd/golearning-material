# Essential Go Patterns and Anti-patterns

## Table of Contents
1. [Essential Patterns](#essential-patterns)
2. [Anti-patterns to Avoid](#anti-patterns-to-avoid)
3. [Pattern vs Anti-pattern Comparisons](#pattern-vs-anti-pattern-comparisons)

## Essential Patterns

### 1. Functional Options Pattern
**Purpose**: Provide optional configuration with sensible defaults

```go
// ✅ Good: Functional Options
type Server struct {
    host     string
    port     int
    timeout  time.Duration
    maxConns int
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
    // Sensible defaults
    srv := &Server{
        host:     "localhost",
        port:     8080,
        timeout:  30 * time.Second,
        maxConns: 100,
    }
    
    // Apply options
    for _, opt := range opts {
        opt(srv)
    }
    
    return srv
}

// Usage: Clean and flexible
srv := NewServer(
    WithHost("0.0.0.0"),
    WithPort(9090),
    WithTimeout(60*time.Second),
)
```

### 2. Error Wrapping Pattern
**Purpose**: Add context to errors while preserving the original error

```go
// ✅ Good: Error wrapping with context
func ProcessUser(userID string) error {
    user, err := fetchUser(userID)
    if err != nil {
        return fmt.Errorf("process user %s: %w", userID, err)
    }
    
    if err := validateUser(user); err != nil {
        return fmt.Errorf("validate user %s: %w", userID, err)
    }
    
    if err := saveUser(user); err != nil {
        return fmt.Errorf("save user %s: %w", userID, err)
    }
    
    return nil
}

// Error checking with wrapped errors
err := ProcessUser("123")
if errors.Is(err, ErrUserNotFound) {
    // Handle specific error
}

// Custom error types with wrapping
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
```

### 3. Constructor Pattern
**Purpose**: Initialize structs with validation and sensible defaults

```go
// ✅ Good: Constructor with validation
type Client struct {
    baseURL    string
    httpClient *http.Client
    apiKey     string
}

func NewClient(baseURL, apiKey string) (*Client, error) {
    if baseURL == "" {
        return nil, errors.New("base URL is required")
    }
    
    if apiKey == "" {
        return nil, errors.New("API key is required")
    }
    
    // Parse and validate URL
    _, err := url.Parse(baseURL)
    if err != nil {
        return nil, fmt.Errorf("invalid base URL: %w", err)
    }
    
    return &Client{
        baseURL: baseURL,
        apiKey:  apiKey,
        httpClient: &http.Client{
            Timeout: 30 * time.Second,
        },
    }, nil
}
```

### 4. Interface Segregation Pattern
**Purpose**: Keep interfaces small and focused

```go
// ✅ Good: Small, focused interfaces
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

// Accept minimal interface
func Copy(dst Writer, src Reader) error {
    buf := make([]byte, 1024)
    for {
        n, err := src.Read(buf)
        if err == io.EOF {
            return nil
        }
        if err != nil {
            return err
        }
        
        if _, err := dst.Write(buf[:n]); err != nil {
            return err
        }
    }
}
```

### 5. Result Type Pattern
**Purpose**: Handle operations that might fail without relying on multiple return values

```go
// ✅ Good: Result type for complex operations
type Result[T any] struct {
    value T
    err   error
}

func Success[T any](value T) Result[T] {
    return Result[T]{value: value}
}

func Failure[T any](err error) Result[T] {
    return Result[T]{err: err}
}

func (r Result[T]) IsSuccess() bool {
    return r.err == nil
}

func (r Result[T]) Value() (T, error) {
    return r.value, r.err
}

func (r Result[T]) Must() T {
    if r.err != nil {
        panic(r.err)
    }
    return r.value
}

// Chain operations
func (r Result[T]) Map(fn func(T) T) Result[T] {
    if r.err != nil {
        return r
    }
    return Success(fn(r.value))
}

// Usage
func Divide(a, b float64) Result[float64] {
    if b == 0 {
        return Failure[float64](errors.New("division by zero"))
    }
    return Success(a / b)
}

result := Divide(10, 2).
    Map(func(v float64) float64 { return v * 2 }).
    Map(func(v float64) float64 { return v + 1 })

if result.IsSuccess() {
    fmt.Println(result.Must()) // 11
}
```

### 6. Worker Pool Pattern
**Purpose**: Manage concurrent workers efficiently

```go
// ✅ Good: Worker pool with proper lifecycle management
type Job func() error

type WorkerPool struct {
    workers   int
    jobs      chan Job
    results   chan error
    wg        sync.WaitGroup
}

func NewWorkerPool(workers int) *WorkerPool {
    return &WorkerPool{
        workers: workers,
        jobs:    make(chan Job, workers*2),
        results: make(chan error, workers*2),
    }
}

func (p *WorkerPool) Start(ctx context.Context) {
    for i := 0; i < p.workers; i++ {
        p.wg.Add(1)
        go p.worker(ctx)
    }
}

func (p *WorkerPool) worker(ctx context.Context) {
    defer p.wg.Done()
    
    for {
        select {
        case <-ctx.Done():
            return
        case job, ok := <-p.jobs:
            if !ok {
                return
            }
            p.results <- job()
        }
    }
}

func (p *WorkerPool) Submit(job Job) {
    p.jobs <- job
}

func (p *WorkerPool) Stop() {
    close(p.jobs)
    p.wg.Wait()
    close(p.results)
}

// Usage
ctx := context.Background()
pool := NewWorkerPool(5)
pool.Start(ctx)

// Submit jobs
for i := 0; i < 100; i++ {
    i := i // Capture loop variable
    pool.Submit(func() error {
        return processItem(i)
    })
}

// Process results
go func() {
    for err := range pool.results {
        if err != nil {
            log.Printf("Job failed: %v", err)
        }
    }
}()

pool.Stop()
```

### 7. Context Values Pattern
**Purpose**: Pass request-scoped values through call chains

```go
// ✅ Good: Type-safe context keys
type contextKey string

const (
    userIDKey     contextKey = "userID"
    requestIDKey  contextKey = "requestID"
)

// Helper functions for type safety
func WithUserID(ctx context.Context, userID string) context.Context {
    return context.WithValue(ctx, userIDKey, userID)
}

func GetUserID(ctx context.Context) (string, bool) {
    userID, ok := ctx.Value(userIDKey).(string)
    return userID, ok
}

func WithRequestID(ctx context.Context, requestID string) context.Context {
    return context.WithValue(ctx, requestIDKey, requestID)
}

func GetRequestID(ctx context.Context) (string, bool) {
    requestID, ok := ctx.Value(requestIDKey).(string)
    return requestID, ok
}

// Usage in middleware
func AuthMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        userID := extractUserIDFromToken(r)
        if userID == "" {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }
        
        ctx := WithUserID(r.Context(), userID)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

### 8. Retry Pattern with Backoff
**Purpose**: Handle transient failures gracefully

```go
// ✅ Good: Configurable retry with exponential backoff
type RetryConfig struct {
    MaxAttempts int
    InitialDelay time.Duration
    MaxDelay     time.Duration
    Multiplier   float64
}

func Retry(ctx context.Context, config RetryConfig, fn func() error) error {
    var lastErr error
    delay := config.InitialDelay
    
    for attempt := 0; attempt < config.MaxAttempts; attempt++ {
        if attempt > 0 {
            select {
            case <-ctx.Done():
                return ctx.Err()
            case <-time.After(delay):
            }
            
            // Exponential backoff
            delay = time.Duration(float64(delay) * config.Multiplier)
            if delay > config.MaxDelay {
                delay = config.MaxDelay
            }
        }
        
        lastErr = fn()
        if lastErr == nil {
            return nil
        }
        
        // Check if error is retryable
        var tempErr interface{ Temporary() bool }
        if errors.As(lastErr, &tempErr) && !tempErr.Temporary() {
            return lastErr // Don't retry permanent errors
        }
    }
    
    return fmt.Errorf("max retries exceeded: %w", lastErr)
}

// Usage
config := RetryConfig{
    MaxAttempts:  5,
    InitialDelay: 100 * time.Millisecond,
    MaxDelay:     5 * time.Second,
    Multiplier:   2.0,
}

err := Retry(ctx, config, func() error {
    return callFlakeyService()
})
```

## Anti-patterns to Avoid

### 1. ❌ Naked Returns
**Problem**: Reduces readability and can lead to bugs

```go
// ❌ Bad: Naked return
func calculate(x, y int) (result int, err error) {
    if y == 0 {
        err = errors.New("division by zero")
        return // Which values are returned?
    }
    result = x / y
    return // Unclear what's being returned
}

// ✅ Good: Explicit returns
func calculate(x, y int) (int, error) {
    if y == 0 {
        return 0, errors.New("division by zero")
    }
    return x / y, nil
}
```

### 2. ❌ Interface Pollution
**Problem**: Creating unnecessary abstractions

```go
// ❌ Bad: Premature abstraction
type UserInterface interface {
    GetUser(id string) (*User, error)
    CreateUser(user *User) error
    UpdateUser(user *User) error
    DeleteUser(id string) error
}

type UserService struct {
    db *sql.DB
}

// Only one implementation exists!

// ✅ Good: Start concrete, extract interface when needed
type UserService struct {
    db *sql.DB
}

func (s *UserService) GetUser(id string) (*User, error) {
    // Implementation
}

// Extract interface only when you have multiple implementations
// or need to mock for testing
```

### 3. ❌ Goroutine Leaks
**Problem**: Goroutines that never terminate

```go
// ❌ Bad: Goroutine leak
func watchValues(values <-chan int) {
    go func() {
        for v := range values {
            fmt.Println(v)
        }
        // If channel is never closed, goroutine runs forever
    }()
}

// ❌ Bad: No way to stop goroutine
func startWorker() {
    go func() {
        for {
            doWork()
            time.Sleep(1 * time.Second)
        }
    }()
}

// ✅ Good: Goroutine with proper lifecycle
func watchValues(ctx context.Context, values <-chan int) {
    go func() {
        for {
            select {
            case <-ctx.Done():
                return
            case v, ok := <-values:
                if !ok {
                    return
                }
                fmt.Println(v)
            }
        }
    }()
}
```

### 4. ❌ Empty Interface Overuse
**Problem**: Loss of type safety and unclear APIs

```go
// ❌ Bad: interface{} everywhere
func Process(data interface{}) interface{} {
    // Type assertions everywhere
    switch v := data.(type) {
    case string:
        return len(v)
    case int:
        return v * 2
    default:
        return nil
    }
}

// ✅ Good: Use specific types or generics
func ProcessString(s string) int {
    return len(s)
}

func ProcessInt(n int) int {
    return n * 2
}

// Or with generics (Go 1.18+)
func Map[T, U any](slice []T, fn func(T) U) []U {
    result := make([]U, len(slice))
    for i, v := range slice {
        result[i] = fn(v)
    }
    return result
}
```

### 5. ❌ Ignoring Errors
**Problem**: Silent failures and hard-to-debug issues

```go
// ❌ Bad: Ignoring errors
func saveData(data []byte) {
    os.WriteFile("data.txt", data, 0644) // Error ignored!
}

func processFile(filename string) string {
    data, _ := os.ReadFile(filename) // What if file doesn't exist?
    return string(data)
}

// ✅ Good: Always handle errors
func saveData(data []byte) error {
    if err := os.WriteFile("data.txt", data, 0644); err != nil {
        return fmt.Errorf("failed to save data: %w", err)
    }
    return nil
}

func processFile(filename string) (string, error) {
    data, err := os.ReadFile(filename)
    if err != nil {
        return "", fmt.Errorf("failed to read file %s: %w", filename, err)
    }
    return string(data), nil
}
```

### 6. ❌ Mutex Copying
**Problem**: Breaks mutex functionality

```go
// ❌ Bad: Copying mutex
type Counter struct {
    mu    sync.Mutex
    count int
}

func (c Counter) Increment() { // Value receiver copies mutex!
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count++
}

// ❌ Bad: Copying struct with mutex
func process(c Counter) { // Copies the mutex
    c.Increment()
}

// ✅ Good: Use pointer receivers
type Counter struct {
    mu    sync.Mutex
    count int
}

func (c *Counter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count++
}

func process(c *Counter) { // Pass pointer
    c.Increment()
}
```

### 7. ❌ Context in Structs
**Problem**: Contexts should be passed as parameters

```go
// ❌ Bad: Storing context in struct
type Service struct {
    ctx context.Context // Don't do this!
    db  *sql.DB
}

func (s *Service) GetUser(id string) (*User, error) {
    // Using stored context - bad practice
    return s.db.QueryContext(s.ctx, "SELECT ...")
}

// ✅ Good: Pass context as parameter
type Service struct {
    db *sql.DB
}

func (s *Service) GetUser(ctx context.Context, id string) (*User, error) {
    return s.db.QueryContext(ctx, "SELECT ...")
}
```

### 8. ❌ Init Function Overuse
**Problem**: Makes testing difficult and creates hidden dependencies

```go
// ❌ Bad: Complex init with side effects
var db *sql.DB

func init() {
    var err error
    db, err = sql.Open("postgres", os.Getenv("DATABASE_URL"))
    if err != nil {
        panic(err) // Panic in init!
    }
}

// ✅ Good: Explicit initialization
type App struct {
    db *sql.DB
}

func NewApp(databaseURL string) (*App, error) {
    db, err := sql.Open("postgres", databaseURL)
    if err != nil {
        return nil, fmt.Errorf("failed to connect to database: %w", err)
    }
    
    return &App{db: db}, nil
}
```

### 9. ❌ Panic in Libraries
**Problem**: Forces error handling on callers

```go
// ❌ Bad: Panic in library code
func ParseConfig(data []byte) *Config {
    var cfg Config
    if err := json.Unmarshal(data, &cfg); err != nil {
        panic(fmt.Sprintf("invalid config: %v", err))
    }
    return &cfg
}

// ✅ Good: Return errors
func ParseConfig(data []byte) (*Config, error) {
    var cfg Config
    if err := json.Unmarshal(data, &cfg); err != nil {
        return nil, fmt.Errorf("invalid config: %w", err)
    }
    return &cfg, nil
}
```

### 10. ❌ Globals and Singletons
**Problem**: Hidden dependencies and testing difficulties

```go
// ❌ Bad: Global variables
var (
    globalDB     *sql.DB
    globalConfig *Config
    globalLogger *Logger
)

func GetUser(id string) (*User, error) {
    // Hidden dependency on globalDB
    return globalDB.Query(...)
}

// ✅ Good: Dependency injection
type UserService struct {
    db     *sql.DB
    logger *Logger
}

func NewUserService(db *sql.DB, logger *Logger) *UserService {
    return &UserService{
        db:     db,
        logger: logger,
    }
}

func (s *UserService) GetUser(id string) (*User, error) {
    // Explicit dependencies
    return s.db.Query(...)
}
```

## Pattern vs Anti-pattern Comparisons

### Error Handling

```go
// ❌ Anti-pattern: Error strings comparison
if err.Error() == "not found" {
    // Fragile: breaks if error message changes
}

// ✅ Pattern: Sentinel errors
var ErrNotFound = errors.New("not found")

if errors.Is(err, ErrNotFound) {
    // Robust error checking
}
```

### Concurrency

```go
// ❌ Anti-pattern: Shared memory without synchronization
var counter int

func increment() {
    counter++ // Race condition!
}

// ✅ Pattern: Use channels or mutexes
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

### Configuration

```go
// ❌ Anti-pattern: Many boolean parameters
func NewServer(host string, port int, useTLS bool, 
    enableLogging bool, enableMetrics bool) *Server {
    // Hard to read at call site
}

// Usage unclear:
srv := NewServer("localhost", 8080, true, false, true)

// ✅ Pattern: Functional options
srv := NewServer(
    WithHost("localhost"),
    WithPort(8080),
    WithTLS(true),
    WithMetrics(true),
)
```

### Testing

```go
// ❌ Anti-pattern: Hard-coded dependencies
func ProcessOrder(orderID string) error {
    db := getGlobalDB() // Hard to test
    order := db.GetOrder(orderID)
    return order.Process()
}

// ✅ Pattern: Dependency injection
type OrderProcessor struct {
    db OrderRepository
}

func (p *OrderProcessor) ProcessOrder(orderID string) error {
    order := p.db.GetOrder(orderID)
    return order.Process()
}
```

## Summary

### Key Patterns to Embrace
1. **Functional Options** - Flexible configuration
2. **Error Wrapping** - Context-rich errors
3. **Small Interfaces** - Better abstractions
4. **Constructor Functions** - Safe initialization
5. **Context Values** - Request-scoped data
6. **Worker Pools** - Efficient concurrency

### Anti-patterns to Avoid
1. **Interface Pollution** - Don't create unnecessary abstractions
2. **Goroutine Leaks** - Always provide termination mechanisms
3. **Error Ignoring** - Handle all errors explicitly
4. **Global State** - Use dependency injection
5. **Empty Interfaces** - Maintain type safety
6. **Complex Init** - Prefer explicit initialization

### Remember
- Start simple, add complexity only when needed
- Make the zero value useful when possible
- Errors are values - handle them explicitly
- Composition over inheritance
- Clear is better than clever