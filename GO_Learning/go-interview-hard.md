# Go Interview - Hard Level

## 1. Advanced Concurrency

### Q: Explain the Go memory model
**A:** The Go memory model defines the conditions under which reads of a variable in one goroutine can be guaranteed to observe values produced by writes to the same variable in a different goroutine.

```go
// Happens-before relationships
var a, b int

// Goroutine 1
go func() {
    a = 1    // (1)
    b = 2    // (2)
}()

// Goroutine 2
go func() {
    if b == 2 {  // (3)
        fmt.Println(a)  // (4) - Not guaranteed to see a = 1!
    }
}()

// Fix with proper synchronization
var a, b int
var mu sync.Mutex

// Goroutine 1
go func() {
    mu.Lock()
    a = 1
    b = 2
    mu.Unlock()
}()

// Goroutine 2
go func() {
    mu.Lock()
    if b == 2 {
        fmt.Println(a)  // Now guaranteed to see a = 1
    }
    mu.Unlock()
}()
```

### Q: How do you detect and fix race conditions?
**A:**
```go
// Race condition example
type Counter struct {
    value int
}

func (c *Counter) Increment() {
    c.value++ // Race condition!
}

// Detection: go run -race main.go
// Or: go test -race ./...

// Fix 1: Mutex
type SafeCounter struct {
    mu    sync.Mutex
    value int
}

func (c *SafeCounter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.value++
}

// Fix 2: Atomic operations
type AtomicCounter struct {
    value int64
}

func (c *AtomicCounter) Increment() {
    atomic.AddInt64(&c.value, 1)
}

// Fix 3: Channels
type ChannelCounter struct {
    ch chan int
}

func (c *ChannelCounter) Increment() {
    c.ch <- 1
}
```

### Q: Implement a semaphore using channels
**A:**
```go
type Semaphore struct {
    ch chan struct{}
}

func NewSemaphore(capacity int) *Semaphore {
    return &Semaphore{
        ch: make(chan struct{}, capacity),
    }
}

func (s *Semaphore) Acquire(ctx context.Context) error {
    select {
    case s.ch <- struct{}{}:
        return nil
    case <-ctx.Done():
        return ctx.Err()
    }
}

func (s *Semaphore) Release() {
    <-s.ch
}

// Weighted semaphore
type WeightedSemaphore struct {
    ch chan int
    max int
}

func NewWeightedSemaphore(max int) *WeightedSemaphore {
    ch := make(chan int, 1)
    ch <- max
    return &WeightedSemaphore{ch: ch, max: max}
}

func (s *WeightedSemaphore) Acquire(ctx context.Context, weight int) error {
    select {
    case available := <-s.ch:
        if available >= weight {
            s.ch <- available - weight
            return nil
        }
        s.ch <- available
        return errors.New("insufficient capacity")
    case <-ctx.Done():
        return ctx.Err()
    }
}
```

## 2. Context Deep Dive

### Q: How would you implement a distributed tracing system?
**A:**
```go
type TraceID string
type SpanID string

type Span struct {
    TraceID  TraceID
    SpanID   SpanID
    ParentID SpanID
    Name     string
    Start    time.Time
    End      time.Time
    Tags     map[string]string
}

type traceKey struct{}

func WithTrace(ctx context.Context, traceID TraceID) context.Context {
    return context.WithValue(ctx, traceKey{}, traceID)
}

func GetTraceID(ctx context.Context) (TraceID, bool) {
    traceID, ok := ctx.Value(traceKey{}).(TraceID)
    return traceID, ok
}

// Middleware for HTTP
func TracingMiddleware(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        traceID := r.Header.Get("X-Trace-ID")
        if traceID == "" {
            traceID = generateTraceID()
        }
        
        ctx := WithTrace(r.Context(), TraceID(traceID))
        span := &Span{
            TraceID: TraceID(traceID),
            SpanID:  generateSpanID(),
            Name:    r.URL.Path,
            Start:   time.Now(),
        }
        
        // Store span in context
        ctx = context.WithValue(ctx, "span", span)
        
        // Continue with traced context
        next.ServeHTTP(w, r.WithContext(ctx))
        
        // Complete span
        span.End = time.Now()
        // Send to tracing backend
    }
}
```

