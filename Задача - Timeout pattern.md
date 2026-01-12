# Задача - Timeout pattern

Timeout (таймаут) - ограничение времени выполнения операции. Критически важен для надёжных систем, чтобы не зависать навсегда.

## Зачем нужны таймауты

### Проблема: Операция может зависнуть

```go
// ❌ Может зависнуть навсегда
resp, err := http.Get("https://slow-api.com")
```

**Проблемы без таймаута:**
- HTTP запрос к медленному серверу
- Чтение из канала, который никто не заполняет
- Запрос к БД на медленном запросе
- Deadlock
- Исчерпание ресурсов (все горутины заблокированы)

### Решение: Timeout

**Timeout гарантирует:**
- Операция завершится (успехом или ошибкой) за заданное время
- Система не зависнет
- Ресурсы освободятся

## Timeout с time.After

### Базовый пример

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    ch := make(chan string)

    // Горутина отправляет после 3 секунд
    go func() {
        time.Sleep(3 * time.Second)
        ch <- "result"
    }()

    // Ждём максимум 2 секунды
    select {
    case result := <-ch:
        fmt.Println("Success:", result)

    case <-time.After(2 * time.Second):
        fmt.Println("Timeout!")
    }
}
```

**Вывод:** `Timeout!` (операция не успела за 2 секунды)

### Важно: утечка таймера

```go
// ❌ Проблема: таймер не останавливается
func processWithTimeout(data string) error {
    select {
    case result := <-process(data):
        return nil  // Таймер продолжает работать!

    case <-time.After(5 * time.Second):
        return errors.New("timeout")
    }
}
```

**Проблема:** `time.After` создаёт таймер, который не отменяется при успешном завершении.

**Правильно:**
```go
func processWithTimeout(data string) error {
    timer := time.NewTimer(5 * time.Second)
    defer timer.Stop()  // Останавливаем таймер

    select {
    case result := <-process(data):
        return nil

    case <-timer.C:
        return errors.New("timeout")
    }
}
```

## Timeout с Context

### Рекомендуемый способ

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func operation(ctx context.Context) error {
    // Симуляция долгой операции
    select {
    case <-time.After(3 * time.Second):
        return nil

    case <-ctx.Done():
        return ctx.Err()  // context.DeadlineExceeded
    }
}

func main() {
    // Контекст с таймаутом 2 секунды
    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel()

    if err := operation(ctx); err != nil {
        fmt.Println("Error:", err)  // context deadline exceeded
    } else {
        fmt.Println("Success")
    }
}
```

**Преимущества context:**
- Автоматическая отмена вложенных операций
- Стандартный способ в Go
- Работает с библиотеками (HTTP, DB, gRPC)

## HTTP запросы с таймаутом

### Вариант 1: Timeout в Client

```go
package main

import (
    "fmt"
    "net/http"
    "time"
)

func main() {
    client := &http.Client{
        Timeout: 5 * time.Second,  // Общий таймаут
    }

    resp, err := client.Get("https://httpbin.org/delay/10")
    if err != nil {
        fmt.Println("Error:", err)  // Timeout после 5 секунд
        return
    }
    defer resp.Body.Close()

    fmt.Println("Success:", resp.Status)
}
```

### Вариант 2: Timeout через Context

```go
package main

import (
    "context"
    "fmt"
    "net/http"
    "time"
)

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel()

    req, _ := http.NewRequestWithContext(ctx, "GET", "https://httpbin.org/delay/5", nil)

    client := &http.Client{}
    resp, err := client.Do(req)
    if err != nil {
        fmt.Println("Error:", err)  // context deadline exceeded
        return
    }
    defer resp.Body.Close()

    fmt.Println("Success:", resp.Status)
}
```

### Детальные таймауты

```go
client := &http.Client{
    Timeout: 10 * time.Second,  // Общий таймаут

    Transport: &http.Transport{
        DialContext: (&net.Dialer{
            Timeout:   2 * time.Second,  // Таймаут соединения
            KeepAlive: 30 * time.Second,
        }).DialContext,

        TLSHandshakeTimeout:   3 * time.Second,  // Таймаут TLS handshake
        ResponseHeaderTimeout: 5 * time.Second,  // Таймаут получения заголовков
        ExpectContinueTimeout: 1 * time.Second,
    },
}
```

## Database запросы с таймаутом

