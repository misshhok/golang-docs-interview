# Go - Context

`context.Context` - стандартный механизм для передачи deadlines, cancellation сигналов и request-scoped значений через границы API и между [[Go - Горутины (goroutines)|горутинами]].

## Зачем нужен Context

**Проблемы без context:**
- ❌ Невозможно отменить длинные операции
- ❌ Нет способа установить таймауты для операций
- ❌ Горутины могут "утекать" (goroutine leaks)
- ❌ Нет способа передать request-specific данные

**Что решает context:**
- ✅ Cancellation - отмена операций
- ✅ Deadlines - автоматическая отмена по времени
- ✅ Values - передача request-scoped данных
- ✅ Graceful shutdown

## Context interface

```go
type Context interface {
    // Deadline возвращает время когда context будет отменен
    Deadline() (deadline time.Time, ok bool)

    // Done возвращает канал который закрывается при отмене
    Done() <-chan struct{}

    // Err возвращает ошибку почему context был отменен
    Err() error

    // Value возвращает значение по ключу
    Value(key interface{}) interface{}
}
```

**Важно**: Context **иммутабельный** - каждое изменение создает новый context!

## Создание Context

### 1. context.Background()

Корневой context, никогда не отменяется.

```go
func main() {
    ctx := context.Background()
    // Используется как корень для других contexts
}
```

**Когда использовать:**
- В `main()` функции
- В тестах
- Как корневой context для incoming requests

### 2. context.TODO()

Placeholder когда неясно какой context использовать.

```go
func processData() {
    ctx := context.TODO() // Временное решение
    doWork(ctx)
}
```

**Когда использовать:**
- Временно, когда рефакторите код
- Когда неясен правильный context
- Лучше использовать `Background()` в production

### 3. context.WithCancel()

Context который можно отменить вручную.

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel() // Всегда вызывайте cancel!

    go func() {
        for {
            select {
            case <-ctx.Done():
                fmt.Println("Worker cancelled:", ctx.Err())
                return
            default:
                fmt.Println("Working...")
                time.Sleep(500 * time.Millisecond)
            }
        }
    }()

    time.Sleep(2 * time.Second)
    cancel() // Отмена
    time.Sleep(1 * time.Second)
}
```

**Всегда вызывайте `defer cancel()`** даже если не планируете отменять - это освобождает ресурсы!

### 4. context.WithTimeout()

Автоматическая отмена через заданное время.

```go
func fetchWithTimeout(url string) (string, error) {
    // Отмена через 5 секунд
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return "", err
    }

    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return "", err // Может быть context.DeadlineExceeded
    }
    defer resp.Body.Close()

    // Читаем response
    body, err := io.ReadAll(resp.Body)
    return string(body), err
}
```

### 5. context.WithDeadline()

Отмена в конкретное время.

```go
func processUntil(deadline time.Time) {
    ctx, cancel := context.WithDeadline(context.Background(), deadline)
    defer cancel()

    select {
    case <-time.After(1 * time.Hour):
        fmt.Println("Work completed")
    case <-ctx.Done():
        fmt.Println("Deadline exceeded:", ctx.Err())
    }
}

func main() {
    deadline := time.Now().Add(5 * time.Second)
    processUntil(deadline)
}
```

### 6. context.WithValue()

Передача request-scoped данных.

```go
type key string

const requestIDKey key = "requestID"

func withRequestID(ctx context.Context, id string) context.Context {
    return context.WithValue(ctx, requestIDKey, id)
}

func getRequestID(ctx context.Context) string {
    if id, ok := ctx.Value(requestIDKey).(string); ok {
        return id
    }
    return ""
}

func handler(w http.ResponseWriter, r *http.Request) {
    requestID := generateID()
    ctx := withRequestID(r.Context(), requestID)

    processRequest(ctx)
}

func processRequest(ctx context.Context) {
    id := getRequestID(ctx)
    log.Printf("Processing request %s", id)
}
```

## Паттерны использования

### 1. HTTP Server с graceful shutdown

```go
func main() {
    srv := &http.Server{Addr: ":8080"}

    // Обработка сигналов
    stop := make(chan os.Signal, 1)
    signal.Notify(stop, os.Interrupt, syscall.SIGTERM)

    go func() {
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatal(err)
        }
    }()

    <-stop // Ждем сигнал
    log.Println("Shutting down server...")

    // Graceful shutdown с таймаутом
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := srv.Shutdown(ctx); err != nil {
        log.Fatal("Server forced to shutdown:", err)
    }

    log.Println("Server exited")
}
```

### 2. Worker Pool с cancellation

```go
func workerPool(ctx context.Context, jobs <-chan Job) {
    var wg sync.WaitGroup

    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()

            for {
                select {
                case <-ctx.Done():
                    fmt.Printf("Worker %d stopped\n", id)
                    return
                case job, ok := <-jobs:
                    if !ok {
                        return
                    }
                    process(job)
                }
            }
        }(i)
    }

    wg.Wait()
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    jobs := make(chan Job, 100)

    go workerPool(ctx, jobs)

    // Отправляем задачи...

    cancel() // Отмена всех workers
    time.Sleep(time.Second)
}
```

### 3. Database запросы с timeout

```go
func getUserFromDB(ctx context.Context, userID int) (*User, error) {
    // Context автоматически отменяет запрос при timeout
    query := "SELECT * FROM users WHERE id = $1"

    var user User
    err := db.QueryRowContext(ctx, query, userID).Scan(
        &user.ID,
        &user.Name,
        &user.Email,
    )

    if err != nil {
        return nil, err
    }

    return &user, nil
}

