# Go - Горутины (goroutines)

Горутина - это легковесный поток выполнения, управляемый Go runtime. Горутины позволяют выполнять функции конкурентно и являются основой модели конкурентности в Go.

## Что такое горутина

Горутина - это функция, которая выполняется конкурентно с другими функциями. В отличие от потоков ОС, горутины:
- **Легковесные** - занимают ~2KB памяти (vs 1-2MB для OS thread)
- **Управляются Go runtime** - планировщик Go сам распределяет горутины по OS threads
- **Дешевые в создании** - можно создавать тысячи и миллионы горутин

## Создание горутины

Для запуска функции как горутины используется ключевое слово `go`:

```go
package main

import (
    "fmt"
    "time"
)

func sayHello() {
    fmt.Println("Hello from goroutine!")
}

func main() {
    // Запуск функции как горутины
    go sayHello()

    // Даем время горутине выполниться
    time.Sleep(100 * time.Millisecond)

    fmt.Println("Main function")
}
```

### Анонимные функции как горутины

```go
func main() {
    // Анонимная функция как горутина
    go func() {
        fmt.Println("Anonymous goroutine")
    }()

    // С параметрами
    message := "Hello"
    go func(msg string) {
        fmt.Println(msg)
    }(message) // Передаем значение, а не захватываем переменную!

    time.Sleep(100 * time.Millisecond)
}
```

**Важно**: Передавайте переменные как параметры, а не захватывайте из внешней области видимости в цикле:

```go
// ❌ НЕПРАВИЛЬНО
for i := 0; i < 5; i++ {
    go func() {
        fmt.Println(i) // Все горутины выведут 5!
    }()
}

// ✅ ПРАВИЛЬНО
for i := 0; i < 5; i++ {
    go func(n int) {
        fmt.Println(n)
    }(i) // Передаем копию i
}
```

## Планировщик горутин (Go Scheduler)

Go использует модель **M:N планирования**:
- **M goroutines** исполняются на **N OS threads**
- Планировщик автоматически мультиплексирует горутины на доступные потоки

### Компоненты планировщика (GMP модель)

- **G (Goroutine)** - горутина, содержит стек, instruction pointer, и другую информацию
- **M (Machine)** - OS thread, который исполняет горутины
- **P (Processor)** - контекст планирования, содержит очередь runnable горутин

```
[G1] [G2] [G3] [G4]  ← Множество горутин
       ↓
    [P1] [P2]         ← Processors (по умолчанию GOMAXPROCS)
       ↓
    [M1] [M2]         ← OS Threads
```

### GOMAXPROCS

Переменная окружения `GOMAXPROCS` определяет количество OS threads, которые могут исполнять Go код одновременно:

```go
import "runtime"

func main() {
    // Получить текущее значение
    fmt.Println(runtime.GOMAXPROCS(0))

    // Установить новое значение (количество CPU cores)
    runtime.GOMAXPROCS(runtime.NumCPU())
}
```

По умолчанию `GOMAXPROCS` = количеству CPU cores.

## Жизненный цикл горутины

1. **Создание** - `go func()` создает горутину и добавляет её в очередь планировщика
2. **Runnable** - горутина готова к исполнению, ждет свободный P
3. **Running** - горутина исполняется на M
4. **Waiting** - горутина заблокирована (I/O, channel операция, sync)
5. **Dead** - горутина завершилась

## Блокирование vs Конкурентность

**Блокирующие операции** (горутина переходит в Waiting):
- Чтение/запись в канал (если канал не готов)
- I/O операции (network, disk)
- `time.Sleep()`
- Ожидание на мьютексе
- System calls

Когда горутина блокируется, планировщик может переключиться на другую runnable горутину.

## Завершение горутин

### Проблема: основная функция не ждет

```go
func main() {
    go func() {
        time.Sleep(1 * time.Second)
        fmt.Println("This may not print!")
    }()

    // main завершается сразу, не дожидаясь горутины
    // Программа завершается, все горутины убиваются
}
```

### Решение 1: WaitGroup

```go
import "sync"

func main() {
    var wg sync.WaitGroup

    wg.Add(1) // Добавляем счетчик
    go func() {
        defer wg.Done() // Уменьшаем счетчик при завершении
        fmt.Println("Goroutine finished")
    }()

    wg.Wait() // Ждем, пока счетчик не станет 0
}
```