### Q: Implement context with custom cancellation logic
**A:**
```go
type customContext struct {
    context.Context
    cancelFunc func()
    mu         sync.Mutex
    err        error
}

func WithCustomCancel(parent context.Context) (context.Context, context.CancelFunc) {
    ctx := &customContext{
        Context: parent,
    }
    
    // Propagate parent cancellation
    go func() {
        <-parent.Done()
        ctx.cancel(parent.Err())
    }()
    
    return ctx, ctx.cancel
}

func (c *customContext) Done() <-chan struct{} {
    ch := make(chan struct{})
    go func() {
        <-c.Context.Done()
        close(ch)
    }()
    return ch
}

func (c *customContext) Err() error {
    c.mu.Lock()
    defer c.mu.Unlock()
    if c.err != nil {
        return c.err
    }
    return c.Context.Err()
}

func (c *customContext) cancel(err error) {
    c.mu.Lock()
    defer c.mu.Unlock()
    if c.err != nil {
        return // Already cancelled
    }
    c.err = err
    if c.cancelFunc != nil {
        c.cancelFunc()
    }
}
```

## 3. Reflection and Code Generation

### Q: Build a JSON validator using reflection
**A:**
```go
type Validator struct {
    rules map[string]Rule
}

type Rule interface {
    Validate(reflect.Value) error
}

type RequiredRule struct{}

func (r RequiredRule) Validate(v reflect.Value) error {
    if v.IsZero() {
        return errors.New("field is required")
    }
    return nil
}

type MinRule struct {
    Min int
}

func (r MinRule) Validate(v reflect.Value) error {
    switch v.Kind() {
    case reflect.Int, reflect.Int64:
        if v.Int() < int64(r.Min) {
            return fmt.Errorf("value must be >= %d", r.Min)
        }
    case reflect.String:
        if len(v.String()) < r.Min {
            return fmt.Errorf("length must be >= %d", r.Min)
        }
    }
    return nil
}

func Validate(v interface{}) error {
    val := reflect.ValueOf(v)
    if val.Kind() == reflect.Ptr {
        val = val.Elem()
    }
    
    typ := val.Type()
    for i := 0; i < val.NumField(); i++ {
        field := typ.Field(i)
        fieldVal := val.Field(i)
        
        // Parse tags
        if tag := field.Tag.Get("validate"); tag != "" {
            rules := parseRules(tag)
            for _, rule := range rules {
                if err := rule.Validate(fieldVal); err != nil {
                    return fmt.Errorf("field %s: %w", field.Name, err)
                }
            }
        }
    }
    return nil
}

// Usage
type User struct {
    Name  string `validate:"required,min=3"`
    Age   int    `validate:"required,min=0"`
    Email string `validate:"required,email"`
}
```

### Q: Implement a dependency injection container
**A:**
```go
type Container struct {
    mu        sync.RWMutex
    providers map[reflect.Type]Provider
    instances map[reflect.Type]interface{}
}

type Provider func(c *Container) (interface{}, error)

func NewContainer() *Container {
    return &Container{
        providers: make(map[reflect.Type]Provider),
        instances: make(map[reflect.Type]interface{}),
    }
}

func (c *Container) Register(typ reflect.Type, provider Provider) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.providers[typ] = provider
}

func (c *Container) Get(typ reflect.Type) (interface{}, error) {
    c.mu.RLock()
    if instance, ok := c.instances[typ]; ok {
        c.mu.RUnlock()
        return instance, nil
    }
    c.mu.RUnlock()
    
    c.mu.Lock()
    defer c.mu.Unlock()
    
    // Double-check after acquiring write lock
    if instance, ok := c.instances[typ]; ok {
        return instance, nil
    }
    
    provider, ok := c.providers[typ]
    if !ok {
        return nil, fmt.Errorf("no provider for type %v", typ)
    }
    
    instance, err := provider(c)
    if err != nil {
        return nil, err
    }
    
    c.instances[typ] = instance
    return instance, nil
}

// Automatic injection
func (c *Container) Inject(target interface{}) error {
    val := reflect.ValueOf(target)
    if val.Kind() != reflect.Ptr {
        return errors.New("target must be a pointer")
    }
    
    val = val.Elem()
    typ := val.Type()
    
    for i := 0; i < val.NumField(); i++ {
        field := val.Field(i)
        fieldType := typ.Field(i)
        
        if tag := fieldType.Tag.Get("inject"); tag != "" {
            instance, err := c.Get(field.Type())
            if err != nil {
                return err
            }
            field.Set(reflect.ValueOf(instance))
        }
    }
    
    return nil
}
```

