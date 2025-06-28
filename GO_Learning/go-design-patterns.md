# Go Architecture & Design Patterns Cheat Sheet

## 1. Creational Patterns

### Singleton Pattern
```go
// Thread-safe singleton using sync.Once
type Database struct {
    connection string
}

var (
    instance *Database
    once     sync.Once
)

func GetDatabase() *Database {
    once.Do(func() {
        instance = &Database{
            connection: "localhost:5432",
        }
    })
    return instance
}

// Alternative: Using init function
var db *Database

func init() {
    db = &Database{
        connection: "localhost:5432",
    }
}
```

### Factory Pattern
```go
// Factory interface
type StorageFactory interface {
    CreateStorage() Storage
}

type Storage interface {
    Save(data []byte) error
    Load(id string) ([]byte, error)
}

// Concrete implementations
type S3Storage struct {
    bucket string
}

func (s *S3Storage) Save(data []byte) error {
    // S3 implementation
    return nil
}

func (s *S3Storage) Load(id string) ([]byte, error) {
    // S3 implementation
    return nil, nil
}

type FileStorage struct {
    basePath string
}

func (f *FileStorage) Save(data []byte) error {
    // File implementation
    return nil
}

func (f *FileStorage) Load(id string) ([]byte, error) {
    // File implementation
    return nil, nil
}

// Factory implementations
type S3Factory struct {
    bucket string
}

func (f *S3Factory) CreateStorage() Storage {
    return &S3Storage{bucket: f.bucket}
}

type FileFactory struct {
    basePath string
}

func (f *FileFactory) CreateStorage() Storage {
    return &FileStorage{basePath: f.basePath}
}

// Usage
func NewStorageFactory(storageType string) StorageFactory {
    switch storageType {
    case "s3":
        return &S3Factory{bucket: "my-bucket"}
    case "file":
        return &FileFactory{basePath: "/data"}
    default:
        return &FileFactory{basePath: "/tmp"}
    }
}
```

### Builder Pattern
```go
// Server configuration builder
type Server struct {
    host         string
    port         int
    timeout      time.Duration
    maxConns     int
    tls          bool
    certFile     string
    keyFile      string
    middlewares  []Middleware
}

type ServerBuilder struct {
    server *Server
}

func NewServerBuilder() *ServerBuilder {
    return &ServerBuilder{
        server: &Server{
            host:     "localhost",
            port:     8080,
            timeout:  30 * time.Second,
            maxConns: 100,
        },
    }
}

func (b *ServerBuilder) WithHost(host string) *ServerBuilder {
    b.server.host = host
    return b
}

func (b *ServerBuilder) WithPort(port int) *ServerBuilder {
    b.server.port = port
    return b
}

func (b *ServerBuilder) WithTimeout(timeout time.Duration) *ServerBuilder {
    b.server.timeout = timeout
    return b
}

func (b *ServerBuilder) WithTLS(certFile, keyFile string) *ServerBuilder {
    b.server.tls = true
    b.server.certFile = certFile
    b.server.keyFile = keyFile
    return b
}

func (b *ServerBuilder) WithMiddleware(m Middleware) *ServerBuilder {
    b.server.middlewares = append(b.server.middlewares, m)
    return b
}

func (b *ServerBuilder) Build() (*Server, error) {
    // Validation
    if b.server.tls && (b.server.certFile == "" || b.server.keyFile == "") {
        return nil, errors.New("TLS enabled but cert/key files not provided")
    }
    return b.server, nil
}

// Usage
server, err := NewServerBuilder().
    WithHost("0.0.0.0").
    WithPort(443).
    WithTLS("cert.pem", "key.pem").
    WithTimeout(60 * time.Second).
    Build()
```

### Object Pool Pattern
```go
type Connection struct {
    id   int
    host string
}

type ConnectionPool struct {
    mu          sync.Mutex
    connections chan *Connection
    factory     func() *Connection
}

func NewConnectionPool(size int, factory func() *Connection) *ConnectionPool {
    pool := &ConnectionPool{
        connections: make(chan *Connection, size),
        factory:     factory,
    }
    
    // Pre-fill pool
    for i := 0; i < size; i++ {
        conn := factory()
        conn.id = i
        pool.connections <- conn
    }
    
    return pool
}

func (p *ConnectionPool) Get(ctx context.Context) (*Connection, error) {
    select {
    case conn := <-p.connections:
        return conn, nil
    case <-ctx.Done():
        return nil, ctx.Err()
    }
}

func (p *ConnectionPool) Put(conn *Connection) {
    select {
    case p.connections <- conn:
        // Connection returned to pool
    default:
        // Pool full, discard connection
    }
}

// Usage
pool := NewConnectionPool(10, func() *Connection {
    return &Connection{host: "localhost:5432"}
})

ctx := context.Background()
conn, err := pool.Get(ctx)
if err != nil {
    log.Fatal(err)
}
defer pool.Put(conn)
```

## 2. Structural Patterns

