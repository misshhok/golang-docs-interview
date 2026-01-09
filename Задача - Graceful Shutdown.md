# Задача - Graceful Shutdown

Graceful shutdown (корректное завершение) - правильная остановка приложения с завершением текущих операций и освобождением ресурсов. Критически важная тема для production систем!

## Зачем нужен Graceful Shutdown

### Проблема: Резкое завершение

```go
func main() {
    http.HandleFunc("/", handler)
    http.ListenAndServe(":8080", nil)
}
// При Ctrl+C или kill - сервер мгновенно останавливается
// Текущие запросы оборваны, данные могут быть потеряны
```

**Последствия:**
- Обрыв HTTP запросов в процессе обработки
- Незавершённые транзакции в БД
- Потеря данных в очередях
- Битые файлы при записи
- Разорванные соединения

### Решение: Graceful Shutdown

**Graceful shutdown:**
1. Получает сигнал завершения (SIGTERM, SIGINT)
2. Прекращает принимать новые запросы
3. Завершает текущие запросы
4. Закрывает соединения к БД, Redis, и т.д.
5. Освобождает ресурсы
6. Выходит

## Обработка OS сигналов

```go
package main

import (
    "fmt"
    "os"
    "os/signal"
    "syscall"
    "time"
)

func main() {
    // Создаём канал для сигналов
    stop := make(chan os.Signal, 1)

    // Регистрируем какие сигналы слушать
    signal.Notify(stop, syscall.SIGINT, syscall.SIGTERM)

    // Симуляция работы
    go func() {
        for {
            fmt.Println("Working...")
            time.Sleep(time.Second)
        }
    }()

    // Ждём сигнал
    <-stop
    fmt.Println("\nReceived shutdown signal, cleaning up...")

    // Cleanup
    time.Sleep(2 * time.Second)
    fmt.Println("Shutdown complete")
}
```

**Сигналы:**
- `SIGINT` (Ctrl+C) - прерывание с клавиатуры
- `SIGTERM` (kill) - запрос на завершение
- `SIGKILL` - принудительное завершение (не ловится!)

## Graceful Shutdown HTTP сервера

```go
package main

import (
    "context"
    "fmt"
    "log"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"
)

func handler(w http.ResponseWriter, r *http.Request) {
    // Симуляция долгого запроса
    time.Sleep(5 * time.Second)
    fmt.Fprintf(w, "Request completed!\n")
}

func main() {
    // Создаём сервер
    srv := &http.Server{
        Addr:    ":8080",
        Handler: http.HandlerFunc(handler),
    }

    // Запускаем сервер в горутине
    go func() {
        log.Println("Server starting on :8080")
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("Server error: %v", err)
        }
    }()

    // Канал для сигналов
    stop := make(chan os.Signal, 1)
    signal.Notify(stop, syscall.SIGINT, syscall.SIGTERM)

    // Ждём сигнал
    <-stop
    log.Println("Shutting down server...")

    // Даём 30 секунд на завершение текущих запросов
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := srv.Shutdown(ctx); err != nil {
        log.Printf("Server forced to shutdown: %v", err)
    }

    log.Println("Server stopped")
}
```

**Что происходит:**
1. `srv.Shutdown(ctx)` прекращает принимать новые запросы
2. Ждёт завершения текущих запросов
3. Если за 30 секунд не завершились → принудительно останавливает
4. Возвращает управление

**Тестирование:**
```bash
# Терминал 1
go run main.go

# Терминал 2
curl http://localhost:8080  # Запрос займёт 5 секунд

# Во время выполнения запроса в терминале 1
Ctrl+C

# Сервер дождётся завершения запроса перед остановкой
```

## Worker Pool с Graceful Shutdown

