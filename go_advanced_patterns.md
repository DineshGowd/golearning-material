# Go Patterns & Anti-Patterns

## ✅ Go Patterns

### Use context.Context for request lifecycle
```go
func handler(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()
	// use ctx to cancel, timeout etc.
}
```
**Why?** It allows request-scoped deadlines, cancellations, and passing request-specific values.

### Use error handling explicitly
```go
if err := doSomething(); err != nil {
	log.Printf("Error: %v", err)
}
```
**Why?** Explicit error handling makes Go code predictable and maintainable.

### Close resources with defer
```go
f, err := os.Open("file.txt")
if err != nil {
	log.Fatal(err)
}
defer f.Close()
```
**Why?** Ensures that resources are released no matter how the function exits.

### Keep function signatures small and clean
```go
func process(userID int, data string) error
```
**Why?** Small signatures are easier to read, test, and maintain.

### Use `go` keyword only when you can manage the goroutine
```go
go func() {
	doWork()
}()
```
**Why?** Always pair goroutines with cancellation or WaitGroup to avoid leaks.


## ❌ Go Anti-Patterns

### Ignoring errors
```go
_ = doSomething()
```
**Why it's bad:** Ignoring errors hides failures and causes silent bugs.

### Closing channels from receiver side
```go
for v := range ch {
	if done {
		close(ch) // ❌ Wrong
	}
}
```
**Why it's bad:** Only the sender should close the channel.

### Exposing internal struct fields directly
```go
type User struct {
	Name string
	Password string
}
```
**Why it's bad:** Breaks encapsulation. Use methods or exported fields carefully.

### Using goroutines without control
```go
go longRunningTask() // ❌ No context, no sync
```
**Why it's bad:** Leads to memory leaks or runaway concurrency.

### Putting business logic in HTTP handlers
```go
func handler(w http.ResponseWriter, r *http.Request) {
	doEverythingHere()
}
```
**Why it's bad:** Handlers should delegate to services. Keeps code modular and testable.


## 🚀 Advanced Patterns

### 1. Middleware Pattern using `http.Handler`
```go
func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        log.Printf("Started %s %s", r.Method, r.URL.Path)
        next.ServeHTTP(w, r)
        log.Printf("Completed %s", r.URL.Path)
    })
}
```
**Why?** Adds reusable logic (like logging, auth) without cluttering handlers.

### 2. Context-Aware Logger
```go
func logWithContext(ctx context.Context, msg string) {
    requestID, _ := ctx.Value("requestID").(string)
    log.Printf("[%s] %s", requestID, msg)
}
```
**Why?** Enables per-request tracing and diagnostics.

### 3. Use `select` with channels for concurrency
```go
select {
case msg := <-ch:
    fmt.Println("Received:", msg)
case <-time.After(2 * time.Second):
    fmt.Println("Timeout")
}
```
**Why?** Handles multiple channel operations safely and responsively.

### 4. REST API Structure Example
```go
// handler.go
func (s *Server) GetUser(w http.ResponseWriter, r *http.Request) {
    userID := chi.URLParam(r, "id")
    user, err := s.store.GetUser(userID)
    if err != nil {
        http.Error(w, "User not found", http.StatusNotFound)
        return
    }
    json.NewEncoder(w).Encode(user)
}
```
**Why?** Keeps routing, business logic, and storage separated.

### 5. Controlled Goroutines with sync.WaitGroup
```go
var wg sync.WaitGroup
wg.Add(1)
go func() {
    defer wg.Done()
    doWork()
}()
wg.Wait()
```
**Why?** Ensures all goroutines complete before program exits.