```go
package main

import (
    "context"
    "database/sql"
    "fmt"
    "time"

    _ "github.com/lib/pq"
)

func queryWithTimeout(db *sql.DB) error {
    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel()

    var count int
    err := db.QueryRowContext(ctx, "SELECT COUNT(*) FROM users").Scan(&count)
    if err != nil {
        return err  // context deadline exceeded если запрос долгий
    }

    fmt.Println("Count:", count)
    return nil
}

func main() {
    db, _ := sql.Open("postgres", "connection_string")
    defer db.Close()

    if err := queryWithTimeout(db); err != nil {
        fmt.Println("Error:", err)
    }
}
```

## Множественные операции с общим таймаутом

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func step1(ctx context.Context) error {
    select {
    case <-time.After(1 * time.Second):
        fmt.Println("Step 1 completed")
        return nil
    case <-ctx.Done():
        return ctx.Err()
    }
}

func step2(ctx context.Context) error {
    select {
    case <-time.After(1 * time.Second):
        fmt.Println("Step 2 completed")
        return nil
    case <-ctx.Done():
        return ctx.Err()
    }
}

func step3(ctx context.Context) error {
    select {
    case <-time.After(1 * time.Second):
        fmt.Println("Step 3 completed")
        return nil
    case <-ctx.Done():
        return ctx.Err()
    }
}

func pipeline(ctx context.Context) error {
    if err := step1(ctx); err != nil {
        return err
    }

    if err := step2(ctx); err != nil {
        return err
    }

    if err := step3(ctx); err != nil {
        return err
    }

    return nil
}

func main() {
    // Общий таймаут на все операции
    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel()

    if err := pipeline(ctx); err != nil {
        fmt.Println("Pipeline error:", err)
    } else {
        fmt.Println("Pipeline completed")
    }
}
```

**Вывод:**
```
Step 1 completed
Step 2 completed
Pipeline error: context deadline exceeded
```

Step 3 не успевает выполниться (1 + 1 + 1 = 3 секунды > 2 секунды timeout).

## Retry с таймаутом

```go
package main

import (
    "context"
    "errors"
    "fmt"
    "time"
)

func retryWithTimeout(ctx context.Context, attempts int, delay time.Duration, fn func() error) error {
    for i := 0; i < attempts; i++ {
        // Проверяем таймаут перед попыткой
        if ctx.Err() != nil {
            return ctx.Err()
        }

        err := fn()
        if err == nil {
            return nil
        }

        fmt.Printf("Attempt %d failed: %v\n", i+1, err)

        if i < attempts-1 {
            // Ждём с учётом таймаута
            select {
            case <-time.After(delay):
                // Продолжаем
            case <-ctx.Done():
                return ctx.Err()
            }
        }
    }

    return errors.New("all attempts failed")
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    operation := func() error {
        return errors.New("temporary error")
    }

    err := retryWithTimeout(ctx, 10, time.Second, operation)
    fmt.Println("Result:", err)
}
```

## Worker Pool с таймаутом на задачу

```go
package main

import (
    "context"
    "fmt"
    "sync"
    "time"
)

type Job struct {
    ID      int
    Timeout time.Duration
}

func worker(id int, jobs <-chan Job, wg *sync.WaitGroup) {
    defer wg.Done()

    for job := range jobs {
        ctx, cancel := context.WithTimeout(context.Background(), job.Timeout)

        fmt.Printf("Worker %d: starting job %d (timeout: %v)\n", id, job.ID, job.Timeout)

        err := processJob(ctx, job)
        if err != nil {
            fmt.Printf("Worker %d: job %d failed: %v\n", id, job.ID, err)
        } else {
            fmt.Printf("Worker %d: job %d completed\n", id, job.ID)
        }

        cancel()
    }
}

func processJob(ctx context.Context, job Job) error {
    select {
    case <-time.After(2 * time.Second):  // Симуляция работы
        return nil

    case <-ctx.Done():
        return ctx.Err()
    }
}