### Adapter Pattern
```go
// Legacy payment interface
type LegacyPayment interface {
    MakePayment(amount float64) string
}

type OldPaymentSystem struct{}

func (o *OldPaymentSystem) MakePayment(amount float64) string {
    return fmt.Sprintf("Paid $%.2f using legacy system", amount)
}

// New payment interface
type PaymentProcessor interface {
    ProcessPayment(ctx context.Context, amount float64, currency string) error
}

// Adapter
type PaymentAdapter struct {
    legacy LegacyPayment
}

func NewPaymentAdapter(legacy LegacyPayment) PaymentProcessor {
    return &PaymentAdapter{legacy: legacy}
}

func (a *PaymentAdapter) ProcessPayment(ctx context.Context, amount float64, currency string) error {
    // Adapt to legacy interface
    if currency != "USD" {
        return errors.New("legacy system only supports USD")
    }
    
    result := a.legacy.MakePayment(amount)
    if result == "" {
        return errors.New("payment failed")
    }
    
    return nil
}
```

### Decorator Pattern
```go
// Component interface
type DataSource interface {
    Read() ([]byte, error)
    Write(data []byte) error
}

// Concrete component
type FileDataSource struct {
    filename string
}

func (f *FileDataSource) Read() ([]byte, error) {
    return os.ReadFile(f.filename)
}

func (f *FileDataSource) Write(data []byte) error {
    return os.WriteFile(f.filename, data, 0644)
}

// Base decorator
type DataSourceDecorator struct {
    source DataSource
}

// Encryption decorator
type EncryptionDecorator struct {
    DataSourceDecorator
    key []byte
}

func NewEncryptionDecorator(source DataSource, key []byte) *EncryptionDecorator {
    return &EncryptionDecorator{
        DataSourceDecorator: DataSourceDecorator{source: source},
        key:                key,
    }
}

func (e *EncryptionDecorator) Read() ([]byte, error) {
    data, err := e.source.Read()
    if err != nil {
        return nil, err
    }
    return e.decrypt(data), nil
}

func (e *EncryptionDecorator) Write(data []byte) error {
    encrypted := e.encrypt(data)
    return e.source.Write(encrypted)
}

// Compression decorator
type CompressionDecorator struct {
    DataSourceDecorator
}

func NewCompressionDecorator(source DataSource) *CompressionDecorator {
    return &CompressionDecorator{
        DataSourceDecorator: DataSourceDecorator{source: source},
    }
}

func (c *CompressionDecorator) Read() ([]byte, error) {
    data, err := c.source.Read()
    if err != nil {
        return nil, err
    }
    return c.decompress(data), nil
}

func (c *CompressionDecorator) Write(data []byte) error {
    compressed := c.compress(data)
    return c.source.Write(compressed)
}

// Usage - decorators can be stacked
source := &FileDataSource{filename: "data.txt"}
encrypted := NewEncryptionDecorator(source, []byte("secret-key"))
compressed := NewCompressionDecorator(encrypted)

// Now reads will decompress then decrypt
data, _ := compressed.Read()
```

### Facade Pattern
```go
// Complex subsystems
type UserService struct{}

func (u *UserService) GetUser(id string) (*User, error) {
    // Complex user retrieval logic
    return &User{ID: id, Name: "John"}, nil
}

type OrderService struct{}

func (o *OrderService) GetUserOrders(userID string) ([]*Order, error) {
    // Complex order retrieval logic
    return []*Order{{ID: "1", UserID: userID}}, nil
}

type PaymentService struct{}

func (p *PaymentService) GetPaymentMethods(userID string) ([]*PaymentMethod, error) {
    // Complex payment retrieval logic
    return []*PaymentMethod{{ID: "1", Type: "card"}}, nil
}

// Facade
type CustomerFacade struct {
    userService    *UserService
    orderService   *OrderService
    paymentService *PaymentService
}

func NewCustomerFacade() *CustomerFacade {
    return &CustomerFacade{
        userService:    &UserService{},
        orderService:   &OrderService{},
        paymentService: &PaymentService{},
    }
}

type CustomerProfile struct {
    User           *User
    Orders         []*Order
    PaymentMethods []*PaymentMethod
}

func (f *CustomerFacade) GetCustomerProfile(userID string) (*CustomerProfile, error) {
    // Simplified interface to complex subsystems
    user, err := f.userService.GetUser(userID)
    if err != nil {
        return nil, err
    }
    
    orders, err := f.orderService.GetUserOrders(userID)
    if err != nil {
        return nil, err
    }
    
    payments, err := f.paymentService.GetPaymentMethods(userID)
    if err != nil {
        return nil, err
    }
    
    return &CustomerProfile{
        User:           user,
        Orders:         orders,
        PaymentMethods: payments,
    }, nil
}
```

### Proxy Pattern
```go
// Subject interface
type Database interface {
    Query(sql string) ([]Row, error)
}

// Real subject
type RealDatabase struct {
    connectionString string
}

func (d *RealDatabase) Query(sql string) ([]Row, error) {
    fmt.Println("Executing query:", sql)
    // Real database query
    return []Row{}, nil
}

// Proxy with caching
type CachedDatabase struct {
    realDB Database
    cache  map[string][]Row
    mu     sync.RWMutex
}

func NewCachedDatabase(db Database) *CachedDatabase {
    return &CachedDatabase{
        realDB: db,
        cache:  make(map[string][]Row),
    }
}

func (c *CachedDatabase) Query(sql string) ([]Row, error) {
    // Check cache first
    c.mu.RLock()
    if rows, ok := c.cache[sql]; ok {
        c.mu.RUnlock()
        fmt.Println("Returning from cache")
        return rows, nil
    }
    c.mu.RUnlock()
    
    // Query real database
    rows, err := c.realDB.Query(sql)
    if err != nil {
        return nil, err
    }
    
    // Cache results
    c.mu.Lock()
    c.cache[sql] = rows
    c.mu.Unlock()
    
    return rows, nil
}

// Security proxy
type SecureDatabase struct {
    realDB   Database
    userRole string
}

func NewSecureDatabase(db Database, role string) *SecureDatabase {
    return &SecureDatabase{
        realDB:   db,
        userRole: role,
    }
}

func (s *SecureDatabase) Query(sql string) ([]Row, error) {
    // Check permissions
    if s.userRole != "admin" && strings.Contains(sql, "DELETE") {
        return nil, errors.New("permission denied")
    }
    
    return s.realDB.Query(sql)
}
```

