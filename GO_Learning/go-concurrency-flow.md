# Go Concurrency Flow Patterns - Sequential Guide

## Table of Contents
1. [Flow Overview](#flow-overview)
2. [Level 1: Basic Goroutines](#level-1-basic-goroutines)
3. [Level 2: Channels for Communication](#level-2-channels-for-communication)
4. [Level 3: WaitGroup for Synchronization](#level-3-waitgroup-for-synchronization)
5. [Level 4: Context for Cancellation](#level-4-context-for-cancellation)
6. [Level 5: Complete Patterns](#level-5-complete-patterns)
7. [Common Flow Patterns](#common-flow-patterns)

## Flow Overview

```
Start → Goroutines → Channels → WaitGroup → Context → Complete Pattern
         ↓            ↓           ↓            ↓
      (Launch)    (Communicate) (Synchronize) (Control)
```

## Level 1: Basic Goroutines

### Flow: Main → Goroutine → Problem (no synchronization)

```go
// ❌ Problem: Goroutine might not complete
func main() {
    go func() {
        fmt.Println("Hello from goroutine")
        time.Sleep(1 * time.Second)
        fmt.Println("Goroutine done")
    }()
    
    fmt.Println("Main function ends")
    // Program exits immediately, goroutine incomplete!
}

// ✅ Basic Fix: Wait (but not ideal)
func main() {
    go func() {
        fmt.Println("Hello from goroutine")
        time.Sleep(100 * time.Millisecond)
        fmt.Println("Goroutine done")
    }()
    
    time.Sleep(200 * time.Millisecond) // Crude wait
    fmt.Println("Main function ends")
}
```

**Key Learning**: Goroutines need synchronization mechanisms

## Level 2: Channels for Communication

### Flow: Goroutine → Channel → Main

```go
// Pattern 1: Simple Channel Communication
func main() {
    // Step 1: Create channel
    done := make(chan bool)
    
    // Step 2: Launch goroutine
    go func() {
        fmt.Println("Working...")
        time.Sleep(100 * time.Millisecond)
        fmt.Println("Done working")
        
        // Step 3: Signal completion
        done <- true
    }()
    
    // Step 4: Wait for signal
    <-done
    fmt.Println("All work completed")
}

// Pattern 2: Returning Values via Channel
func main() {
    // Step 1: Create typed channel
    result := make(chan int)
    
    // Step 2: Launch computation
    go func() {
        sum := 0
        for i := 1; i <= 100; i++ {
            sum += i
        }
        // Step 3: Send result
        result <- sum
    }()
    
    // Step 4: Receive result
    fmt.Printf("Sum: %d\n", <-result)
}

// Pattern 3: Multiple Goroutines with Channels
func main() {
    // Step 1: Create channels
    numbers := make(chan int)
    results := make(chan int)
    
    // Step 2: Launch workers
    for i := 0; i < 3; i++ {
        go func(id int) {
            for num := range numbers {
                fmt.Printf("Worker %d processing %d\n", id, num)
                results <- num * num
            }
        }(i)
    }
    
    // Step 3: Send work
    go func() {
        for i := 1; i <= 5; i++ {
            numbers <- i
        }
        close(numbers)
    }()
    
    // Step 4: Collect results
    for i := 0; i < 5; i++ {
        fmt.Printf("Result: %d\n", <-results)
    }
}
```

**Key Learning**: Channels enable goroutine communication but don't handle multiple goroutine coordination well

## Level 3: WaitGroup for Synchronization

### Flow: Main → WaitGroup → Multiple Goroutines → Wait

```go
// Pattern 1: Basic WaitGroup Usage
func main() {
    var wg sync.WaitGroup
    
    // Step 1: Add to WaitGroup before launching goroutines
    for i := 0; i < 5; i++ {
        wg.Add(1) // Increment counter
        
        // Step 2: Launch goroutine
        go func(id int) {
            // Step 3: Ensure Done is called
            defer wg.Done() // Decrement counter
            
            fmt.Printf("Worker %d starting\n", id)
            time.Sleep(time.Duration(id) * 100 * time.Millisecond)
            fmt.Printf("Worker %d finished\n", id)
        }(i)
    }
    
    // Step 4: Wait for all goroutines
    wg.Wait()
    fmt.Println("All workers completed")
}

// Pattern 2: WaitGroup with Channels
func main() {
    var wg sync.WaitGroup
    results := make(chan int, 10)
    
    // Step 1: Launch workers
    for i := 1; i <= 5; i++ {
        wg.Add(1)
        go func(num int) {
            defer wg.Done()
            
            // Do work
            result := num * num
            results <- result
        }(i)
    }
    
    // Step 2: Close channel when all done
    go func() {
        wg.Wait()
        close(results)
    }()
    
    // Step 3: Read results
    for result := range results {
        fmt.Printf("Result: %d\n", result)
    }
}

// Pattern 3: Nested WaitGroups
func main() {
    var outerWg sync.WaitGroup
    
    for i := 0; i < 3; i++ {
        outerWg.Add(1)
        go func(groupID int) {
            defer outerWg.Done()
            
            var innerWg sync.WaitGroup
            for j := 0; j < 3; j++ {
                innerWg.Add(1)
                go func(workerID int) {
                    defer innerWg.Done()
                    
                    fmt.Printf("Group %d, Worker %d\n", groupID, workerID)
                    time.Sleep(100 * time.Millisecond)
                }(j)
            }
            
            innerWg.Wait()
            fmt.Printf("Group %d completed\n", groupID)
        }(i)
    }
    
    outerWg.Wait()
    fmt.Println("All groups completed")
}
```

**Key Learning**: WaitGroup handles synchronization but lacks cancellation capability

## Level 4: Context for Cancellation

### Flow: Context → Goroutines → Cancellation/Timeout

```go
// Pattern 1: Basic Context Cancellation
func main() {
    // Step 1: Create context with cancel
    ctx, cancel := context.WithCancel(context.Background())
    
    // Step 2: Launch goroutine with context
    go func() {
        for {
            select {
            case <-ctx.Done():
                fmt.Println("Goroutine cancelled")
                return
            default:
                fmt.Println("Working...")
                time.Sleep(500 * time.Millisecond)
            }
        }
    }()
    
    // Step 3: Cancel after some time
    time.Sleep(2 * time.Second)
    cancel()
    
    // Give time for goroutine to finish
    time.Sleep(100 * time.Millisecond)
    fmt.Println("Main finished")
}

// Pattern 2: Context with Timeout
func main() {
    // Step 1: Create context with timeout
    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel()
    
    // Step 2: Launch operation
    result := make(chan string)
    
    go func() {
        // Simulate long operation
        time.Sleep(3 * time.Second)
        result <- "Operation completed"
    }()
    
    // Step 3: Wait with timeout
    select {
    case res := <-result:
        fmt.Println(res)
    case <-ctx.Done():
        fmt.Println("Operation timed out:", ctx.Err())
    }
}

// Pattern 3: Context with Values
func main() {
    // Step 1: Create context with value
    ctx := context.WithValue(context.Background(), "userID", "12345")
    
    // Step 2: Pass context through call chain
    processRequest(ctx)
}

func processRequest(ctx context.Context) {
    // Step 3: Extract value
    if userID, ok := ctx.Value("userID").(string); ok {
        fmt.Printf("Processing request for user: %s\n", userID)
    }
    
    // Pass context down
    doWork(ctx)
}

func doWork(ctx context.Context) {
    if userID, ok := ctx.Value("userID").(string); ok {
        fmt.Printf("Doing work for user: %s\n", userID)
    }
}
```

## Level 5: Complete Patterns

### Pattern 1: Producer-Consumer with Full Flow

```go
func main() {
    // Step 1: Setup context for cancellation
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    
    // Step 2: Create channels
    jobs := make(chan int, 10)
    results := make(chan int, 10)
    
    // Step 3: Setup WaitGroup for workers
    var wg sync.WaitGroup
    
    // Step 4: Start workers
    for w := 1; w <= 3; w++ {
        wg.Add(1)
        go worker(ctx, w, jobs, results, &wg)
    }
    
    // Step 5: Start producer
    go producer(ctx, jobs)
    
    // Step 6: Close results when workers done
    go func() {
        wg.Wait()
        close(results)
    }()
    
    // Step 7: Consume results
    for result := range results {
        fmt.Printf("Result: %d\n", result)
    }
    
    fmt.Println("All work completed")
}

func producer(ctx context.Context, jobs chan<- int) {
    defer close(jobs)
    
    for i := 1; i <= 20; i++ {
        select {
        case <-ctx.Done():
            fmt.Println("Producer cancelled")
            return
        case jobs <- i:
            fmt.Printf("Produced job %d\n", i)
            time.Sleep(100 * time.Millisecond)
        }
    }
}

func worker(ctx context.Context, id int, jobs <-chan int, results chan<- int, wg *sync.WaitGroup) {
    defer wg.Done()
    
    for {
        select {
        case <-ctx.Done():
            fmt.Printf("Worker %d cancelled\n", id)
            return
        case job, ok := <-jobs:
            if !ok {
                fmt.Printf("Worker %d finished\n", id)
                return
            }
            
            // Process job
            result := job * job
            
            select {
            case <-ctx.Done():
                return
            case results <- result:
                fmt.Printf("Worker %d processed job %d\n", id, job)
            }
        }
    }
}
```

### Pattern 2: Pipeline with Full Flow Control

```go
func main() {
    // Step 1: Context for overall control
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    // Step 2: Pipeline stages
    numbers := generateNumbers(ctx)
    squared := squareNumbers(ctx, numbers)
    printed := printNumbers(ctx, squared)
    
    // Step 3: Wait for completion or cancellation
    select {
    case <-printed:
        fmt.Println("Pipeline completed")
    case <-time.After(5 * time.Second):
        fmt.Println("Pipeline timeout")
        cancel()
    }
}

// Stage 1: Generate numbers
func generateNumbers(ctx context.Context) <-chan int {
    out := make(chan int)
    
    go func() {
        defer close(out)
        for i := 1; i <= 10; i++ {
            select {
            case <-ctx.Done():
                return
            case out <- i:
                time.Sleep(200 * time.Millisecond)
            }
        }
    }()
    
    return out
}

// Stage 2: Square numbers
func squareNumbers(ctx context.Context, in <-chan int) <-chan int {
    out := make(chan int)
    
    go func() {
        defer close(out)
        for num := range in {
            select {
            case <-ctx.Done():
                return
            case out <- num * num:
            }
        }
    }()
    
    return out
}

// Stage 3: Print numbers
func printNumbers(ctx context.Context, in <-chan int) <-chan struct{} {
    done := make(chan struct{})
    
    go func() {
        defer close(done)
        for num := range in {
            select {
            case <-ctx.Done():
                return
            default:
                fmt.Printf("Result: %d\n", num)
            }
        }
    }()
    
    return done
}
```

### Pattern 3: Worker Pool with Rate Limiting

```go
type Job struct {
    ID   int
    Data string
}

type Result struct {
    JobID int
    Data  string
    Error error
}

func main() {
    // Step 1: Setup context
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()
    
    // Step 2: Create channels
    jobs := make(chan Job, 100)
    results := make(chan Result, 100)
    
    // Step 3: Rate limiter
    limiter := time.NewTicker(100 * time.Millisecond)
    defer limiter.Stop()
    
    // Step 4: Start worker pool
    var wg sync.WaitGroup
    for w := 1; w <= 5; w++ {
        wg.Add(1)
        go rateLimitedWorker(ctx, w, jobs, results, limiter.C, &wg)
    }
    
    // Step 5: Submit jobs
    go func() {
        defer close(jobs)
        for i := 1; i <= 50; i++ {
            select {
            case <-ctx.Done():
                return
            case jobs <- Job{ID: i, Data: fmt.Sprintf("job-%d", i)}:
            }
        }
    }()
    
    // Step 6: Collect results
    go func() {
        wg.Wait()
        close(results)
    }()
    
    // Step 7: Process results
    successCount := 0
    errorCount := 0
    
    for result := range results {
        if result.Error != nil {
            errorCount++
            fmt.Printf("Job %d failed: %v\n", result.JobID, result.Error)
        } else {
            successCount++
            fmt.Printf("Job %d succeeded: %s\n", result.JobID, result.Data)
        }
    }
    
    fmt.Printf("Completed: %d successful, %d errors\n", successCount, errorCount)
}

func rateLimitedWorker(
    ctx context.Context,
    id int,
    jobs <-chan Job,
    results chan<- Result,
    rateLimiter <-chan time.Time,
    wg *sync.WaitGroup,
) {
    defer wg.Done()
    
    for job := range jobs {
        select {
        case <-ctx.Done():
            results <- Result{
                JobID: job.ID,
                Error: ctx.Err(),
            }
            return
        case <-rateLimiter:
            // Rate limited - proceed with job
        }
        
        // Process job
        result := processJob(ctx, job)
        
        select {
        case <-ctx.Done():
            return
        case results <- result:
        }
    }
}

func processJob(ctx context.Context, job Job) Result {
    // Simulate work
    select {
    case <-ctx.Done():
        return Result{JobID: job.ID, Error: ctx.Err()}
    case <-time.After(50 * time.Millisecond):
        return Result{
            JobID: job.ID,
            Data:  fmt.Sprintf("processed-%s", job.Data),
        }
    }
}
```

## Common Flow Patterns

### 1. Sequential Processing Flow

```go
// Flow: Init → Process → Cleanup
func processSequentially(ctx context.Context, items []string) error {
    // Step 1: Initialize resources
    processor := NewProcessor()
    defer processor.Close()
    
    // Step 2: Process each item
    for _, item := range items {
        select {
        case <-ctx.Done():
            return ctx.Err()
        default:
            if err := processor.Process(item); err != nil {
                return fmt.Errorf("failed to process %s: %w", item, err)
            }
        }
    }
    
    // Step 3: Cleanup happens via defer
    return nil
}
```

### 2. Fan-Out/Fan-In Flow

```go
// Flow: Split → Process in Parallel → Merge
func fanOutFanIn(ctx context.Context, input []int) []int {
    // Step 1: Create channels for fan-out
    numWorkers := 3
    channels := make([]<-chan int, numWorkers)
    
    // Step 2: Distribute work (fan-out)
    for i := 0; i < numWorkers; i++ {
        ch := make(chan int)
        channels[i] = ch
        
        go func(workerID int, out chan<- int) {
            defer close(out)
            
            // Process subset of input
            for j := workerID; j < len(input); j += numWorkers {
                select {
                case <-ctx.Done():
                    return
                case out <- input[j] * input[j]:
                }
            }
        }(i, ch)
    }
    
    // Step 3: Merge results (fan-in)
    results := merge(ctx, channels...)
    
    // Step 4: Collect all results
    var output []int
    for result := range results {
        output = append(output, result)
    }
    
    return output
}

func merge(ctx context.Context, channels ...<-chan int) <-chan int {
    out := make(chan int)
    var wg sync.WaitGroup
    
    // Start goroutine for each input channel
    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan int) {
            defer wg.Done()
            for val := range c {
                select {
                case <-ctx.Done():
                    return
                case out <- val:
                }
            }
        }(ch)
    }
    
    // Close output channel when all input channels are done
    go func() {
        wg.Wait()
        close(out)
    }()
    
    return out
}
```

### 3. Retry with Backoff Flow

```go
// Flow: Try → Fail → Wait → Retry → Success/Final Failure
func retryWithBackoff(ctx context.Context, operation func() error) error {
    backoff := 100 * time.Millisecond
    maxBackoff := 5 * time.Second
    
    for attempt := 0; attempt < 5; attempt++ {
        // Step 1: Try operation
        err := operation()
        if err == nil {
            return nil // Success
        }
        
        // Step 2: Check if we should retry
        if attempt == 4 {
            return fmt.Errorf("operation failed after %d attempts: %w", attempt+1, err)
        }
        
        // Step 3: Wait with backoff
        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-time.After(backoff):
            // Step 4: Increase backoff
            backoff *= 2
            if backoff > maxBackoff {
                backoff = maxBackoff
            }
        }
    }
    
    return errors.New("unexpected error")
}
```

### 4. Graceful Shutdown Flow

```go
// Flow: Signal → Stop Accepting → Drain → Cleanup → Exit
func gracefulShutdown() {
    // Step 1: Setup signal handling
    sigChan := make(chan os.Signal, 1)
    signal.Notify(sigChan, os.Interrupt, syscall.SIGTERM)
    
    // Step 2: Create shutdown context
    ctx, cancel := context.WithCancel(context.Background())
    
    // Step 3: Start workers
    var wg sync.WaitGroup
    jobs := make(chan Job, 100)
    
    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            
            for {
                select {
                case <-ctx.Done():
                    fmt.Printf("Worker %d shutting down\n", id)
                    return
                case job, ok := <-jobs:
                    if !ok {
                        fmt.Printf("Worker %d finished\n", id)
                        return
                    }
                    processJob(ctx, job)
                }
            }
        }(i)
    }
    
    // Step 4: Main loop
    go func() {
        for i := 0; ; i++ {
            select {
            case <-ctx.Done():
                close(jobs)
                return
            default:
                jobs <- Job{ID: i}
                time.Sleep(100 * time.Millisecond)
            }
        }
    }()
    
    // Step 5: Wait for shutdown signal
    <-sigChan
    fmt.Println("Shutdown signal received")
    
    // Step 6: Initiate graceful shutdown
    cancel() // Signal all goroutines to stop
    
    // Step 7: Wait for workers to finish
    shutdownCtx, shutdownCancel := context.WithTimeout(
        context.Background(), 
        10*time.Second,
    )
    defer shutdownCancel()
    
    done := make(chan struct{})
    go func() {
        wg.Wait()
        close(done)
    }()
    
    // Step 8: Wait for completion or timeout
    select {
    case <-done:
        fmt.Println("Graceful shutdown completed")
    case <-shutdownCtx.Done():
        fmt.Println("Shutdown timeout - forcing exit")
    }
}
```

## Summary: Complete Flow Pattern

### The Master Pattern: Combining All Elements

```go
func masterConcurrencyPattern(ctx context.Context) error {
    // 1. Context for cancellation
    ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
    defer cancel()
    
    // 2. Channels for communication
    jobs := make(chan Job, 100)
    results := make(chan Result, 100)
    errors := make(chan error, 10)
    
    // 3. WaitGroup for synchronization
    var wg sync.WaitGroup
    
    // 4. Launch goroutines
    // Workers
    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            
            for job := range jobs {
                select {
                case <-ctx.Done():
                    return
                default:
                    result, err := processJobWithTimeout(ctx, job)
                    if err != nil {
                        select {
                        case errors <- err:
                        case <-ctx.Done():
                            return
                        }
                    } else {
                        select {
                        case results <- result:
                        case <-ctx.Done():
                            return
                        }
                    }
                }
            }
        }(i)
    }
    
    // Error collector
    var errorList []error
    go func() {
        for err := range errors {
            errorList = append(errorList, err)
        }
    }()
    
    // Result collector
    var resultList []Result
    go func() {
        for result := range results {
            resultList = append(resultList, result)
        }
    }()
    
    // 5. Send work
    go func() {
        defer close(jobs)
        for i := 0; i < 100; i++ {
            select {
            case <-ctx.Done():
                return
            case jobs <- Job{ID: i}:
            }
        }
    }()
    
    // 6. Wait for completion
    wg.Wait()
    close(results)
    close(errors)
    
    // 7. Check results
    if len(errorList) > 0 {
        return fmt.Errorf("encountered %d errors", len(errorList))
    }
    
    fmt.Printf("Processed %d results successfully\n", len(resultList))
    return nil
}
```

## Key Takeaways

### Flow Order:
1. **Context** - Always start with context for cancellation/timeout
2. **Channels** - Create channels for communication
3. **WaitGroup** - Setup synchronization before launching goroutines
4. **Goroutines** - Launch workers with proper cleanup (defer)
5. **Coordination** - Use select for non-blocking operations
6. **Cleanup** - Close channels, wait for goroutines, handle errors

### Best Practices:
- Always pass context as first parameter
- Close channels from sender side only
- Use defer for cleanup operations
- Handle context cancellation in select statements
- Add to WaitGroup before launching goroutines
- Use buffered channels to prevent goroutine blocking
- Always have a way to stop goroutines