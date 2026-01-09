# Go - Select statement

`select` позволяет горутине ждать на множественных операциях с каналами одновременно. Это ключевой инструмент для координации конкурентных операций в Go.

## Основы select

### Синтаксис

```go
select {
case <-ch1:
    // Выполняется если ch1 готов к чтению
case val := <-ch2:
    // Выполняется если ch2 готов и читаем значение
case ch3 <- value:
    // Выполняется если ch3 готов к записи
default:
    // Выполняется если ни один case не готов (опционально)
}
```

### Базовый пример

```go
func main() {
    ch1 := make(chan string)
    ch2 := make(chan string)

    go func() {
        time.Sleep(1 * time.Second)
        ch1 <- "one"
    }()

    go func() {
        time.Sleep(2 * time.Second)
        ch2 <- "two"
    }()

    for i := 0; i < 2; i++ {
        select {
        case msg1 := <-ch1:
            fmt.Println("Received:", msg1)
        case msg2 := <-ch2:
            fmt.Println("Received:", msg2)
        }
    }
}
```

## Ключевые особенности select

### 1. Случайный выбор при множественных готовых case

Если несколько case готовы одновременно, Go **случайно** выбирает один:

```go
func main() {
    ch := make(chan int, 2)
    ch <- 1
    ch <- 2

    select {
    case val := <-ch:
        fmt.Println(val) // Может быть 1 или 2!
    case val := <-ch:
        fmt.Println(val)
    }
}
```

Это предотвращает голодание (starvation) одного из case.

### 2. Блокировка без default

Без `default`, `select` блокируется пока хотя бы один case не станет готов:

```go
select {
case msg := <-ch:
    fmt.Println(msg)
}
// Блокируется вечно если никто не отправляет в ch
```

### 3. Non-blocking с default

`default` выполняется сразу если ни один case не готов:

```go
select {
case msg := <-ch:
    fmt.Println("Received:", msg)
default:
    fmt.Println("No message received")
}
// Не блокируется никогда
```

### 4. Nil каналы игнорируются

Операции с `nil` каналами блокируются навсегда и игнорируются в `select`:

```go
var ch chan int // nil канал

select {
case <-ch: // Этот case никогда не выберется
    fmt.Println("Never happens")
default:
    fmt.Println("Default executed")
}
```

## Паттерны использования select

### 1. Timeout Pattern

Ограничение времени ожидания операции:

```go
func fetchWithTimeout(url string, timeout time.Duration) (string, error) {
    result := make(chan string, 1)

    go func() {
        // Долгая операция
        data := fetch(url)
        result <- data
    }()

    select {
    case data := <-result:
        return data, nil
    case <-time.After(timeout):
        return "", fmt.Errorf("timeout after %v", timeout)
    }
}

func main() {
    data, err := fetchWithTimeout("https://example.com", 2*time.Second)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    fmt.Println("Data:", data)
}
```

**Важно**: `time.After()` создает новый канал и таймер каждый раз. В циклах используйте `time.NewTimer()`:

```go
// ❌ Утечка таймеров в цикле
for {
    select {
    case <-ch:
        // handle
    case <-time.After(1 * time.Second):
        // Создается новый таймер каждую итерацию!
    }
}

// ✅ Правильно
timer := time.NewTimer(1 * time.Second)
defer timer.Stop()

for {
    timer.Reset(1 * time.Second)
    select {
    case <-ch:
        // handle
    case <-timer.C:
        // timeout
    }
}
```

### 2. Done/Context Pattern

Graceful shutdown с использованием done канала:

```go
func worker(done <-chan struct{}, jobs <-chan int) {
    for {
        select {
        case job := <-jobs:
            fmt.Println("Processing job:", job)
        case <-done:
            fmt.Println("Worker stopped")
            return
        }
    }
}

func main() {
    done := make(chan struct{})
    jobs := make(chan int, 5)

    go worker(done, jobs)

    for i := 0; i < 3; i++ {
        jobs <- i
    }

    close(done) // Сигнал остановки
    time.Sleep(1 * time.Second)
}
```

С использованием `context`:

```go
func worker(ctx context.Context, jobs <-chan int) {
    for {
        select {
        case job := <-jobs:
            fmt.Println("Processing job:", job)
        case <-ctx.Done():
            fmt.Println("Worker stopped:", ctx.Err())
            return
        }
    }
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    jobs := make(chan int, 5)

    go worker(ctx, jobs)

    for i := 0; i < 3; i++ {
        jobs <- i
    }

    cancel() // Отмена через context
    time.Sleep(1 * time.Second)
}
```

### 3. Non-blocking Channel Operations

Попытка отправки/получения без блокировки:

```go
// Non-blocking send
select {
case ch <- value:
    fmt.Println("Sent value")
default:
    fmt.Println("Channel not ready, skipped")
}

// Non-blocking receive
select {
case value := <-ch:
    fmt.Println("Received:", value)
default:
    fmt.Println("No value available")
}
```

### 4. Merge Pattern (Fan-In)

Объединение нескольких каналов в один:

```go
func merge(ch1, ch2 <-chan int) <-chan int {
    out := make(chan int)

    go func() {
        defer close(out)
        for ch1 != nil || ch2 != nil {
            select {
            case v, ok := <-ch1:
                if !ok {
                    ch1 = nil // Отключаем закрытый канал
                    continue
                }
                out <- v
            case v, ok := <-ch2:
                if !ok {
                    ch2 = nil
                    continue
                }
                out <- v
            }
        }
    }()

    return out
}

func main() {
    ch1 := make(chan int)
    ch2 := make(chan int)

    go func() {
        for i := 0; i < 3; i++ {
            ch1 <- i
        }
        close(ch1)
    }()

    go func() {
        for i := 10; i < 13; i++ {
            ch2 <- i
        }
        close(ch2)
    }()

    for val := range merge(ch1, ch2) {
        fmt.Println(val) // 0, 10, 1, 11, 2, 12 (порядок не гарантирован)
    }
}
```

### 5. Rate Limiting Pattern

Ограничение частоты операций:

```go
func rateLimiter() {
    // Максимум 5 запросов в секунду
    limiter := time.Tick(200 * time.Millisecond)
    requests := make(chan int, 10)

    // Генерируем запросы
    for i := 1; i <= 10; i++ {
        requests <- i
    }
    close(requests)

    // Обрабатываем с ограничением частоты
    for req := range requests {
        <-limiter // Ждем следующего тика
        fmt.Println("Request", req, "at", time.Now())
    }
}
```

С возможностью burst:

```go
func rateLimiterWithBurst() {
    // Буфер позволяет "burst" из 3 запросов
    limiter := make(chan struct{}, 3)

    // Заполняем буфер
    for i := 0; i < 3; i++ {
        limiter <- struct{}{}
    }

    // Пополняем буфер каждые 200ms
    go func() {
        for range time.Tick(200 * time.Millisecond) {
            select {
            case limiter <- struct{}{}:
            default:
                // Буфер полон
            }
        }
    }()

    requests := make(chan int, 10)
    for i := 1; i <= 10; i++ {
        requests <- i
    }
    close(requests)

    for req := range requests {
        <-limiter
        fmt.Println("Request", req, "at", time.Now())
    }
}
```

### 6. Priority Channel Pattern

Приоритезация одного канала над другим:

```go
func prioritySelect(high, low <-chan int) {
    for {
        select {
        case val := <-high:
            fmt.Println("High priority:", val)
        default:
            // Если high канал не готов, проверяем low
            select {
            case val := <-high:
                fmt.Println("High priority:", val)
            case val := <-low:
                fmt.Println("Low priority:", val)
            }
        }
    }
}
```

### 7. Heartbeat Pattern

Периодическая отправка "жив" сигнала:

```go
func worker(heartbeat chan<- struct{}) {
    ticker := time.NewTicker(1 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-ticker.C:
            // Отправляем heartbeat
            select {
            case heartbeat <- struct{}{}:
            default:
                // Никто не слушает, пропускаем
            }
        default:
            // Основная работа
            doWork()
        }
    }
}

func monitor(heartbeat <-chan struct{}) {
    timeout := time.After(3 * time.Second)

    for {
        select {
        case <-heartbeat:
            fmt.Println("Worker alive")
            timeout = time.After(3 * time.Second) // Reset timeout
        case <-timeout:
            fmt.Println("Worker dead!")
            return
        }
    }
}
```