### Решение 2: Каналы

```go
func main() {
    done := make(chan bool)

    go func() {
        fmt.Println("Working...")
        done <- true // Сигнализируем о завершении
    }()

    <-done // Ждем сигнала
}
```

## Ограничение количества горутин

### Проблема: слишком много горутин

```go
// ❌ Может создать миллионы горутин и исчерпать память
for i := 0; i < 1000000; i++ {
    go processItem(i)
}
```

### Решение: Worker Pool Pattern

```go
func workerPool(jobs <-chan int, results chan<- int, numWorkers int) {
    var wg sync.WaitGroup

    // Создаем фиксированное количество workers
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for job := range jobs {
                results <- processJob(job)
            }
        }()
    }

    wg.Wait()
    close(results)
}

func main() {
    jobs := make(chan int, 100)
    results := make(chan int, 100)

    // Запускаем 10 workers
    go workerPool(jobs, results, 10)

    // Отправляем задачи
    for i := 0; i < 100; i++ {
        jobs <- i
    }
    close(jobs)

    // Получаем результаты
    for result := range results {
        fmt.Println(result)
    }
}
```

## Best Practices

### 1. Всегда имейте способ завершить горутину

```go
// ✅ С context для отмены
func worker(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            return // Выход при отмене
        default:
            // Работа
        }
    }
}
```

### 2. Избегайте утечек горутин (goroutine leaks)

```go
// ❌ Утечка: горутина ждет вечно
func leak() {
    ch := make(chan int)
    go func() {
        val := <-ch // Никто никогда не отправит значение!
        fmt.Println(val)
    }()
}

// ✅ С таймаутом
func noLeak() {
    ch := make(chan int)
    go func() {
        select {
        case val := <-ch:
            fmt.Println(val)
        case <-time.After(1 * time.Second):
            return // Выход по таймауту
        }
    }()
}
```

### 3. Не забывайте о panic в горутинах

```go
func safeLaunch() {
    go func() {
        defer func() {
            if r := recover(); r != nil {
                fmt.Println("Recovered from panic:", r)
            }
        }()

        // Код, который может вызвать panic
        panic("something went wrong")
    }()
}
```

Panic в горутине **не ловится** основной функцией и завершит всю программу!

### 4. Используйте каналы для коммуникации

> "Do not communicate by sharing memory; instead, share memory by communicating."
> — Go proverb

```go
// ❌ Sharing memory (требует синхронизации)
var counter int
var mu sync.Mutex

go func() {
    mu.Lock()
    counter++
    mu.Unlock()
}()

// ✅ Communicating (через каналы)
counterChan := make(chan int)
go func() {
    count := 0
    for range counterChan {
        count++
    }
}()
```

## Типичные ошибки

### 1. Забыть дождаться завершения горутин

```go
// ❌ Горутины не успеют выполниться
func main() {
    for i := 0; i < 10; i++ {
        go fmt.Println(i)
    }
    // main завершается сразу
}
```

### 2. Гонка данных (data race)

```go
// ❌ Data race: конкурентный доступ без синхронизации
var counter int
for i := 0; i < 1000; i++ {
    go func() {
        counter++ // НЕБЕЗОПАСНО!
    }()
}
```

Используйте `go run -race` для обнаружения гонок данных.

### 3. Создание слишком большого количества горутин

Помните, что горутины не бесплатны. Для обработки большого количества задач используйте worker pool pattern.

## Производительность

**Сравнение затрат:**
- Создание горутины: ~2-3 мкс
- Создание OS thread: ~1000 мкс (в 300+ раз медленнее!)
- Память горутины: ~2KB
- Память OS thread: ~1-2MB

**Типичные числа:**
- Можно легко создать 100,000+ горутин на обычном ПК
- На production серверах работают миллионы горутин

## Связанные темы

- [[Go - Каналы (channels)]] - коммуникация между горутинами
- [[Go - Context]] - управление жизненным циклом и отмена
- [[Go - Пакет sync]] - примитивы синхронизации
- [[Go - WaitGroup и Once]] - синхронизация завершения
- [[Go - Mutex и RWMutex]] - защита shared state
- [[Многопоточность vs Параллелизм vs Конкурентность]] - концептуальные различия
- [[Go - Select statement]] - работа с множественными каналами
- [[Go - Атомарные операции]] - lock-free синхронизация