func handler(w http.ResponseWriter, r *http.Request) {
    // Timeout 3 секунды для DB запроса
    ctx, cancel := context.WithTimeout(r.Context(), 3*time.Second)
    defer cancel()

    user, err := getUserFromDB(ctx, 123)
    if err != nil {
        if err == context.DeadlineExceeded {
            http.Error(w, "Request timeout", http.StatusRequestTimeout)
            return
        }
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    json.NewEncoder(w).Encode(user)
}
```

### 4. Каскадная отмена (Context chaining)

```go
func operation(ctx context.Context) error {
    // Создаем child context с более коротким timeout
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    return subOperation(ctx)
}

func subOperation(ctx context.Context) error {
    select {
    case <-time.After(10 * time.Second):
        return nil
    case <-ctx.Done():
        return ctx.Err() // Отменен родительским context
    }
}
```

Отмена parent context **автоматически отменяет** child contexts!

### 5. Request-scoped logging

```go
type contextKey string

const loggerKey contextKey = "logger"

func withLogger(ctx context.Context, logger *log.Logger) context.Context {
    return context.WithValue(ctx, loggerKey, logger)
}

func getLogger(ctx context.Context) *log.Logger {
    if logger, ok := ctx.Value(loggerKey).(*log.Logger); ok {
        return logger
    }
    return log.Default()
}

func middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        requestID := generateID()
        logger := log.New(os.Stdout, fmt.Sprintf("[%s] ", requestID), log.LstdFlags)

        ctx := withLogger(r.Context(), logger)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

func handler(w http.ResponseWriter, r *http.Request) {
    logger := getLogger(r.Context())
    logger.Println("Handling request")
}
```

### 6. Fan-Out с context

```go
func fanOut(ctx context.Context, urls []string) []Result {
    results := make(chan Result, len(urls))
    var wg sync.WaitGroup

    for _, url := range urls {
        wg.Add(1)
        go func(u string) {
            defer wg.Done()

            select {
            case <-ctx.Done():
                results <- Result{URL: u, Err: ctx.Err()}
            case results <- fetch(ctx, u):
            }
        }(url)
    }

    go func() {
        wg.Wait()
        close(results)
    }()

    // Собираем результаты
    var out []Result
    for r := range results {
        out = append(out, r)
    }

    return out
}
```

## Ошибки Context

### context.Canceled

```go
err := ctx.Err()
if errors.Is(err, context.Canceled) {
    // Context был отменен через cancel()
}
```

### context.DeadlineExceeded

```go
err := ctx.Err()
if errors.Is(err, context.DeadlineExceeded) {
    // Превышен timeout или deadline
}
```

## Типичные ошибки

### 1. Забыть вызвать cancel()

```go
// ❌ Утечка ресурсов
func leak() {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    // Забыли defer cancel()!
    doWork(ctx)
} // cancel никогда не вызовется!

// ✅ Всегда используйте defer
func noLeak() {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel() // Освобождает ресурсы
    doWork(ctx)
}
```

### 2. Хранить context в структурах

```go
// ❌ НЕ храните context в структурах
type Server struct {
    ctx context.Context // Плохо!
}

// ✅ Передавайте context как первый параметр
type Server struct {
    // ...
}

func (s *Server) Process(ctx context.Context, data Data) error {
    // Хорошо
}
```

**Исключение**: Если структура существует только в рамках одного request.

### 3. Использовать nil context

```go
// ❌ Nil context
doWork(nil) // Panic!

// ✅ Используйте Background или TODO
doWork(context.Background())
```

### 4. Неправильный ключ для WithValue

```go
// ❌ Использование string напрямую - конфликты!
ctx := context.WithValue(ctx, "userID", 123)

// ✅ Используйте приватный тип
type contextKey string
const userIDKey contextKey = "userID"

ctx := context.WithValue(ctx, userIDKey, 123)
```

### 5. Сохранение данных в WithValue

```go
// ❌ Не используйте для обязательных параметров
func processUser(ctx context.Context) {
    userID := ctx.Value(userIDKey).(int) // Может panic!
    // ...
}

// ✅ Обязательные параметры - явные аргументы
func processUser(ctx context.Context, userID int) {
    // ...
}

// ✅ WithValue только для request-scoped данных
// - Request IDs
// - Authentication tokens
// - Loggers
```

### 6. Игнорирование ctx.Done()

```go
// ❌ Игнорирование отмены
func longOperation(ctx context.Context) {
    for i := 0; i < 1000000; i++ {
        // Долгая работа без проверки ctx.Done()
        heavyComputation(i)
    }
}

// ✅ Проверка ctx.Done() в цикле
func longOperation(ctx context.Context) error {
    for i := 0; i < 1000000; i++ {
        select {
        case <-ctx.Done():
            return ctx.Err()
        default:
            heavyComputation(i)
        }
    }
    return nil
}
```

### 7. Создание context в неправильном месте

```go
// ❌ Context создан слишком глубоко
func handler(w http.ResponseWriter, r *http.Request) {
    doWork()
}

func doWork() {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    // Слишком поздно, не учитывает таймаут всего request
}

// ✅ Context от http.Request
func handler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context() // Request context
    doWork(ctx)
}