## 3. Behavioral Patterns

### Strategy Pattern
```go
// Strategy interface
type PaymentStrategy interface {
    Pay(amount float64) error
}

// Concrete strategies
type CreditCardPayment struct {
    cardNumber string
    cvv        string
}

func (c *CreditCardPayment) Pay(amount float64) error {
    fmt.Printf("Paying $%.2f with credit card %s\n", amount, c.cardNumber)
    return nil
}

type PayPalPayment struct {
    email string
}

func (p *PayPalPayment) Pay(amount float64) error {
    fmt.Printf("Paying $%.2f with PayPal account %s\n", amount, p.email)
    return nil
}

type CryptoPayment struct {
    walletAddress string
}

func (c *CryptoPayment) Pay(amount float64) error {
    fmt.Printf("Paying $%.2f with crypto wallet %s\n", amount, c.walletAddress)
    return nil
}

// Context
type PaymentProcessor struct {
    strategy PaymentStrategy
}

func (p *PaymentProcessor) SetStrategy(strategy PaymentStrategy) {
    p.strategy = strategy
}

func (p *PaymentProcessor) ProcessPayment(amount float64) error {
    if p.strategy == nil {
        return errors.New("payment strategy not set")
    }
    return p.strategy.Pay(amount)
}

// Usage
processor := &PaymentProcessor{}

// Credit card payment
processor.SetStrategy(&CreditCardPayment{
    cardNumber: "1234-5678-9012-3456",
    cvv:        "123",
})
processor.ProcessPayment(100.00)

// Switch to PayPal
processor.SetStrategy(&PayPalPayment{
    email: "user@example.com",
})
processor.ProcessPayment(50.00)
```

### Observer Pattern
```go
// Subject interface
type Subject interface {
    Attach(Observer)
    Detach(Observer)
    Notify()
}

// Observer interface
type Observer interface {
    Update(data interface{})
}

// Concrete subject
type StockPrice struct {
    observers []Observer
    symbol    string
    price     float64
    mu        sync.RWMutex
}

func NewStockPrice(symbol string) *StockPrice {
    return &StockPrice{
        symbol:    symbol,
        observers: make([]Observer, 0),
    }
}

func (s *StockPrice) Attach(observer Observer) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.observers = append(s.observers, observer)
}

func (s *StockPrice) Detach(observer Observer) {
    s.mu.Lock()
    defer s.mu.Unlock()
    for i, obs := range s.observers {
        if obs == observer {
            s.observers = append(s.observers[:i], s.observers[i+1:]...)
            break
        }
    }
}

func (s *StockPrice) Notify() {
    s.mu.RLock()
    observers := make([]Observer, len(s.observers))
    copy(observers, s.observers)
    s.mu.RUnlock()
    
    for _, observer := range observers {
        observer.Update(map[string]interface{}{
            "symbol": s.symbol,
            "price":  s.price,
        })
    }
}

func (s *StockPrice) SetPrice(price float64) {
    s.mu.Lock()
    s.price = price
    s.mu.Unlock()
    s.Notify()
}

// Concrete observers
type PriceDisplay struct {
    name string
}

func (p *PriceDisplay) Update(data interface{}) {
    stockData := data.(map[string]interface{})
    fmt.Printf("[%s] Stock %s: $%.2f\n", 
        p.name, stockData["symbol"], stockData["price"])
}

type PriceAlert struct {
    threshold float64
}

func (p *PriceAlert) Update(data interface{}) {
    stockData := data.(map[string]interface{})
    price := stockData["price"].(float64)
    if price > p.threshold {
        fmt.Printf("ALERT: %s exceeded threshold! Price: $%.2f\n", 
            stockData["symbol"], price)
    }
}

// Usage
stock := NewStockPrice("AAPL")

display := &PriceDisplay{name: "Main Display"}
alert := &PriceAlert{threshold: 150.0}

stock.Attach(display)
stock.Attach(alert)

stock.SetPrice(145.0)
stock.SetPrice(155.0) // Triggers alert
```

