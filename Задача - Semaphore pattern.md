# Задача - Semaphore pattern

Semaphore (семафор) - паттерн для ограничения количества одновременно выполняющихся операций. В Go реализуется через буферизированный канал.

## Что такое Semaphore

**Semaphore** - механизм синхронизации, ограничивающий количество потоков, которые могут одновременно получить доступ к ресурсу.

**Классический semaphore** имеет две операции:
- **Acquire (P)** - захватить слот, заблокироваться если все слоты заняты
- **Release (V)** - освободить слот

**В Go:** Используем буферизированный канал как semaphore.

## Базовая реализация

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    // Семафор с 3 слотами (максимум 3 одновременных операции)
    sem := make(chan struct{}, 3)

    for i := 1; i <= 10; i++ {
        // Acquire: занимаем слот
        sem <- struct{}{}

        go func(id int) {
            // Release: освобождаем слот при выходе
            defer func() { <-sem }()

            fmt.Printf("Task %d started\n", id)
            time.Sleep(2 * time.Second)  // Симуляция работы
            fmt.Printf("Task %d finished\n", id)
        }(i)
    }

    // Ждём пока все задачи завершатся
    for i := 0; i < cap(sem); i++ {
        sem <- struct{}{}
    }

    fmt.Println("All tasks completed")
}
```

**Как работает:**
- Буферизированный канал размером 3
- Первые 3 `sem <- struct{}{}` не блокируются
- 4-я попытка блокируется пока кто-то не освободит слот (`<-sem`)
- Максимум 3 горутины работают одновременно

## Semaphore для ограничения HTTP запросов

```go
package main

import (
    "fmt"
    "io"
    "net/http"
    "sync"
    "time"
)

func fetchURL(url string, sem chan struct{}, wg *sync.WaitGroup) {
    defer wg.Done()

    // Acquire
    sem <- struct{}{}
    defer func() { <-sem }()  // Release

    fmt.Printf("Fetching %s\n", url)

    client := &http.Client{Timeout: 10 * time.Second}
    resp, err := client.Get(url)
    if err != nil {
        fmt.Printf("Error fetching %s: %v\n", url, err)
        return
    }
    defer resp.Body.Close()

    body, _ := io.ReadAll(resp.Body)
    fmt.Printf("Fetched %s: %d bytes\n", url, len(body))
}

func main() {
    urls := []string{
        "https://golang.org",
        "https://github.com",
        "https://stackoverflow.com",
        "https://reddit.com",
        "https://news.ycombinator.com",
        "https://medium.com",
        "https://dev.to",
        "https://habr.com",
    }

    // Ограничение: максимум 3 одновременных запроса
    sem := make(chan struct{}, 3)
    var wg sync.WaitGroup

    for _, url := range urls {
        wg.Add(1)
        go fetchURL(url, sem, &wg)
    }

    wg.Wait()
    fmt.Println("All URLs fetched")
}
```

**Применение:**
- API с rate limits
- Ограничение нагрузки на внешние сервисы
- Предотвращение исчерпания ресурсов (сокеты, файловые дескрипторы)

## Semaphore со встроенной библиотекой

С Go 1.13+ есть пакет `golang.org/x/sync/semaphore`:

```go
package main

import (
    "context"
    "fmt"
    "golang.org/x/sync/semaphore"
    "sync"
    "time"
)

func main() {
    // Семафор с весом 3
    sem := semaphore.NewWeighted(3)

    var wg sync.WaitGroup

    for i := 1; i <= 10; i++ {
        wg.Add(1)

        go func(id int) {
            defer wg.Done()

            // Acquire с весом 1
            if err := sem.Acquire(context.Background(), 1); err != nil {
                fmt.Printf("Failed to acquire: %v\n", err)
                return
            }
            defer sem.Release(1)

            fmt.Printf("Task %d started\n", id)
            time.Sleep(time.Second)
            fmt.Printf("Task %d finished\n", id)
        }(i)
    }

    wg.Wait()
}
```

**Преимущества:**
- Поддержка context (cancellation, timeout)
- Weighted semaphore (разный вес операций)
- TryAcquire (неблокирующая попытка)

## Weighted Semaphore (разный вес)

```go
package main

import (
    "context"
    "fmt"
    "golang.org/x/sync/semaphore"
    "time"
)