## 4. Performance Optimization

### Q: How do you optimize memory allocations?
**A:**
```go
// 1. Pre-allocate slices
// Bad
var result []int
for i := 0; i < n; i++ {
    result = append(result, i) // Multiple allocations
}

// Good
result := make([]int, 0, n) // Single allocation
for i := 0; i < n; i++ {
    result = append(result, i)
}

// 2. Use sync.Pool for temporary objects
var bufferPool = sync.Pool{
    New: func() interface{} {
        return make([]byte, 0, 1024)
    },
}

func process(data []byte) {
    buf := bufferPool.Get().([]byte)
    buf = buf[:0] // Reset length
    defer bufferPool.Put(buf)
    
    // Use buffer
    buf = append(buf, data...)
}

// 3. Avoid string concatenation in loops
// Bad
var s string
for _, word := range words {
    s += word // Creates new string each time
}

// Good
var b strings.Builder
b.Grow(calculateSize(words)) // Pre-allocate
for _, word := range words {
    b.WriteString(word)
}
s := b.String()

// 4. Reuse objects
type Node struct {
    Value int
    Next  *Node
}

var nodePool = sync.Pool{
    New: func() interface{} {
        return &Node{}
    },
}

func newNode(value int) *Node {
    n := nodePool.Get().(*Node)
    n.Value = value
    n.Next = nil
    return n
}

func freeNode(n *Node) {
    *n = Node{} // Clear
    nodePool.Put(n)
}
```

### Q: Profile and optimize a CPU-intensive function
**A:**
```go
// Original slow function
func slowFunction(data []int) int {
    result := 0
    for i := 0; i < len(data); i++ {
        for j := 0; j < len(data); j++ {
            result += data[i] * data[j]
        }
    }
    return result
}

// Step 1: Profile
// go test -cpuprofile=cpu.prof -bench=.
// go tool pprof cpu.prof

// Step 2: Identify hotspots and optimize
func optimizedFunction(data []int) int {
    n := len(data)
    sum := 0
    for i := 0; i < n; i++ {
        sum += data[i]
    }
    
    // Mathematical optimization: (Σxi)² = Σxi² + 2Σ(xi*xj) where i<j
    result := sum * sum
    return result
}

// Step 3: Parallel processing for large datasets
func parallelFunction(data []int) int {
    n := len(data)
    numWorkers := runtime.NumCPU()
    chunkSize := (n + numWorkers - 1) / numWorkers
    
    results := make(chan int, numWorkers)
    var wg sync.WaitGroup
    
    for i := 0; i < numWorkers; i++ {
        start := i * chunkSize
        end := start + chunkSize
        if end > n {
            end = n
        }
        
        wg.Add(1)
        go func(chunk []int) {
            defer wg.Done()
            partial := 0
            for _, v := range chunk {
                partial += v
            }
            results <- partial
        }(data[start:end])
    }
    
    go func() {
        wg.Wait()
        close(results)
    }()
    
    total := 0
    for partial := range results {
        total += partial
    }
    
    return total * total
}
```

## 5. Advanced Generics