### Command Pattern
```go
// Command interface
type Command interface {
    Execute() error
    Undo() error
}

// Receiver
type Light struct {
    isOn bool
    name string
}

func (l *Light) TurnOn() {
    l.isOn = true
    fmt.Printf("Light %s is ON\n", l.name)
}

func (l *Light) TurnOff() {
    l.isOn = false
    fmt.Printf("Light %s is OFF\n", l.name)
}

// Concrete commands
type LightOnCommand struct {
    light *Light
}

func (c *LightOnCommand) Execute() error {
    c.light.TurnOn()
    return nil
}

func (c *LightOnCommand) Undo() error {
    c.light.TurnOff()
    return nil
}

type LightOffCommand struct {
    light *Light
}

func (c *LightOffCommand) Execute() error {
    c.light.TurnOff()
    return nil
}

func (c *LightOffCommand) Undo() error {
    c.light.TurnOn()
    return nil
}

// Invoker
type RemoteControl struct {
    commands map[string]Command
    history  []Command
}

func NewRemoteControl() *RemoteControl {
    return &RemoteControl{
        commands: make(map[string]Command),
        history:  make([]Command, 0),
    }
}

func (r *RemoteControl) SetCommand(button string, cmd Command) {
    r.commands[button] = cmd
}

func (r *RemoteControl) PressButton(button string) error {
    cmd, ok := r.commands[button]
    if !ok {
        return errors.New("button not configured")
    }
    
    if err := cmd.Execute(); err != nil {
        return err
    }
    
    r.history = append(r.history, cmd)
    return nil
}

func (r *RemoteControl) Undo() error {
    if len(r.history) == 0 {
        return errors.New("nothing to undo")
    }
    
    lastCmd := r.history[len(r.history)-1]
    r.history = r.history[:len(r.history)-1]
    
    return lastCmd.Undo()
}

// Macro command
type MacroCommand struct {
    commands []Command
}

func (m *MacroCommand) Execute() error {
    for _, cmd := range m.commands {
        if err := cmd.Execute(); err != nil {
            return err
        }
    }
    return nil
}

func (m *MacroCommand) Undo() error {
    // Undo in reverse order
    for i := len(m.commands) - 1; i >= 0; i-- {
        if err := m.commands[i].Undo(); err != nil {
            return err
        }
    }
    return nil
}
```

### Chain of Responsibility Pattern
```go
// Handler interface
type Handler interface {
    SetNext(Handler)
    Handle(*Request) error
}

type Request struct {
    Type   string
    Amount float64
    UserID string
}

// Base handler
type BaseHandler struct {
    next Handler
}

func (h *BaseHandler) SetNext(next Handler) {
    h.next = next
}

// Concrete handlers
type AuthenticationHandler struct {
    BaseHandler
}

func (h *AuthenticationHandler) Handle(req *Request) error {
    fmt.Println("Authenticating user...")
    if req.UserID == "" {
        return errors.New("authentication failed")
    }
    
    if h.next != nil {
        return h.next.Handle(req)
    }
    return nil
}

type AuthorizationHandler struct {
    BaseHandler
    maxAmount float64
}

func (h *AuthorizationHandler) Handle(req *Request) error {
    fmt.Println("Checking authorization...")
    if req.Amount > h.maxAmount {
        return errors.New("amount exceeds authorized limit")
    }
    
    if h.next != nil {
        return h.next.Handle(req)
    }
    return nil
}

type ValidationHandler struct {
    BaseHandler
}

func (h *ValidationHandler) Handle(req *Request) error {
    fmt.Println("Validating request...")
    if req.Amount <= 0 {
        return errors.New("invalid amount")
    }
    
    if h.next != nil {
        return h.next.Handle(req)
    }
    return nil
}

type ProcessingHandler struct {
    BaseHandler
}

func (h *ProcessingHandler) Handle(req *Request) error {
    fmt.Printf("Processing %s request for $%.2f\n", req.Type, req.Amount)
    return nil
}

// Usage
func BuildChain() Handler {
    auth := &AuthenticationHandler{}
    authz := &AuthorizationHandler{maxAmount: 10000}
    valid := &ValidationHandler{}
    proc := &ProcessingHandler{}
    
    auth.SetNext(authz)
    authz.SetNext(valid)
    valid.SetNext(proc)
    
    return auth
}
```

## 4. Concurrency Patterns

### Worker Pool Pattern
```go
type Task func() error

type WorkerPool struct {
    maxWorkers int
    tasks      chan Task
    results    chan error
    wg         sync.WaitGroup
}

func NewWorkerPool(maxWorkers int) *WorkerPool {
    return &WorkerPool{
        maxWorkers: maxWorkers,
        tasks:      make(chan Task),
        results:    make(chan error, maxWorkers),
    }
}

func (p *WorkerPool) Start(ctx context.Context) {
    for i := 0; i < p.maxWorkers; i++ {
        p.wg.Add(1)
        go p.worker(ctx, i)
    }
}

func (p *WorkerPool) worker(ctx context.Context, id int) {
    defer p.wg.Done()
    
    for {
        select {
        case task, ok := <-p.tasks:
            if !ok {
                return
            }
            
            err := task()
            
            select {
            case p.results <- err:
            case <-ctx.Done():
                return
            }
            
        case <-ctx.Done():
            return
        }
    }
}

func (p *WorkerPool) Submit(task Task) {
    p.tasks <- task
}

func (p *WorkerPool) Stop() {
    close(p.tasks)
    p.wg.Wait()
    close(p.results)
}

// Usage
ctx := context.Background()
pool := NewWorkerPool(5)
pool.Start(ctx)

// Submit tasks
for i := 0; i < 20; i++ {
    taskID := i
    pool.Submit(func() error {
        fmt.Printf("Processing task %d\n", taskID)
        time.Sleep(100 * time.Millisecond)
        return nil
    })
}

// Collect results
go func() {
    for err := range pool.results {
        if err != nil {
            log.Printf("Task error: %v", err)
        }
    }
}()

pool.Stop()
```