## Сравнение с другими конструкциями

### select vs switch

```go
// switch - выбор по значению
switch value {
case 1:
    // ...
case 2:
    // ...
}

// select - выбор по готовности канала
select {
case <-ch1:
    // ...
case <-ch2:
    // ...
}
```

**Ключевые различия:**
- `switch` проверяет case сверху вниз, выбирает первый подходящий
- `select` выбирает **случайный** готовый case
- `switch` не блокирует
- `select` блокирует (без default)

## Типичные ошибки

### 1. Забыть default в non-blocking операции

```go
// ❌ Блокируется если канал не готов
select {
case ch <- value:
    fmt.Println("Sent")
}

// ✅ Non-blocking
select {
case ch <- value:
    fmt.Println("Sent")
default:
    fmt.Println("Channel full")
}
```

### 2. Утечка таймеров в цикле

```go
// ❌ Создает новый таймер каждую итерацию
for {
    select {
    case <-time.After(1 * time.Second):
        // Утечка!
    }
}

// ✅ Переиспользуем таймер
timer := time.NewTimer(1 * time.Second)
defer timer.Stop()
for {
    select {
    case <-timer.C:
        timer.Reset(1 * time.Second)
    }
}
```

### 3. Забыть обработать закрытие канала

```go
// ❌ Не проверяем, закрыт ли канал
select {
case val := <-ch:
    process(val) // val может быть zero value!
}

// ✅ Проверяем закрытие
select {
case val, ok := <-ch:
    if !ok {
        // Канал закрыт
        return
    }
    process(val)
}
```

### 4. Busy-wait с пустым default

```go
// ❌ Тратит 100% CPU
for {
    select {
    case val := <-ch:
        process(val)
    default:
        // Пустой default - цикл крутится вхолостую!
    }
}

// ✅ Блокируемся когда нечего делать
for val := range ch {
    process(val)
}
```

### 5. Неправильный nil канал паттерн

```go
// ❌ Неправильно
ch := make(chan int)
close(ch)

select {
case <-ch:
    // Этот case будет выбираться постоянно!
}

// ✅ Отключаем канал через nil
ch := make(chan int)
close(ch)

if val, ok := <-ch; !ok {
    ch = nil // Отключаем канал в select
}

select {
case <-ch: // Теперь игнорируется
}
```

## Производительность

**Стоимость операций:**
- `select` с 2 case: ~10-20 ns (без блокировки)
- `select` с блокировкой: ~100-200 ns
- `select` с 10+ case: ~50-100 ns

`select` очень эффективен даже с большим количеством cases.

## Best Practices

### 1. Используйте select для мультиплексирования

```go
// ✅ Один select обрабатывает множество источников
select {
case msg := <-ch1:
    handleType1(msg)
case msg := <-ch2:
    handleType2(msg)
case <-ctx.Done():
    return
}
```

### 2. Всегда добавляйте context.Done() в долгие select'ы

```go
// ✅ Возможность отмены
select {
case result := <-resultCh:
    return result
case <-ctx.Done():
    return ctx.Err()
}
```

### 3. Используйте nil каналы для динамического отключения cases

```go
// ✅ Отключаем закрытые каналы
if !ok {
    ch1 = nil // Этот case больше не рассматривается
}
```

### 4. Избегайте пустого default в циклах

```go
// ❌ Busy-wait
for {
    select {
    case <-ch:
    default:
    }
}

// ✅ Добавьте логику или time.Sleep в default
for {
    select {
    case <-ch:
    default:
        time.Sleep(100 * time.Millisecond)
    }
}
```

## Связанные темы

- [[Go - Каналы (channels)]] - основы работы с каналами
- [[Go - Горутины (goroutines)]] - конкурентное выполнение
- [[Go - Context]] - управление отменой и таймаутами
- [[Go - Пакет sync]] - альтернативные примитивы синхронизации
- [[Многопоточность vs Параллелизм vs Конкурентность]] - концепции