func doWork(ctx context.Context) {
    // Работает с переданным context
}
```

## Best Practices

### 1. Context всегда первый параметр

```go
// ✅ Правильно
func DoWork(ctx context.Context, arg1 string, arg2 int) error

// ❌ Неправильно
func DoWork(arg1 string, ctx context.Context, arg2 int) error
```

### 2. Называйте параметр "ctx"

```go
// ✅ Стандартное именование
func Process(ctx context.Context) error

// ❌ Нестандартное
func Process(context context.Context) error
func Process(c context.Context) error
```

### 3. Не храните context в структурах

```go
// ❌ Плохо
type Request struct {
    ctx  context.Context
    data Data
}

// ✅ Хорошо - передавайте явно
type Request struct {
    data Data
}

func (r *Request) Process(ctx context.Context) error
```

### 4. Используйте приватные типы для ключей

```go
// ✅ Правильно
type contextKey string

const (
    userIDKey   contextKey = "userID"
    requestIDKey contextKey = "requestID"
)

// ❌ Неправильно - риск конфликтов
const userIDKey = "userID"
```

### 5. WithValue только для request-scoped данных

```go
// ✅ Хорошие кандидаты для WithValue:
// - Request ID
// - Authentication token
// - Request-specific logger
// - Trace ID

// ❌ НЕ используйте для:
// - Обязательных параметров функции
// - Опциональных параметров
// - Configuration
```

### 6. Всегда проверяйте ctx.Done()

```go
// ✅ В долгих операциях
func process(ctx context.Context, items []Item) error {
    for _, item := range items {
        select {
        case <-ctx.Done():
            return ctx.Err()
        default:
            if err := processItem(item); err != nil {
                return err
            }
        }
    }
    return nil
}
```

### 7. Используйте context в стандартной библиотеке

```go
// ✅ HTTP requests
req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)

// ✅ Database queries
db.QueryContext(ctx, query, args...)

// ✅ Command execution
cmd := exec.CommandContext(ctx, "ls", "-la")
```

## Context vs Каналы

**Context предпочтительнее когда:**
- ✅ Нужно распространить cancellation по call stack
- ✅ Работаете с standard library (database, http)
- ✅ Нужен deadline/timeout
- ✅ Request-scoped данные

**Каналы [[Go - Каналы (channels)]] предпочтительнее когда:**
- ✅ Передача данных между горутинами
- ✅ Fan-out/fan-in паттерны
- ✅ Pipeline обработка
- ✅ Ownership transfer

**Можно комбинировать:**
```go
func worker(ctx context.Context, jobs <-chan Job) {
    for {
        select {
        case <-ctx.Done():
            return
        case job, ok := <-jobs:
            if !ok {
                return
            }
            process(job)
        }
    }
}
```

## Производительность

**Стоимость операций:**
- `Background()`: ~0 ns (singleton)
- `WithCancel()`: ~200-300 ns
- `WithTimeout()`: ~300-400 ns
- `WithValue()`: ~50-100 ns per value
- `Done()` check: ~1-2 ns
- `Value()` lookup: ~10-20 ns per level

**Советы:**
- Не создавайте context в hot loops
- Не используйте много вложенных WithValue
- Done() канал очень эффективен в select

## Связанные темы

- [[Go - Горутины (goroutines)]] - управление жизненным циклом горутин
- [[Go - Каналы (channels)]] - альтернатива для некоторых use cases
- [[Go - Select statement]] - использование ctx.Done() в select
- [[Go - Пакет sync]] - другие примитивы синхронизации
- [[Go - WaitGroup и Once]] - координация завершения
- [[Go - Пакет net-http]] - использование context в HTTP