### Pipeline Pattern
```go
// Pipeline stage function type
type Stage func(in <-chan interface{}) <-chan interface{}

// Pipeline builder
type Pipeline struct {
    stages []Stage
}

func NewPipeline(stages ...Stage) *Pipeline {
    return &Pipeline{stages: stages}
}

func (p *Pipeline) Execute(input <-chan interface{}) <-chan interface{} {
    output := input
    for _, stage := range p.stages {
        output = stage(output)
    }
    return output
}

// Example stages
func MultiplyBy(factor int) Stage {
    return func(in <-chan interface{}) <-chan interface{} {
        out := make(chan interface{})
        go func() {
            defer close(out)
            for val := range in {
                if num, ok := val.(int); ok {
                    out <- num * factor
                }
            }
        }()
        return out
    }
}

func FilterEven() Stage {
    return func(in <-chan interface{}) <-chan interface{} {
        out := make(chan interface{})
        go func() {
            defer close(out)
            for val := range in {
                if num, ok := val.(int); ok && num%2 == 0 {
                    out <- num
                }
            }
        }()
        return out
    }
}

func ConvertToString() Stage {
    return func(in <-chan interface{}) <-chan interface{} {
        out := make(chan interface{})
        go func() {
            defer close(out)
            for val := range in {
                out <- fmt.Sprintf("%v", val)
            }
        }()
        return out
    }
}

// Usage
input := make(chan interface{})
go func() {
    defer close(input)
    for i := 1; i <= 10; i++ {
        input <- i
    }
}()

pipeline := NewPipeline(
    MultiplyBy(2),
    FilterEven(),
    ConvertToString(),
)

output := pipeline.Execute(input)
for result := range output {
    fmt.Println(result)
}
```

### Fan-In/Fan-Out Pattern
```go
// Fan-out: distribute work across multiple goroutines
func FanOut(in <-chan int, workers int) []<-chan int {
    channels := make([]<-chan int, workers)
    
    for i := 0; i < workers; i++ {
        ch := make(chan int)
        channels[i] = ch
        
        go func(output chan int) {
            for val := range in {
                // Simulate work
                result := val * val
                output <- result
            }
            close(output)
        }(ch)
    }
    
    return channels
}

// Fan-in: merge multiple channels into one
func FanIn(channels ...<-chan int) <-chan int {
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

// Usage
input := make(chan int)
go func() {
    defer close(input)
    for i := 1; i <= 100; i++ {
        input <- i
    }
}()

// Fan-out to 5 workers
workers := FanOut(input, 5)

// Fan-in results
results := FanIn(workers...)

for result := range results {
    fmt.Println(result)
}
```

## 5. Architectural Patterns

### Hexagonal Architecture (Ports and Adapters)
```go
// Domain model
type Order struct {
    ID         string
    CustomerID string
    Items      []OrderItem
    Total      float64
    Status     string
}

// Port (interface) - defined in domain
type OrderRepository interface {
    Save(order *Order) error
    FindByID(id string) (*Order, error)
    FindByCustomer(customerID string) ([]*Order, error)
}

type PaymentService interface {
    ProcessPayment(orderID string, amount float64) error
}

type NotificationService interface {
    SendOrderConfirmation(order *Order) error
}

// Use case / Application service
type OrderService struct {
    repo         OrderRepository
    payment      PaymentService
    notification NotificationService
}

func NewOrderService(
    repo OrderRepository,
    payment PaymentService,
    notification NotificationService,
) *OrderService {
    return &OrderService{
        repo:         repo,
        payment:      payment,
        notification: notification,
    }
}

func (s *OrderService) CreateOrder(order *Order) error {
    // Business logic
    if err := s.validateOrder(order); err != nil {
        return err
    }
    
    // Save order
    if err := s.repo.Save(order); err != nil {
        return err
    }
    
    // Process payment
    if err := s.payment.ProcessPayment(order.ID, order.Total); err != nil {
        order.Status = "payment_failed"
        s.repo.Save(order)
        return err
    }
    
    // Send confirmation
    if err := s.notification.SendOrderConfirmation(order); err != nil {
        // Log but don't fail
        log.Printf("Failed to send notification: %v", err)
    }
    
    order.Status = "confirmed"
    return s.repo.Save(order)
}

// Adapters (implementations)
type PostgresOrderRepository struct {
    db *sql.DB
}

func (r *PostgresOrderRepository) Save(order *Order) error {
    // PostgreSQL implementation
    return nil
}

func (r *PostgresOrderRepository) FindByID(id string) (*Order, error) {
    // PostgreSQL implementation
    return nil, nil
}

func (r *PostgresOrderRepository) FindByCustomer(customerID string) ([]*Order, error) {
    // PostgreSQL implementation
    return nil, nil
}

type StripePaymentService struct {
    apiKey string
}

func (s *StripePaymentService) ProcessPayment(orderID string, amount float64) error {
    // Stripe API implementation
    return nil
}

type EmailNotificationService struct {
    smtpServer string
}

func (e *EmailNotificationService) SendOrderConfirmation(order *Order) error {
    // Email implementation
    return nil
}

// HTTP adapter
type OrderHTTPHandler struct {
    service *OrderService
}

func (h *OrderHTTPHandler) CreateOrder(w http.ResponseWriter, r *http.Request) {
    var order Order
    if err := json.NewDecoder(r.Body).Decode(&order); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }
    
    if err := h.service.CreateOrder(&order); err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    json.NewEncoder(w).Encode(order)
}
```