### Q: Implement a type-safe event bus using generics
**A:**
```go
type Event interface {
    EventType() string
}

type EventHandler[T Event] func(T)

type EventBus struct {
    mu       sync.RWMutex
    handlers map[string][]interface{}
}

func NewEventBus() *EventBus {
    return &EventBus{
        handlers: make(map[string][]interface{}),
    }
}

func Subscribe[T Event](bus *EventBus, handler EventHandler[T]) func() {
    var zero T
    eventType := zero.EventType()
    
    bus.mu.Lock()
    bus.handlers[eventType] = append(bus.handlers[eventType], handler)
    bus.mu.Unlock()
    
    // Return unsubscribe function
    return func() {
        bus.mu.Lock()
        defer bus.mu.Unlock()
        
        handlers := bus.handlers[eventType]
        for i, h := range handlers {
            if reflect.ValueOf(h).Pointer() == reflect.ValueOf(handler).Pointer() {
                bus.handlers[eventType] = append(handlers[:i], handlers[i+1:]...)
                break
            }
        }
    }
}

func Publish[T Event](bus *EventBus, event T) {
    eventType := event.EventType()
    
    bus.mu.RLock()
    handlers := bus.handlers[eventType]
    bus.mu.RUnlock()
    
    for _, h := range handlers {
        if handler, ok := h.(EventHandler[T]); ok {
            go handler(event)
        }
    }
}

// Usage
type UserCreatedEvent struct {
    UserID string
}

func (e UserCreatedEvent) EventType() string {
    return "user.created"
}

bus := NewEventBus()
unsubscribe := Subscribe(bus, func(e UserCreatedEvent) {
    fmt.Printf("User created: %s\n", e.UserID)
})
defer unsubscribe()

Publish(bus, UserCreatedEvent{UserID: "123"})
```

### Q: Build a generic data pipeline
**A:**
```go
type Pipeline[T any] struct {
    stages []Stage[T]
}

type Stage[T any] func(context.Context, <-chan T) <-chan T

func NewPipeline[T any](stages ...Stage[T]) *Pipeline[T] {
    return &Pipeline[T]{stages: stages}
}

func (p *Pipeline[T]) Run(ctx context.Context, input <-chan T) <-chan T {
    output := input
    for _, stage := range p.stages {
        output = stage(ctx, output)
    }
    return output
}

// Generic stage builders
func Map[T, U any](fn func(T) U) Stage[T] {
    return func(ctx context.Context, in <-chan T) <-chan T {
        out := make(chan T)
        go func() {
            defer close(out)
            for {
                select {
                case <-ctx.Done():
                    return
                case val, ok := <-in:
                    if !ok {
                        return
                    }
                    // Note: This would need adjustment for T->U conversion
                    out <- val
                }
            }
        }()
        return out
    }
}

func Filter[T any](predicate func(T) bool) Stage[T] {
    return func(ctx context.Context, in <-chan T) <-chan T {
        out := make(chan T)
        go func() {
            defer close(out)
            for {
                select {
                case <-ctx.Done():
                    return
                case val, ok := <-in:
                    if !ok {
                        return
                    }
                    if predicate(val) {
                        out <- val
                    }
                }
            }
        }()
        return out
    }
}

func Batch[T any](size int) Stage[T] {
    return func(ctx context.Context, in <-chan T) <-chan T {
        out := make(chan T)
        go func() {
            defer close(out)
            batch := make([]T, 0, size)
            
            for {
                select {
                case <-ctx.Done():
                    return
                case val, ok := <-in:
                    if !ok {
                        // Flush remaining
                        for _, v := range batch {
                            out <- v
                        }
                        return
                    }
                    batch = append(batch, val)
                    if len(batch) >= size {
                        for _, v := range batch {
                            out <- v
                        }
                        batch = batch[:0]
                    }
                }
            }
        }()
        return out
    }
}
```

## 6. System Design Questions