```go
package main

import (
    "context"
    "fmt"
    "os"
    "os/signal"
    "sync"
    "syscall"
    "time"
)

type WorkerPool struct {
    workers int
    jobs    chan int
    wg      sync.WaitGroup
    ctx     context.Context
    cancel  context.CancelFunc
}

func NewWorkerPool(workers int) *WorkerPool {
    ctx, cancel := context.WithCancel(context.Background())

    return &WorkerPool{
        workers: workers,
        jobs:    make(chan int, 100),
        ctx:     ctx,
        cancel:  cancel,
    }
}

func (wp *WorkerPool) Start() {
    for i := 1; i <= wp.workers; i++ {
        wp.wg.Add(1)
        go wp.worker(i)
    }
}

func (wp *WorkerPool) worker(id int) {
    defer wp.wg.Done()
    fmt.Printf("Worker %d started\n", id)

    for {
        select {
        case job, ok := <-wp.jobs:
            if !ok {
                fmt.Printf("Worker %d: jobs channel closed\n", id)
                return
            }

            fmt.Printf("Worker %d processing job %d\n", id, job)
            time.Sleep(2 * time.Second)  // Симуляция работы
            fmt.Printf("Worker %d completed job %d\n", id, job)

        case <-wp.ctx.Done():
            fmt.Printf("Worker %d: received shutdown signal\n", id)
            return
        }
    }
}

func (wp *WorkerPool) Submit(job int) error {
    select {
    case wp.jobs <- job:
        return nil
    case <-wp.ctx.Done():
        return fmt.Errorf("pool is shutting down")
    }
}

// Graceful shutdown: завершаем текущие задачи
func (wp *WorkerPool) Shutdown() {
    fmt.Println("Initiating graceful shutdown...")

    close(wp.jobs)  // Прекращаем принимать новые задачи
    wp.wg.Wait()    // Ждём завершения текущих

    fmt.Println("All workers finished")
}

// Immediate shutdown: останавливаем немедленно
func (wp *WorkerPool) ShutdownNow() {
    fmt.Println("Initiating immediate shutdown...")

    wp.cancel()   // Отменяем контекст
    wp.wg.Wait()  // Ждём выхода workers

    fmt.Println("All workers stopped")
}

func main() {
    pool := NewWorkerPool(3)
    pool.Start()

    // Добавляем задачи
    for i := 1; i <= 10; i++ {
        pool.Submit(i)
    }

    // Слушаем сигналы
    stop := make(chan os.Signal, 1)
    signal.Notify(stop, syscall.SIGINT, syscall.SIGTERM)

    <-stop

    // Graceful shutdown
    pool.Shutdown()

    // Для immediate используйте:
    // pool.ShutdownNow()
}
```

## Shutdown с таймаутом

```go
package main

import (
    "context"
    "fmt"
    "os"
    "os/signal"
    "sync"
    "syscall"
    "time"
)

type Application struct {
    workers int
    jobs    chan int
    wg      sync.WaitGroup
}

func NewApplication(workers int) *Application {
    return &Application{
        workers: workers,
        jobs:    make(chan int, 100),
    }
}

func (app *Application) Start() {
    for i := 1; i <= app.workers; i++ {
        app.wg.Add(1)
        go app.worker(i)
    }
}

func (app *Application) worker(id int) {
    defer app.wg.Done()

    for job := range app.jobs {
        fmt.Printf("Worker %d processing job %d\n", id, job)
        time.Sleep(2 * time.Second)
        fmt.Printf("Worker %d completed job %d\n", id, job)
    }
}

func (app *Application) Submit(job int) {
    app.jobs <- job
}

// Shutdown с таймаутом
func (app *Application) Shutdown(timeout time.Duration) error {
    fmt.Println("Shutting down...")

    close(app.jobs)

    // Канал для сигнала завершения
    done := make(chan struct{})

    go func() {
        app.wg.Wait()
        close(done)
    }()

    // Ждём либо завершения, либо таймаута
    select {
    case <-done:
        fmt.Println("Shutdown completed gracefully")
        return nil

    case <-time.After(timeout):
        return fmt.Errorf("shutdown timeout exceeded")
    }
}

func main() {
    app := NewApplication(3)
    app.Start()

    // Добавляем задачи
    for i := 1; i <= 5; i++ {
        app.Submit(i)
    }

    // Ждём сигнал
    stop := make(chan os.Signal, 1)
    signal.Notify(stop, syscall.SIGINT, syscall.SIGTERM)
    <-stop

    // Shutdown с таймаутом 10 секунд
    if err := app.Shutdown(10 * time.Second); err != nil {
        fmt.Printf("Shutdown error: %v\n", err)
        os.Exit(1)
    }
}
```

## Полный пример: Приложение с БД и HTTP