### Clean Architecture
```go
// Entities (Enterprise Business Rules)
type User struct {
    ID       string
    Email    string
    Password string
}

// Use Cases (Application Business Rules)
type UserUseCase interface {
    CreateUser(email, password string) (*User, error)
    GetUser(id string) (*User, error)
    AuthenticateUser(email, password string) (*User, error)
}

type userUseCase struct {
    userRepo UserRepository
    hasher   PasswordHasher
}

func NewUserUseCase(repo UserRepository, hasher PasswordHasher) UserUseCase {
    return &userUseCase{
        userRepo: repo,
        hasher:   hasher,
    }
}

func (u *userUseCase) CreateUser(email, password string) (*User, error) {
    // Business rules
    if !isValidEmail(email) {
        return nil, errors.New("invalid email")
    }
    
    hashedPassword, err := u.hasher.Hash(password)
    if err != nil {
        return nil, err
    }
    
    user := &User{
        ID:       generateID(),
        Email:    email,
        Password: hashedPassword,
    }
    
    return u.userRepo.Create(user)
}

// Interface Adapters
type UserRepository interface {
    Create(user *User) (*User, error)
    FindByID(id string) (*User, error)
    FindByEmail(email string) (*User, error)
}

type PasswordHasher interface {
    Hash(password string) (string, error)
    Compare(hash, password string) error
}

// Frameworks & Drivers
type PostgresUserRepository struct {
    db *sql.DB
}

func (r *PostgresUserRepository) Create(user *User) (*User, error) {
    query := `INSERT INTO users (id, email, password) VALUES ($1, $2, $3)`
    _, err := r.db.Exec(query, user.ID, user.Email, user.Password)
    return user, err
}

type BCryptHasher struct {
    cost int
}

func (h *BCryptHasher) Hash(password string) (string, error) {
    bytes, err := bcrypt.GenerateFromPassword([]byte(password), h.cost)
    return string(bytes), err
}

// Presentation layer
type UserController struct {
    useCase UserUseCase
}

func (c *UserController) Register(w http.ResponseWriter, r *http.Request) {
    var req struct {
        Email    string `json:"email"`
        Password string `json:"password"`
    }
    
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }
    
    user, err := c.useCase.CreateUser(req.Email, req.Password)
    if err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }
    
    json.NewEncoder(w).Encode(user)
}
```

### CQRS (Command Query Responsibility Segregation)
```go
// Commands (write operations)
type CreateOrderCommand struct {
    CustomerID string
    Items      []OrderItem
}

type UpdateOrderStatusCommand struct {
    OrderID string
    Status  string
}

type CommandHandler interface {
    Handle(ctx context.Context, cmd interface{}) error
}

type OrderCommandHandler struct {
    repo OrderWriteRepository
    bus  EventBus
}

func (h *OrderCommandHandler) Handle(ctx context.Context, cmd interface{}) error {
    switch c := cmd.(type) {
    case CreateOrderCommand:
        return h.handleCreateOrder(ctx, c)
    case UpdateOrderStatusCommand:
        return h.handleUpdateStatus(ctx, c)
    default:
        return errors.New("unknown command")
    }
}

func (h *OrderCommandHandler) handleCreateOrder(ctx context.Context, cmd CreateOrderCommand) error {
    order := &Order{
        ID:         generateID(),
        CustomerID: cmd.CustomerID,
        Items:      cmd.Items,
        Status:     "pending",
    }
    
    if err := h.repo.Save(order); err != nil {
        return err
    }
    
    // Publish event
    event := OrderCreatedEvent{
        OrderID:    order.ID,
        CustomerID: order.CustomerID,
        Timestamp:  time.Now(),
    }
    
    return h.bus.Publish(event)
}

// Queries (read operations)
type GetOrderQuery struct {
    OrderID string
}

type GetCustomerOrdersQuery struct {
    CustomerID string
}

type QueryHandler interface {
    Handle(ctx context.Context, query interface{}) (interface{}, error)
}

type OrderQueryHandler struct {
    repo OrderReadRepository
}

func (h *OrderQueryHandler) Handle(ctx context.Context, query interface{}) (interface{}, error) {
    switch q := query.(type) {
    case GetOrderQuery:
        return h.repo.FindByID(q.OrderID)
    case GetCustomerOrdersQuery:
        return h.repo.FindByCustomer(q.CustomerID)
    default:
        return nil, errors.New("unknown query")
    }
}

// Separate read/write models
type OrderWriteRepository interface {
    Save(order *Order) error
    Update(order *Order) error
}

type OrderReadRepository interface {
    FindByID(id string) (*OrderView, error)
    FindByCustomer(customerID string) ([]*OrderView, error)
}

// Read model (optimized for queries)
type OrderView struct {
    ID           string
    CustomerName string
    TotalAmount  float64
    Status       string
    Items        []OrderItemView
}

// Event sourcing integration
type OrderAggregate struct {
    ID     string
    events []Event
}

func (a *OrderAggregate) Apply(event Event) {
    switch e := event.(type) {
    case OrderCreatedEvent:
        a.ID = e.OrderID
    case OrderStatusUpdatedEvent:
        // Update aggregate state
    }
    a.events = append(a.events, event)
}

func (a *OrderAggregate) GetUncommittedEvents() []Event {
    return a.events
}
```