### Q: Design a distributed rate limiter
**A:**
```go
// Token bucket with Redis backend
type DistributedRateLimiter struct {
    redis    *redis.Client
    rate     int
    interval time.Duration
}

func (rl *DistributedRateLimiter) Allow(ctx context.Context, key string) (bool, error) {
    now := time.Now()
    windowStart := now.Truncate(rl.interval)
    
    pipe := rl.redis.Pipeline()
    
    // Increment counter
    incr := pipe.Incr(ctx, fmt.Sprintf("rl:%s:%d", key, windowStart.Unix()))
    pipe.Expire(ctx, fmt.Sprintf("rl:%s:%d", key, windowStart.Unix()), 2*rl.interval)
    
    _, err := pipe.Exec(ctx)
    if err != nil {
        return false, err
    }
    
    count := incr.Val()
    return count <= int64(rl.rate), nil
}

// Sliding window log
type SlidingWindowLimiter struct {
    redis    *redis.Client
    limit    int
    window   time.Duration
}

func (swl *SlidingWindowLimiter) Allow(ctx context.Context, key string) (bool, error) {
    now := time.Now()
    windowStart := now.Add(-swl.window)
    
    pipe := swl.redis.Pipeline()
    
    // Remove old entries
    pipe.ZRemRangeByScore(ctx, key, "0", strconv.FormatInt(windowStart.UnixNano(), 10))
    
    // Add current request
    pipe.ZAdd(ctx, key, &redis.Z{
        Score:  float64(now.UnixNano()),
        Member: now.UnixNano(),
    })
    
    // Count requests in window
    count := pipe.ZCard(ctx, key)
    
    // Set expiry
    pipe.Expire(ctx, key, swl.window)
    
    _, err := pipe.Exec(ctx)
    if err != nil {
        return false, err
    }
    
    return count.Val() <= int64(swl.limit), nil
}
```

### Q: Implement a circuit breaker
**A:**
```go
type State int

const (
    StateClosed State = iota
    StateOpen
    StateHalfOpen
)

type CircuitBreaker struct {
    mu              sync.RWMutex
    state           State
    failures        int
    successes       int
    lastFailureTime time.Time
    
    maxFailures      int
    resetTimeout     time.Duration
    halfOpenRequests int
}

func NewCircuitBreaker(maxFailures int, resetTimeout time.Duration) *CircuitBreaker {
    return &CircuitBreaker{
        state:            StateClosed,
        maxFailures:      maxFailures,
        resetTimeout:     resetTimeout,
        halfOpenRequests: 3,
    }
}

func (cb *CircuitBreaker) Execute(fn func() error) error {
    if !cb.canExecute() {
        return errors.New("circuit breaker is open")
    }
    
    err := fn()
    cb.recordResult(err)
    return err
}

func (cb *CircuitBreaker) canExecute() bool {
    cb.mu.RLock()
    defer cb.mu.RUnlock()
    
    switch cb.state {
    case StateClosed:
        return true
    case StateOpen:
        if time.Since(cb.lastFailureTime) > cb.resetTimeout {
            cb.mu.RUnlock()
            cb.mu.Lock()
            cb.state = StateHalfOpen
            cb.successes = 0
            cb.mu.Unlock()
            cb.mu.RLock()
            return true
        }
        return false
    case StateHalfOpen:
        return cb.successes < cb.halfOpenRequests
    }
    return false
}

func (cb *CircuitBreaker) recordResult(err error) {
    cb.mu.Lock()
    defer cb.mu.Unlock()
    
    if err != nil {
        cb.failures++
        cb.lastFailureTime = time.Now()
        
        if cb.state == StateHalfOpen || cb.failures >= cb.maxFailures {
            cb.state = StateOpen
        }
    } else {
        if cb.state == StateHalfOpen {
            cb.successes++
            if cb.successes >= cb.halfOpenRequests {
                cb.state = StateClosed
                cb.failures = 0
            }
        } else if cb.state == StateClosed {
            cb.failures = 0
        }
    }
}
```

## 7. Complex Coding Problems

