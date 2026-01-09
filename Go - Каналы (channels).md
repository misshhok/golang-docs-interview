# Go - Каналы (channels)

Каналы (channels) - это типизированные conduits для передачи данных между горутинами. Каналы обеспечивают безопасную коммуникацию и синхронизацию без явных блокировок.

## Основы каналов

### Создание канала

```go
// Небуферизованный канал
ch := make(chan int)

// Буферизованный канал (capacity = 10)
buffered := make(chan string, 10)

// Канал только для чтения/записи
var readOnly <-chan int = ch  // Только чтение
var writeOnly chan<- int = ch // Только запись
```

### Отправка и получение данных

```go
ch := make(chan int)

// Отправка значения в канал
ch <- 42

// Получение значения из канала
value := <-ch

// Получение с проверкой закрытия канала
value, ok := <-ch
if ok {
    fmt.Println("Received:", value)
} else {
    fmt.Println("Channel closed")
}
```

## Небуферизованные vs Буферизованные каналы

### Небуферизованный канал (Unbuffered)

Отправка **блокируется** пока получатель не готов принять данные. Обеспечивает синхронизацию.

```go
func main() {
    ch := make(chan string)

    go func() {
        ch <- "Hello" // Блокируется, пока main не прочитает
        fmt.Println("Sent")
    }()

    msg := <-ch // Блокируется, пока горутина не отправит
    fmt.Println(msg)
}
```

**Характеристики:**
- Capacity = 0
- Отправка и получение происходят одновременно (rendezvous)
- Гарантирует синхронизацию между отправителем и получателем

### Буферизованный канал (Buffered)

Отправка блокируется только когда буфер полон. Получение блокируется когда буфер пуст.

```go
func main() {
    ch := make(chan int, 3) // Буфер на 3 элемента

    // Можно отправить 3 значения без блокировки
    ch <- 1
    ch <- 2
    ch <- 3

    // Следующая отправка заблокируется, пока кто-то не прочитает
    // ch <- 4 // deadlock!

    fmt.Println(<-ch) // 1
    fmt.Println(<-ch) // 2
    fmt.Println(<-ch) // 3
}
```

**Когда использовать буферизованные каналы:**
- Известно конечное количество отправителей
- Хотите уменьшить латентность (отправитель не ждет получателя)
- Реализация семафора или rate limiter

```go
// Пример: ограничение количества одновременных операций
semaphore := make(chan struct{}, 10) // Максимум 10 одновременно

for i := 0; i < 100; i++ {
    semaphore <- struct{}{} // Захватываем слот
    go func(id int) {
        defer func() { <-semaphore }() // Освобождаем слот
        processTask(id)
    }(i)
}
```

## Закрытие каналов

### Закрытие канала

```go
ch := make(chan int, 3)
ch <- 1
ch <- 2
ch <- 3

close(ch) // Закрываем канал

// Можно читать из закрытого канала
fmt.Println(<-ch) // 1
fmt.Println(<-ch) // 2
fmt.Println(<-ch) // 3
fmt.Println(<-ch) // 0 (zero value)

// Проверка, закрыт ли канал
value, ok := <-ch
if !ok {
    fmt.Println("Channel closed")
}
```

**Правила закрытия:**
- ❌ Нельзя отправлять в закрытый канал (panic!)
- ✅ Можно читать из закрытого канала (возвращает zero value)
- ❌ Нельзя закрыть канал дважды (panic!)
- ✅ Закрывать канал должен **отправитель**, не получатель
- ✅ Закрытие канала не обязательно (GC сам почистит)

### Итерация по каналу

```go
ch := make(chan int, 5)

// Отправитель
go func() {
    for i := 0; i < 5; i++ {
        ch <- i
    }
    close(ch) // Важно закрыть!
}()

// Получатель
for value := range ch {
    fmt.Println(value) // 0, 1, 2, 3, 4
}
// range автоматически выходит когда канал закрыт
```

**Важно**: `range` будет ждать вечно, если канал не закрыт!

## Направленные каналы (Directional Channels)

Используйте направленные каналы для ясности API и предотвращения ошибок:

```go
// Функция только отправляет в канал
func producer(ch chan<- int) {
    for i := 0; i < 10; i++ {
        ch <- i
    }
    close(ch)
}

// Функция только читает из канала
func consumer(ch <-chan int) {
    for value := range ch {
        fmt.Println(value)
    }
}

func main() {
    ch := make(chan int)
    go producer(ch)
    consumer(ch)
}
```

**Преобразования:**
- Bidirectional → Read-only: `var readOnly <-chan int = ch` ✅
- Bidirectional → Write-only: `var writeOnly chan<- int = ch` ✅
- Read-only → Bidirectional: ❌ Нельзя
- Write-only → Bidirectional: ❌ Нельзя

## Паттерны работы с каналами

### 1. Pipeline Pattern

Соединение этапов обработки через каналы:

```go
func generator(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        for _, n := range nums {
            out <- n
        }
        close(out)
    }()
    return out
}

func square(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            out <- n * n
        }
        close(out)
    }()
    return out
}

func main() {
    // Pipeline: generator → square
    nums := generator(1, 2, 3, 4)
    squared := square(nums)

    for result := range squared {
        fmt.Println(result) // 1, 4, 9, 16
    }
}
```

### 2. Fan-Out, Fan-In Pattern

**Fan-Out**: Множество горутин читают из одного канала

```go
func worker(id int, jobs <-chan int, results chan<- int) {
    for job := range jobs {
        fmt.Printf("Worker %d processing job %d\n", id, job)
        results <- job * 2
    }
}

func fanOut() {
    jobs := make(chan int, 100)
    results := make(chan int, 100)

    // Fan-out: 3 workers читают из одного канала
    for w := 1; w <= 3; w++ {
        go worker(w, jobs, results)
    }

    // Отправляем задачи
    for j := 1; j <= 9; j++ {
        jobs <- j
    }
    close(jobs)

    // Получаем результаты
    for a := 1; a <= 9; a++ {
        <-results
    }
}
```

**Fan-In**: Множество горутин пишут в один канал

```go
func merge(channels ...<-chan int) <-chan int {
    out := make(chan int)
    var wg sync.WaitGroup

    // Fan-in: читаем из всех каналов
    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan int) {
            defer wg.Done()
            for value := range c {
                out <- value
            }
        }(ch)
    }

    // Закрываем выходной канал когда все входные закрыты
    go func() {
        wg.Wait()
        close(out)
    }()

    return out
}

func main() {
    ch1 := make(chan int)
    ch2 := make(chan int)

    go func() {
        ch1 <- 1
        ch1 <- 2
        close(ch1)
    }()

    go func() {
        ch2 <- 3
        ch2 <- 4
        close(ch2)
    }()

    merged := merge(ch1, ch2)
    for value := range merged {
        fmt.Println(value) // 1, 2, 3, 4 (порядок не гарантирован)
    }
}
```

### 3. Timeout Pattern

```go
func main() {
    ch := make(chan string)

    go func() {
        time.Sleep(2 * time.Second)
        ch <- "result"
    }()

    select {
    case result := <-ch:
        fmt.Println("Received:", result)
    case <-time.After(1 * time.Second):
        fmt.Println("Timeout!")
    }
}
```

### 4. Done Channel Pattern

Сигнализация о завершении работы:

```go
func worker(done <-chan struct{}) {
    for {
        select {
        case <-done:
            fmt.Println("Worker stopped")
            return
        default:
            // Работа
            fmt.Println("Working...")
            time.Sleep(500 * time.Millisecond)
        }
    }
}

func main() {
    done := make(chan struct{})

    go worker(done)

    time.Sleep(2 * time.Second)
    close(done) // Сигнал остановки
    time.Sleep(1 * time.Second)
}
```

Используйте `struct{}` для done каналов - не занимает память!

### 5. Nil Channel Pattern

Nil канал блокируется навсегда (useful в select):

```go
func merge(ch1, ch2 <-chan int) <-chan int {
    out := make(chan int)

    go func() {
        defer close(out)
        for ch1 != nil || ch2 != nil {
            select {
            case v, ok := <-ch1:
                if !ok {
                    ch1 = nil // Отключаем этот case
                    continue
                }
                out <- v
            case v, ok := <-ch2:
                if !ok {
                    ch2 = nil // Отключаем этот case
                    continue
                }
                out <- v
            }
        }
    }()

    return out
}
```