## 6. Domain-Driven Design Patterns

### Repository Pattern
```go
// Domain model
type Product struct {
    ID          string
    Name        string
    Description string
    Price       Money
    Stock       int
}

type Money struct {
    Amount   float64
    Currency string
}

// Repository interface (part of domain layer)
type ProductRepository interface {
    NextID() string
    Save(product *Product) error
    FindByID(id string) (*Product, error)
    FindByName(name string) ([]*Product, error)
    Update(product *Product) error
    Delete(id string) error
}

// Implementation (infrastructure layer)
type PostgresProductRepository struct {
    db *sql.DB
}

func (r *PostgresProductRepository) NextID() string {
    return uuid.New().String()
}

func (r *PostgresProductRepository) Save(product *Product) error {
    query := `
        INSERT INTO products (id, name, description, price_amount, price_currency, stock)
        VALUES ($1, $2, $3, $4, $5, $6)
    `
    _, err := r.db.Exec(query,
        product.ID,
        product.Name,
        product.Description,
        product.Price.Amount,
        product.Price.Currency,
        product.Stock,
    )
    return err
}

// Specification pattern for complex queries
type Specification interface {
    IsSatisfiedBy(product *Product) bool
    ToSQL() (string, []interface{})
}

type PriceRangeSpecification struct {
    MinPrice float64
    MaxPrice float64
}

func (s *PriceRangeSpecification) IsSatisfiedBy(product *Product) bool {
    return product.Price.Amount >= s.MinPrice && product.Price.Amount <= s.MaxPrice
}

func (s *PriceRangeSpecification) ToSQL() (string, []interface{}) {
    return "price_amount BETWEEN ? AND ?", []interface{}{s.MinPrice, s.MaxPrice}
}

// Repository with specification
func (r *PostgresProductRepository) FindBySpecification(spec Specification) ([]*Product, error) {
    where, args := spec.ToSQL()
    query := fmt.Sprintf("SELECT * FROM products WHERE %s", where)
    
    rows, err := r.db.Query(query, args...)
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    
    var products []*Product
    // Scan rows...
    return products, nil
}
```

### Unit of Work Pattern
```go
// Unit of Work interface
type UnitOfWork interface {
    ProductRepository() ProductRepository
    OrderRepository() OrderRepository
    Commit() error
    Rollback() error
}

// Implementation
type SqlUnitOfWork struct {
    db         *sql.DB
    tx         *sql.Tx
    products   ProductRepository
    orders     OrderRepository
    committed  bool
}

func NewSqlUnitOfWork(db *sql.DB) (UnitOfWork, error) {
    tx, err := db.Begin()
    if err != nil {
        return nil, err
    }
    
    return &SqlUnitOfWork{
        db: db,
        tx: tx,
    }, nil
}

func (uow *SqlUnitOfWork) ProductRepository() ProductRepository {
    if uow.products == nil {
        uow.products = &TxProductRepository{tx: uow.tx}
    }
    return uow.products
}

func (uow *SqlUnitOfWork) OrderRepository() OrderRepository {
    if uow.orders == nil {
        uow.orders = &TxOrderRepository{tx: uow.tx}
    }
    return uow.orders
}

func (uow *SqlUnitOfWork) Commit() error {
    if uow.committed {
        return errors.New("unit of work already committed")
    }
    uow.committed = true
    return uow.tx.Commit()
}

func (uow *SqlUnitOfWork) Rollback() error {
    if uow.committed {
        return errors.New("unit of work already committed")
    }
    return uow.tx.Rollback()
}

// Usage
func TransferStock(uow UnitOfWork, fromID, toID string, quantity int) error {
    productRepo := uow.ProductRepository()
    
    from, err := productRepo.FindByID(fromID)
    if err != nil {
        return err
    }
    
    to, err := productRepo.FindByID(toID)
    if err != nil {
        return err
    }
    
    if from.Stock < quantity {
        return errors.New("insufficient stock")
    }
    
    from.Stock -= quantity
    to.Stock += quantity
    
    if err := productRepo.Update(from); err != nil {
        return err
    }
    
    if err := productRepo.Update(to); err != nil {
        return err
    }
    
    return uow.Commit()
}
```

## 7. Go-Specific Patterns

### Functional Options Pattern
```go
// Configuration struct
type Server struct {
    host        string
    port        int
    timeout     time.Duration
    maxConns    int
    tls         bool
    middlewares []Middleware
}

// Option function type
type Option func(*Server)

// Option functions
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

func WithTLS(enabled bool) Option {
    return func(s *Server) {
        s.tls = enabled
    }
}

func WithMiddleware(m Middleware) Option {
    return func(s *Server) {
        s.middlewares = append(s.middlewares, m)
    }
}

// Constructor with functional options
func NewServer(opts ...Option) *Server {
    // Default values
    s := &Server{
        host:     "localhost",
        port:     8080,
        timeout:  30 * time.Second,
        maxConns: 100,
    }
    
    // Apply options
    for _, opt := range opts {
        opt(s)
    }
    
    return s
}

// Usage
server := NewServer(
    WithHost("0.0.0.0"),
    WithPort(443),
    WithTLS(true),
    WithTimeout(60*time.Second),
    WithMiddleware(LoggingMiddleware),
    WithMiddleware(AuthMiddleware),
)
```