func main() {
    // Семафор с общим весом 10
    sem := semaphore.NewWeighted(10)

    // Маленькая задача (вес 1)
    go func() {
        sem.Acquire(context.Background(), 1)
        defer sem.Release(1)

        fmt.Println("Small task running")
        time.Sleep(time.Second)
    }()

    // Большая задача (вес 5)
    go func() {
        sem.Acquire(context.Background(), 5)
        defer sem.Release(5)

        fmt.Println("Large task running")
        time.Sleep(time.Second)
    }()

    // Ещё одна большая задача
    go func() {
        sem.Acquire(context.Background(), 5)
        defer sem.Release(5)

        fmt.Println("Another large task running")
        time.Sleep(time.Second)
    }()

    time.Sleep(3 * time.Second)
}
```

**Применение:** Когда задачи потребляют разное количество ресурсов.

## Semaphore с таймаутом

```go
package main

import (
    "context"
    "fmt"
    "golang.org/x/sync/semaphore"
    "time"
)

func tryAcquire(sem *semaphore.Weighted, id int) {
    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel()

    fmt.Printf("Task %d: trying to acquire\n", id)

    if err := sem.Acquire(ctx, 1); err != nil {
        fmt.Printf("Task %d: timeout! %v\n", id, err)
        return
    }
    defer sem.Release(1)

    fmt.Printf("Task %d: acquired, working...\n", id)
    time.Sleep(5 * time.Second)
    fmt.Printf("Task %d: done\n", id)
}

func main() {
    sem := semaphore.NewWeighted(2)

    for i := 1; i <= 5; i++ {
        go tryAcquire(sem, i)
        time.Sleep(100 * time.Millisecond)
    }

    time.Sleep(10 * time.Second)
}
```

**Вывод:**
```
Task 1: trying to acquire
Task 1: acquired, working...
Task 2: trying to acquire
Task 2: acquired, working...
Task 3: trying to acquire
Task 3: timeout! context deadline exceeded
Task 4: trying to acquire
Task 4: timeout! context deadline exceeded
Task 5: trying to acquire
Task 5: timeout! context deadline exceeded
```

Задачи 3, 4, 5 не смогли получить слот за 2 секунды.

## TryAcquire (неблокирующая попытка)

```go
package main

import (
    "fmt"
    "golang.org/x/sync/semaphore"
    "time"
)

func main() {
    sem := semaphore.NewWeighted(2)

    // Заполняем семафор
    sem.Acquire(context.Background(), 2)

    // TryAcquire не блокируется
    if sem.TryAcquire(1) {
        fmt.Println("Acquired!")
        sem.Release(1)
    } else {
        fmt.Println("Failed to acquire (semaphore full)")
    }

    // Освобождаем
    sem.Release(2)

    // Теперь TryAcquire успешен
    if sem.TryAcquire(1) {
        fmt.Println("Acquired after release!")
        sem.Release(1)
    }
}
```

**Применение:** Когда нужно попробовать захватить без блокировки.

## Реальный пример: Ограничение конкурентных DB запросов

```go
package main

import (
    "context"
    "database/sql"
    "fmt"
    "golang.org/x/sync/semaphore"
    "log"
    "sync"
    "time"

    _ "github.com/lib/pq"
)

type DB struct {
    conn *sql.DB
    sem  *semaphore.Weighted
}

func NewDB(connStr string, maxConcurrent int64) (*DB, error) {
    conn, err := sql.Open("postgres", connStr)
    if err != nil {
        return nil, err
    }

    return &DB{
        conn: conn,
        sem:  semaphore.NewWeighted(maxConcurrent),
    }, nil
}

// Query с ограничением конкурентности
func (db *DB) Query(ctx context.Context, query string) (*sql.Rows, error) {
    // Acquire семафор
    if err := db.sem.Acquire(ctx, 1); err != nil {
        return nil, fmt.Errorf("semaphore acquire: %w", err)
    }
    defer db.sem.Release(1)

    return db.conn.QueryContext(ctx, query)
}

func main() {
    db, err := NewDB("postgres://...", 10)  // Максимум 10 одновременных запросов
    if err != nil {
        log.Fatal(err)
    }
    defer db.conn.Close()

    var wg sync.WaitGroup

    // Запускаем 100 запросов, но максимум 10 одновременно
    for i := 0; i < 100; i++ {
        wg.Add(1)

        go func(id int) {
            defer wg.Done()

            ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
            defer cancel()

            rows, err := db.Query(ctx, "SELECT * FROM users LIMIT 1")
            if err != nil {
                log.Printf("Query %d error: %v", id, err)
                return
            }
            defer rows.Close()

            fmt.Printf("Query %d completed\n", id)
        }(i)
    }

    wg.Wait()
}
```

## Вопрос 1: В чём разница между Semaphore и Worker Pool?

<details>
<summary>Ответ</summary>

| Semaphore | Worker Pool |
|-----------|-------------|
| Ограничивает количество **одновременных операций** | Фиксированное количество **горутин-workers** |
| Горутины создаются и удаляются | Горутины переиспользуются |
| Нет очереди задач | Есть канал задач (очередь) |
| Проще реализация | Сложнее реализация |
| Подходит для независимых операций | Подходит для обработки потока задач |

**Semaphore:**
```go
sem := make(chan struct{}, 10)