## Типичные ошибки

### 1. Deadlock - отправка в небуферизованный канал без получателя

```go
// ❌ Deadlock!
func main() {
    ch := make(chan int)
    ch <- 42 // Блокируется вечно, никто не читает
}

// ✅ Правильно
func main() {
    ch := make(chan int)
    go func() {
        ch <- 42
    }()
    fmt.Println(<-ch)
}
```

### 2. Отправка в закрытый канал

```go
// ❌ Panic: send on closed channel
ch := make(chan int)
close(ch)
ch <- 42 // PANIC!
```

### 3. Закрытие канала получателем

```go
// ❌ Плохая практика: получатель закрывает канал
func consumer(ch chan int) {
    for value := range ch {
        fmt.Println(value)
    }
    close(ch) // Если отправитель попытается отправить - panic!
}

// ✅ Правильно: отправитель закрывает канал
func producer(ch chan int) {
    for i := 0; i < 10; i++ {
        ch <- i
    }
    close(ch) // Отправитель контролирует закрытие
}
```

### 4. Утечка горутин с каналами

```go
// ❌ Утечка: горутина ждет вечно
func leak() {
    ch := make(chan int)
    go func() {
        <-ch // Никто никогда не отправит!
    }()
}

// ✅ С таймаутом или done каналом
func noLeak(done <-chan struct{}) {
    ch := make(chan int)
    go func() {
        select {
        case <-ch:
            // Обработка
        case <-done:
            return
        }
    }()
}
```

### 5. Неправильный размер буфера

```go
// ❌ Слишком большой буфер - лишняя память
ch := make(chan int, 1000000)

// ❌ Буфер = количеству отправок (часто не нужно)
ch := make(chan int, 10)
for i := 0; i < 10; i++ {
    ch <- i // Зачем буфер? Нет параллельной обработки
}

// ✅ Размер буфера = количеству workers или разумный лимит
workQueue := make(chan Task, 100)
```

## Best Practices

### 1. Отправитель закрывает канал

```go
// ✅ Producer закрывает канал
func producer(ch chan<- int) {
    defer close(ch) // Гарантируем закрытие
    for i := 0; i < 10; i++ {
        ch <- i
    }
}
```

### 2. Используйте направленные каналы в сигнатурах

```go
// ✅ Ясно, кто читает, кто пишет
func send(ch chan<- int) { ch <- 42 }
func receive(ch <-chan int) { <-ch }
```

### 3. Используйте буферизацию осознанно

```go
// Небуферизованный - когда нужна синхронизация
sync := make(chan struct{})

// Буферизованный - для decoupling producer/consumer
queue := make(chan Task, 100)

// Буфер размера 1 - для "latest value" семантики
latest := make(chan Data, 1)
```

### 4. Избегайте select с default без необходимости

```go
// ❌ Busy-wait, тратит CPU
for {
    select {
    case v := <-ch:
        process(v)
    default:
        // Цикл крутится вхолостую!
    }
}

// ✅ Блокируется когда нет работы
for v := range ch {
    process(v)
}
```

### 5. Проверяйте закрытие при необходимости

```go
// Если важно знать, закрыт ли канал
value, ok := <-ch
if !ok {
    // Канал закрыт
    return
}
```

## Производительность

**Стоимость операций:**
- Отправка/получение из небуферизованного канала: ~100-200 ns
- Отправка/получение из буферизованного канала: ~50-100 ns
- Блокировка на мьютексе: ~20-50 ns

**Каналы медленнее мьютексов**, но проще в использовании и предотвращают ошибки!

**Когда использовать каналы:**
- Передача ownership данных
- Распределение работы между горутинами
- Коммуникация результатов

**Когда использовать мьютексы:**
- Защита shared state
- Короткие критические секции
- Счетчики, кэши

## Связанные темы

- [[Go - Горутины (goroutines)]] - запуск конкурентных функций
- [[Go - Select statement]] - работа с множественными каналами
- [[Go - Context]] - управление отменой и таймаутами
- [[Go - Пакет sync]] - примитивы синхронизации
- [[Go - Mutex и RWMutex]] - альтернатива каналам для shared state
- [[Многопоточность vs Параллелизм vs Конкурентность]] - концептуальные основы