### Error Handling Pattern
```go
// Sentinel errors
var (
    ErrNotFound      = errors.New("not found")
    ErrAlreadyExists = errors.New("already exists")
    ErrInvalidInput  = errors.New("invalid input")
)

// Error types
type ValidationError struct {
    Field   string
    Message string
}

func (e ValidationError) Error() string {
    return fmt.Sprintf("validation error on field %s: %s", e.Field, e.Message)
}

// Error wrapping
func GetUser(id string) (*User, error) {
    user, err := db.QueryUser(id)
    if err != nil {
        if err == sql.ErrNoRows {
            return nil, ErrNotFound
        }
        return nil, fmt.Errorf("failed to get user %s: %w", id, err)
    }
    return user, nil
}

// Error handling middleware
func ErrorMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                log.Printf("panic: %v", err)
                http.Error(w, "Internal Server Error", http.StatusInternalServerError)
            }
        }()
        
        next.ServeHTTP(w, r)
    })
}

// Result type pattern
type Result[T any] struct {
    value T
    err   error
}

func Ok[T any](value T) Result[T] {
    return Result[T]{value: value}
}

func Err[T any](err error) Result[T] {
    return Result[T]{err: err}
}

func (r Result[T]) IsOk() bool {
    return r.err == nil
}

func (r Result[T]) Unwrap() (T, error) {
    return r.value, r.err
}

// Usage
func Divide(a, b float64) Result[float64] {
    if b == 0 {
        return Err[float64](errors.New("division by zero"))
    }
    return Ok(a / b)
}
```

### Context Pattern
```go
// Request-scoped values
type contextKey string

const (
    userIDKey     contextKey = "userID"
    requestIDKey  contextKey = "requestID"
    loggerKey     contextKey = "logger"
)

// Context helpers
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

// Middleware using context
func ContextMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        ctx := r.Context()
        
        // Add request ID
        requestID := r.Header.Get("X-Request-ID")
        if requestID == "" {
            requestID = generateRequestID()
        }
        ctx = WithRequestID(ctx, requestID)
        
        // Add user ID from auth
        if userID := extractUserID(r); userID != "" {
            ctx = WithUserID(ctx, userID)
        }
        
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// Service using context
type UserService struct {
    repo UserRepository
}

func (s *UserService) GetUserProfile(ctx context.Context) (*UserProfile, error) {
    userID, ok := GetUserID(ctx)
    if !ok {
        return nil, errors.New("user ID not found in context")
    }
    
    // Log with request ID
    if requestID, ok := GetRequestID(ctx); ok {
        log.Printf("[%s] Getting profile for user %s", requestID, userID)
    }
    
    return s.repo.GetProfile(ctx, userID)
}
```

## Best Practices

### 1. Interface Design
```go
// Small, focused interfaces (Interface Segregation Principle)
type Reader interface {
    Read([]byte) (int, error)
}

type Writer interface {
    Write([]byte) (int, error)
}

// Not this
type ReadWriter interface {
    Read([]byte) (int, error)
    Write([]byte) (int, error)
    Close() error
    Seek(int64, int) (int64, error)
    // Too many responsibilities
}

// Accept interfaces, return structs
func ProcessData(r Reader) (*Result, error) {
    // Implementation
    return &Result{}, nil
}
```

### 2. Error Handling
```go
// Always add context to errors
if err != nil {
    return fmt.Errorf("failed to process order %s: %w", orderID, err)
}

// Use error types for behavior
type TemporaryError interface {
    error
    Temporary() bool
}

// Check error behavior
if err, ok := err.(TemporaryError); ok && err.Temporary() {
    // Retry operation
}
```

### 3. Concurrency
```go
// Always pass context
func DoWork(ctx context.Context) error {
    select {
    case <-ctx.Done():
        return ctx.Err()
    case result := <-doExpensiveOperation():
        return processResult(result)
    }
}

// Prevent goroutine leaks
func StartWorker(ctx context.Context) {
    go func() {
        for {
            select {
            case <-ctx.Done():
                return // Goroutine can exit
            case work := <-workCh:
                process(work)
            }
        }
    }()
}
```

### 4. Testing
```go
// Use interfaces for testability
type Clock interface {
    Now() time.Time
}

type RealClock struct{}

func (c RealClock) Now() time.Time {
    return time.Now()
}

type MockClock struct {
    CurrentTime time.Time
}

func (c MockClock) Now() time.Time {
    return c.CurrentTime
}

// Service accepts clock interface
type Service struct {
    clock Clock
}
```

## Summary

- **Creational**: Control object creation (Singleton, Factory, Builder)
- **Structural**: Compose objects (Adapter, Decorator, Proxy)
- **Behavioral**: Object collaboration (Strategy, Observer, Command)
- **Concurrency**: Go-specific patterns (Worker Pool, Pipeline, Fan-In/Out)
- **Architectural**: System organization (Hexagonal, Clean, CQRS)
- **Go Idioms**: Language-specific patterns (Functional Options, Context)

Remember: Don't force patterns. Use them when they solve real problems and improve code maintainability.