func main() {
    jobs := make(chan Job, 10)
    var wg sync.WaitGroup

    // Запускаем workers
    for i := 1; i <= 3; i++ {
        wg.Add(1)
        go worker(i, jobs, &wg)
    }

    // Отправляем задачи с разными таймаутами
    jobs <- Job{ID: 1, Timeout: 3 * time.Second}  // Успеет
    jobs <- Job{ID: 2, Timeout: 1 * time.Second}  // Не успеет
    jobs <- Job{ID: 3, Timeout: 5 * time.Second}  // Успеет

    close(jobs)
    wg.Wait()
}
```

## Вопрос 1: В чём разница между time.After и context.WithTimeout?

**Ответ:**

| time.After | context.WithTimeout |
|------------|---------------------|
| Возвращает канал | Возвращает context |
| Нельзя отменить | Можно отменить через cancel() |
| Утечка при раннем выходе | Автоматическая очистка |
| Локальный таймаут | Распространяется на вложенные операции |
| Простой | Стандартный в Go |

**time.After:**
```go
select {
case result := <-ch:
    return result  // Таймер не останавливается!
case <-time.After(5 * time.Second):
    return nil
}
```

**context.WithTimeout:**
```go
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
defer cancel()  // Всегда останавливаем

select {
case result := <-ch:
    return result  // cancel() вызовется
case <-ctx.Done():
    return nil
}
```

**Когда использовать:**
- `time.After` → простые локальные таймауты (осторожно с утечками)
- `context.WithTimeout` → production код, HTTP, DB, любые библиотеки

---

## Вопрос 2: Как отменить операцию при таймауте?

**Ответ:**

Операция должна проверять context:

```go
func longOperation(ctx context.Context) error {
    for i := 0; i < 100; i++ {
        // Проверяем отмену
        select {
        case <-ctx.Done():
            return ctx.Err()  // Прекращаем работу
        default:
            // Продолжаем
        }

        // Симуляция работы
        time.Sleep(100 * time.Millisecond)
        fmt.Printf("Step %d\n", i)
    }

    return nil
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), time.Second)
    defer cancel()

    longOperation(ctx)  // Остановится через ~1 секунду
}
```

**Без проверки context операция продолжится:**
```go
func badOperation(ctx context.Context) error {
    // ❌ Не проверяем context
    time.Sleep(10 * time.Second)
    return nil  // Выполнится даже после таймаута!
}
```

---

## Вопрос 3: Что произойдёт если не вызвать cancel()?

**Ответ:**

```go
func leak() {
    ctx, cancel := context.WithTimeout(context.Background(), time.Second)
    // defer cancel()  ← Забыли!

    operation(ctx)
}
```

**Последствия:**
- Goroutine leak (таймер продолжает работать)
- Утечка памяти
- Ресурсы не освобождаются

**Правило:** ВСЕГДА вызывайте `defer cancel()` сразу после создания context.

```go
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
defer cancel()  // ✅ Всегда через defer
```

**Безопасно вызывать cancel() несколько раз:**
```go
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
defer cancel()

if earlyExit {
    cancel()  // OK
    return
}

// defer cancel() тоже вызовется - это безопасно
```

---

## Best Practices

1. ✅ **Всегда используйте таймауты для I/O операций**
   ```go
   client := &http.Client{Timeout: 10 * time.Second}
   ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
   ```

2. ✅ **Используйте defer cancel()**
   ```go
   ctx, cancel := context.WithTimeout(ctx, time.Second)
   defer cancel()  // ВСЕГДА
   ```

3. ✅ **Проверяйте context в долгих операциях**
   ```go
   for {
       select {
       case <-ctx.Done():
           return ctx.Err()
       default:
           // Работа
       }
   }
   ```

4. ✅ **Используйте разумные таймауты**
   ```go
   // HTTP: 10-30 секунд
   // Database: 5-15 секунд
   // Микросервисы: 3-10 секунд
   ```

5. ✅ **Предпочитайте context.WithTimeout вместо time.After**
   ```go
   ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
   defer cancel()
   ```

6. ❌ **Не используйте time.After в циклах**
   ```go
   // ❌ Утечка таймеров
   for {
       select {
       case <-time.After(time.Second):  // Новый таймер каждую итерацию!
       }
   }

   // ✅ Правильно
   ticker := time.NewTicker(time.Second)
   defer ticker.Stop()

   for {
       select {
       case <-ticker.C:
       }
   }
   ```

7. ✅ **Логируйте таймауты**
   ```go
   if ctx.Err() == context.DeadlineExceeded {
       log.Printf("Operation timeout: %v", operation)
   }
   ```

## Где спрашивают

- Яндекс
- Авито
- VK
- Т-Банк
- Любые компании с микросервисами

## Связанные темы

- [[Go - Context]]
- [[Go - Select statement]]
- [[Go - Пакет time]]
- [[Задача - Graceful Shutdown]]
- [[Go - Пакет net-http]]