```go
package main

import (
    "context"
    "database/sql"
    "fmt"
    "log"
    "net/http"
    "os"
    "os/signal"
    "sync"
    "syscall"
    "time"

    _ "github.com/lib/pq"
)

type App struct {
    server *http.Server
    db     *sql.DB
    wg     sync.WaitGroup
}

func NewApp() *App {
    return &App{}
}

// Инициализация компонентов
func (app *App) Initialize() error {
    // БД
    var err error
    app.db, err = sql.Open("postgres", "connection_string")
    if err != nil {
        return fmt.Errorf("db connection: %w", err)
    }

    // HTTP сервер
    mux := http.NewServeMux()
    mux.HandleFunc("/", app.handler)

    app.server = &http.Server{
        Addr:    ":8080",
        Handler: mux,
    }

    return nil
}

func (app *App) handler(w http.ResponseWriter, r *http.Request) {
    // Работа с БД
    ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
    defer cancel()

    var count int
    err := app.db.QueryRowContext(ctx, "SELECT COUNT(*) FROM users").Scan(&count)
    if err != nil {
        http.Error(w, "Database error", http.StatusInternalServerError)
        return
    }

    fmt.Fprintf(w, "Users count: %d\n", count)
}

// Запуск приложения
func (app *App) Start() {
    // HTTP сервер
    app.wg.Add(1)
    go func() {
        defer app.wg.Done()

        log.Println("Server starting on :8080")
        if err := app.server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Printf("Server error: %v", err)
        }
    }()
}

// Graceful shutdown
func (app *App) Shutdown() error {
    log.Println("Starting graceful shutdown...")

    // Контекст с таймаутом
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    // 1. Останавливаем HTTP сервер
    log.Println("Shutting down HTTP server...")
    if err := app.server.Shutdown(ctx); err != nil {
        log.Printf("HTTP server shutdown error: %v", err)
    }

    // 2. Ждём завершения горутин
    done := make(chan struct{})
    go func() {
        app.wg.Wait()
        close(done)
    }()

    select {
    case <-done:
        log.Println("All goroutines finished")
    case <-ctx.Done():
        log.Println("Shutdown timeout exceeded")
    }

    // 3. Закрываем БД
    log.Println("Closing database connections...")
    if err := app.db.Close(); err != nil {
        log.Printf("Database close error: %v", err)
        return err
    }

    log.Println("Shutdown complete")
    return nil
}

func main() {
    app := NewApp()

    if err := app.Initialize(); err != nil {
        log.Fatalf("Initialization failed: %v", err)
    }

    app.Start()

    // Слушаем сигналы
    stop := make(chan os.Signal, 1)
    signal.Notify(stop, syscall.SIGINT, syscall.SIGTERM)

    <-stop

    if err := app.Shutdown(); err != nil {
        os.Exit(1)
    }
}
```

## Best Practices

1. ✅ **Всегда обрабатывайте SIGINT и SIGTERM**
   ```go
   signal.Notify(stop, syscall.SIGINT, syscall.SIGTERM)
   ```

2. ✅ **Используйте таймауты**
   ```go
   ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
   defer cancel()
   ```

3. ✅ **Закрывайте ресурсы в правильном порядке**
   ```
   1. Прекратить принимать новые запросы
   2. Завершить текущие операции
   3. Закрыть соединения (БД, Redis, Kafka)
   4. Освободить ресурсы
   ```

4. ✅ **Логируйте этапы shutdown**
   ```go
   log.Println("Shutting down server...")
   log.Println("Closing database...")
   log.Println("Shutdown complete")
   ```

5. ✅ **Используйте context для propagation**
   ```go
   ctx, cancel := context.WithCancel(context.Background())
   // Передайте ctx во все компоненты
   ```

6. ✅ **Буферизируйте канал сигналов**
   ```go
   stop := make(chan os.Signal, 1)  // Буфер 1
   ```

7. ❌ **SIGKILL не ловится!**
   ```bash
   kill -9 pid  # Мгновенная остановка, обработчик не сработает
   kill pid     # SIGTERM, graceful shutdown сработает
   ```

8. ✅ **Тестируйте shutdown в production-like окружении**
   ```go
   // Добавьте metrics
   shutdownDuration := time.Since(start)
   log.Printf("Shutdown took %v", shutdownDuration)
   ```

## Вопросы с собеседований

**Вопрос 1:** Что такое graceful shutdown и зачем он нужен?

**Ответ:** Graceful shutdown - корректное завершение приложения с завершением текущих операций перед выходом. Нужен чтобы:
- Не потерять данные
- Завершить текущие запросы
- Корректно закрыть соединения
- Не оборвать транзакции

**Вопрос 2:** Как в Go остановить HTTP сервер с graceful shutdown?

**Ответ:**
```go
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()

if err := srv.Shutdown(ctx); err != nil {
    log.Printf("Server shutdown error: %v", err)
}
```

`Shutdown()` прекращает принимать новые соединения и ждёт завершения текущих запросов.

**Вопрос 3:** Как обработать сигнал завершения от ОС?

**Ответ:**
```go
stop := make(chan os.Signal, 1)
signal.Notify(stop, syscall.SIGINT, syscall.SIGTERM)

<-stop  // Блокируется пока не придёт сигнал
```

**Вопрос 4:** В каком порядке закрывать ресурсы?

**Ответ:**
1. Прекратить принимать новые запросы (close канал, srv.Shutdown)
2. Дождаться завершения текущих операций (WaitGroup)
3. Закрыть внешние соединения (БД, Redis, Kafka)
4. Освободить остальные ресурсы

## Где спрашивают

- Яндекс
- Авито
- Т-Банк
- Озон
- VK
- Любые компании с production системами

## Связанные темы

- [[Go - Context]]
- [[Go - WaitGroup и Once]]
- [[Задача - Worker Pool]]
- [[Go - Пакет net-http]]
- [[Go - Defer, Panic, Recover]]