### Q: Implement a thread-safe LFU cache
**A:**
```go
type LFUCache struct {
    capacity int
    minFreq  int
    
    cache    map[string]*lfuNode
    freqMap  map[int]*list.List
    mu       sync.Mutex
}

type lfuNode struct {
    key      string
    value    interface{}
    freq     int
    listElem *list.Element
}

func NewLFUCache(capacity int) *LFUCache {
    return &LFUCache{
        capacity: capacity,
        cache:    make(map[string]*lfuNode),
        freqMap:  make(map[int]*list.List),
    }
}

func (c *LFUCache) Get(key string) (interface{}, bool) {
    c.mu.Lock()
    defer c.mu.Unlock()
    
    node, exists := c.cache[key]
    if !exists {
        return nil, false
    }
    
    c.updateFrequency(node)
    return node.value, true
}

func (c *LFUCache) Put(key string, value interface{}) {
    c.mu.Lock()
    defer c.mu.Unlock()
    
    if c.capacity == 0 {
        return
    }
    
    if node, exists := c.cache[key]; exists {
        node.value = value
        c.updateFrequency(node)
        return
    }
    
    if len(c.cache) >= c.capacity {
        c.evictLFU()
    }
    
    node := &lfuNode{
        key:   key,
        value: value,
        freq:  1,
    }
    
    c.cache[key] = node
    if c.freqMap[1] == nil {
        c.freqMap[1] = list.New()
    }
    node.listElem = c.freqMap[1].PushBack(key)
    c.minFreq = 1
}

func (c *LFUCache) updateFrequency(node *lfuNode) {
    freq := node.freq
    c.freqMap[freq].Remove(node.listElem)
    
    if c.freqMap[freq].Len() == 0 {
        delete(c.freqMap, freq)
        if c.minFreq == freq {
            c.minFreq++
        }
    }
    
    node.freq++
    if c.freqMap[node.freq] == nil {
        c.freqMap[node.freq] = list.New()
    }
    node.listElem = c.freqMap[node.freq].PushBack(node.key)
}

func (c *LFUCache) evictLFU() {
    list := c.freqMap[c.minFreq]
    elem := list.Front()
    if elem != nil {
        list.Remove(elem)
        key := elem.Value.(string)
        delete(c.cache, key)
    }
}
```

### Q: Build a lock-free queue
**A:**
```go
type LockFreeQueue struct {
    head unsafe.Pointer
    tail unsafe.Pointer
}

type node struct {
    value interface{}
    next  unsafe.Pointer
}

func NewLockFreeQueue() *LockFreeQueue {
    n := &node{}
    return &LockFreeQueue{
        head: unsafe.Pointer(n),
        tail: unsafe.Pointer(n),
    }
}

func (q *LockFreeQueue) Enqueue(value interface{}) {
    n := &node{value: value}
    for {
        tail := (*node)(atomic.LoadPointer(&q.tail))
        next := (*node)(atomic.LoadPointer(&tail.next))
        
        if tail == (*node)(atomic.LoadPointer(&q.tail)) {
            if next == nil {
                if atomic.CompareAndSwapPointer(&tail.next, 
                    unsafe.Pointer(next), unsafe.Pointer(n)) {
                    atomic.CompareAndSwapPointer(&q.tail, 
                        unsafe.Pointer(tail), unsafe.Pointer(n))
                    break
                }
            } else {
                atomic.CompareAndSwapPointer(&q.tail, 
                    unsafe.Pointer(tail), unsafe.Pointer(next))
            }
        }
    }
}

func (q *LockFreeQueue) Dequeue() (interface{}, bool) {
    for {
        head := (*node)(atomic.LoadPointer(&q.head))
        tail := (*node)(atomic.LoadPointer(&q.tail))
        next := (*node)(atomic.LoadPointer(&head.next))
        
        if head == (*node)(atomic.LoadPointer(&q.head)) {
            if head == tail {
                if next == nil {
                    return nil, false
                }
                atomic.CompareAndSwapPointer(&q.tail, 
                    unsafe.Pointer(tail), unsafe.Pointer(next))
            } else {
                value := next.value
                if atomic.CompareAndSwapPointer(&q.head, 
                    unsafe.Pointer(head), unsafe.Pointer(next)) {
                    return value, true
                }
            }
        }
    }
}
```

## Best Practices for Hard Level

1. **Understand memory model** - Know happens-before relationships
2. **Profile before optimizing** - Use pprof, trace, and benchmarks
3. **Know when to use unsafe** - Last resort for performance
4. **Design for testability** - Interfaces, dependency injection
5. **Consider distributed systems** - CAP theorem, eventual consistency
6. **Master debugging tools** - Delve, race detector, pprof
7. **Understand GC behavior** - Allocation patterns, GC pressure
8. **Design for scale** - Horizontal scaling, sharding strategies

## Architecture & Design Patterns

1. **SOLID principles** apply to Go
2. **Composition over inheritance** - Go's philosophy
3. **Interface segregation** - Many small interfaces
4. **Dependency injection** - For testability
5. **Circuit breakers** - For resilience
6. **Bulkheads** - Isolate failures
7. **Event sourcing** - For audit trails
8. **CQRS** - Separate read/write models