for _, url := range urls {
    sem <- struct{}{}
    go func(url string) {
        defer func() { <-sem }()
        fetch(url)
    }(url)
}
```

**Worker Pool:**
```go
jobs := make(chan string, 100)
results := make(chan Result, 100)

for i := 0; i < 10; i++ {
    go worker(jobs, results)
}

for _, url := range urls {
    jobs <- url
}
```

**Когда использовать:**
- Semaphore → простое ограничение параллелизма
- Worker Pool → управление очередью, сложная логика

</details>

## Вопрос 2: Реализуйте динамический Semaphore

Нужен semaphore, у которого можно изменять лимит в runtime.

<details>
<summary>Решение</summary>

```go
package main

import (
    "context"
    "fmt"
    "sync"
    "time"
)

type DynamicSemaphore struct {
    current int
    max     int
    mu      sync.Mutex
    cond    *sync.Cond
}

func NewDynamicSemaphore(max int) *DynamicSemaphore {
    s := &DynamicSemaphore{
        max: max,
    }
    s.cond = sync.NewCond(&s.mu)
    return s
}

func (s *DynamicSemaphore) Acquire() {
    s.mu.Lock()
    defer s.mu.Unlock()

    for s.current >= s.max {
        s.cond.Wait()  // Ждём пока освободится слот
    }

    s.current++
}

func (s *DynamicSemaphore) Release() {
    s.mu.Lock()
    defer s.mu.Unlock()

    s.current--
    s.cond.Signal()  // Будим одну ожидающую горутину
}

func (s *DynamicSemaphore) SetMax(newMax int) {
    s.mu.Lock()
    defer s.mu.Unlock()

    s.max = newMax
    s.cond.Broadcast()  // Будим все ожидающие горутины
}

func (s *DynamicSemaphore) GetStats() (current, max int) {
    s.mu.Lock()
    defer s.mu.Unlock()
    return s.current, s.max
}

func main() {
    sem := NewDynamicSemaphore(3)

    for i := 1; i <= 10; i++ {
        go func(id int) {
            sem.Acquire()
            defer sem.Release()

            c, m := sem.GetStats()
            fmt.Printf("Task %d running (current: %d, max: %d)\n", id, c, m)
            time.Sleep(2 * time.Second)
        }(i)
    }

    // Через 5 секунд увеличиваем лимит
    time.Sleep(5 * time.Second)
    fmt.Println("Increasing limit to 5")
    sem.SetMax(5)

    time.Sleep(10 * time.Second)
}
```

</details>

## Best Practices

1. ✅ **Используйте defer для Release**
   ```go
   sem <- struct{}{}
   defer func() { <-sem }()
   ```

2. ✅ **Выбирайте правильный размер**
   ```go
   // CPU-bound: количество CPU
   sem := make(chan struct{}, runtime.NumCPU())

   // I/O-bound: больше (тестируйте)
   sem := make(chan struct{}, 100)

   // Rate limit API: по ограничениям API
   sem := make(chan struct{}, 10)  // 10 req/sec
   ```

3. ✅ **Используйте context для cancellation**
   ```go
   if err := sem.Acquire(ctx, 1); err != nil {
       return err
   }
   defer sem.Release(1)
   ```

4. ✅ **Используйте `golang.org/x/sync/semaphore` для продакшена**
   - Поддержка context
   - Weighted semaphore
   - TryAcquire

5. ✅ **Обрабатывайте таймауты**
   ```go
   ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
   defer cancel()

   if err := sem.Acquire(ctx, 1); err != nil {
       // Timeout или cancellation
   }
   ```

6. ❌ **Не забывайте освобождать**
   ```go
   sem <- struct{}{}
   // Забыли <-sem → goroutine leak
   ```

7. ❌ **Не делайте Release без Acquire**
   ```go
   <-sem  // Panic если семафор пуст!
   ```

## Где спрашивают

- Яндекс
- Авито
- VK
- Т-Банк

## Связанные темы

- [[Задача - Worker Pool]]
- [[Go - Каналы (channels)]]
- [[Go - Context]]
- [[Задача - Rate Limiter]]
