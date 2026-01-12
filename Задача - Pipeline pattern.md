# Задача - Pipeline pattern

Pipeline (конвейер) - паттерн, где данные проходят через последовательность стадий обработки, каждая из которых выполняется в отдельной горутине. Очень популярен на собеседованиях!

## Концепция Pipeline

**Pipeline** - серия стадий (stages), соединённых каналами, где:
- Каждая стадия - группа горутин выполняющих одну функцию
- Горутины получают данные из входного канала
- Обрабатывают и отправляют в выходной канал
- Стадии работают конкурентно

```
[Генератор] → ch1 → [Стадия 1] → ch2 → [Стадия 2] → ch3 → [Результат]
```

**Применение:**
- Обработка потоков данных (ETL)
- Обработка изображений/видео
- Парсинг и трансформация данных
- Построение цепочек middleware

## Простой Pipeline

```go
package main

import "fmt"

// Стадия 1: Генератор чисел
func generator(nums ...int) <-chan int {
    out := make(chan int)

    go func() {
        defer close(out)  // Закрываем после отправки всех данных

        for _, n := range nums {
            out <- n
        }
    }()

    return out
}

// Стадия 2: Умножение на 2
func square(in <-chan int) <-chan int {
    out := make(chan int)

    go func() {
        defer close(out)

        for n := range in {  // Читаем пока канал не закроется
            out <- n * n
        }
    }()

    return out
}

// Стадия 3: Фильтр (только чётные)
func filter(in <-chan int) <-chan int {
    out := make(chan int)

    go func() {
        defer close(out)

        for n := range in {
            if n%2 == 0 {
                out <- n
            }
        }
    }()

    return out
}

func main() {
    // Построение pipeline
    numbers := generator(1, 2, 3, 4, 5)
    squared := square(numbers)
    filtered := filter(squared)

    // Получение результатов
    for result := range filtered {
        fmt.Println(result)
    }
}
```

**Вывод:**
```
4
16
```

### Ключевые принципы

1. **Каждая стадия закрывает свой выходной канал**
   ```go
   defer close(out)
   ```

2. **Range по каналу**
   ```go
   for n := range in {  // Автоматически выходит при close(in)
   ```

3. **Однонаправленные каналы**
   ```go
   func stage(in <-chan int) <-chan int  // in - только чтение, возврат - только чтение
   ```

4. **Не закрывать входной канал**
   - Стадия не закрывает входной канал (не владеет им)
   - Закрывает только свой выходной канал

## Pipeline с несколькими workers на стадии

```go
package main

import (
    "fmt"
    "sync"
)

// Генератор
func generator(nums ...int) <-chan int {
    out := make(chan int)

    go func() {
        defer close(out)
        for _, n := range nums {
            out <- n
        }
    }()

    return out
}

// Обработка с несколькими workers
func process(in <-chan int, workers int) <-chan int {
    out := make(chan int)

    var wg sync.WaitGroup
    wg.Add(workers)

    // Запускаем несколько workers для этой стадии
    for i := 0; i < workers; i++ {
        go func(id int) {
            defer wg.Done()

            for n := range in {
                fmt.Printf("Worker %d processing %d\n", id, n)
                out <- n * 2  // Обработка
            }
        }(i)
    }

    // Закрываем out когда все workers завершатся
    go func() {
        wg.Wait()
        close(out)
    }()

    return out
}

func main() {
    numbers := generator(1, 2, 3, 4, 5, 6, 7, 8)
    processed := process(numbers, 3)  // 3 воркера

    for result := range processed {
        fmt.Println("Result:", result)
    }
}
```

**Важно:** Нужна горутина для `close(out)` после `wg.Wait()`, иначе получатель не узнает когда стадия завершилась.

## Pipeline с Context (cancellation)

```go
package main

import (
    "context"
    "fmt"
    "time"
)

// Генератор с поддержкой отмены
func generator(ctx context.Context, nums ...int) <-chan int {
    out := make(chan int)

    go func() {
        defer close(out)

        for _, n := range nums {
            select {
            case out <- n:
                // Отправили
            case <-ctx.Done():
                // Отменено
                fmt.Println("Generator cancelled")
                return
            }
        }
    }()

    return out
}

// Обработка с поддержкой отмены
func square(ctx context.Context, in <-chan int) <-chan int {
    out := make(chan int)

    go func() {
        defer close(out)

        for {
            select {
            case n, ok := <-in:
                if !ok {
                    return  // Входной канал закрыт
                }

                select {
                case out <- n * n:
                    // Отправили
                case <-ctx.Done():
                    fmt.Println("Square cancelled")
                    return
                }

            case <-ctx.Done():
                fmt.Println("Square cancelled")
                return
            }
        }
    }()

    return out
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel()

    numbers := generator(ctx, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
    squared := square(ctx, numbers)

    // Читаем с задержкой (симуляция медленного получателя)
    for n := range squared {
        fmt.Println(n)
        time.Sleep(500 * time.Millisecond)
    }
}
```

**Pipeline останавливается через 2 секунды** благодаря context timeout.

## Реальный пример: Обработка файлов

```go
package main

import (
    "bufio"
    "context"
    "fmt"
    "os"
    "strings"
)

// Стадия 1: Читаем строки из файла
func readLines(ctx context.Context, filename string) <-chan string {
    out := make(chan string)

    go func() {
        defer close(out)

        file, err := os.Open(filename)
        if err != nil {
            fmt.Printf("Error opening file: %v\n", err)
            return
        }
        defer file.Close()

        scanner := bufio.NewScanner(file)
        for scanner.Scan() {
            line := scanner.Text()

            select {
            case out <- line:
            case <-ctx.Done():
                return
            }
        }
    }()

    return out
}

// Стадия 2: Фильтруем пустые строки и комментарии
func filterLines(ctx context.Context, in <-chan string) <-chan string {
    out := make(chan string)

    go func() {
        defer close(out)

        for line := range in {
            trimmed := strings.TrimSpace(line)

            // Пропускаем пустые и комментарии
            if trimmed == "" || strings.HasPrefix(trimmed, "#") {
                continue
            }

            select {
            case out <- trimmed:
            case <-ctx.Done():
                return
            }
        }
    }()

    return out
}

// Стадия 3: Преобразуем в верхний регистр
func toUpper(ctx context.Context, in <-chan string) <-chan string {
    out := make(chan string)

    go func() {
        defer close(out)

        for line := range in {
            upper := strings.ToUpper(line)

            select {
            case out <- upper:
            case <-ctx.Done():
                return
            }
        }
    }()

    return out
}

// Стадия 4: Подсчёт слов
func countWords(ctx context.Context, in <-chan string) <-chan int {
    out := make(chan int)

    go func() {
        defer close(out)

        for line := range in {
            words := strings.Fields(line)

            select {
            case out <- len(words):
            case <-ctx.Done():
                return
            }
        }
    }()

    return out
}

func main() {
    ctx := context.Background()

    // Построение pipeline
    lines := readLines(ctx, "input.txt")
    filtered := filterLines(ctx, lines)
    uppercase := toUpper(ctx, filtered)
    wordCounts := countWords(ctx, uppercase)

    // Обработка результатов
    totalWords := 0
    for count := range wordCounts {
        fmt.Printf("Line has %d words\n", count)
        totalWords += count
    }

    fmt.Printf("Total words: %d\n", totalWords)
}
```

## Fan-out, Fan-in в Pipeline

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func generator(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums {
            out <- n
        }
    }()
    return out
}

// Тяжёлая обработка
func process(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            time.Sleep(100 * time.Millisecond)  // Симуляция тяжёлой работы
            out <- n * n
        }
    }()
    return out
}

// Fan-out: запускаем несколько обработчиков
func fanOut(in <-chan int, workers int) []<-chan int {
    channels := make([]<-chan int, workers)

    for i := 0; i < workers; i++ {
        channels[i] = process(in)
    }

    return channels
}

// Fan-in: объединяем результаты
func fanIn(channels ...<-chan int) <-chan int {
    out := make(chan int)

    var wg sync.WaitGroup
    wg.Add(len(channels))

    // Читаем из каждого канала
    for _, ch := range channels {
        go func(c <-chan int) {
            defer wg.Done()
            for n := range c {
                out <- n
            }
        }(ch)
    }

    // Закрываем out когда все каналы прочитаны
    go func() {
        wg.Wait()
        close(out)
    }()

    return out
}

func main() {
    start := time.Now()

    // Pipeline
    numbers := generator(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

    // Fan-out: 5 параллельных обработчиков
    processors := fanOut(numbers, 5)

    // Fan-in: объединяем результаты
    results := fanIn(processors...)

    // Получаем результаты
    for result := range results {
        fmt.Println(result)
    }

    fmt.Printf("Time: %v\n", time.Since(start))
}
```

**Без fan-out/fan-in:** ~1 секунда (10 * 100ms)
**С fan-out/fan-in:** ~200ms (параллельная обработка)

## Вопрос 1: Проблема с этим pipeline?

```go
func generator() <-chan int {
    out := make(chan int)
    for i := 0; i < 10; i++ {
        out <- i  // Что не так?
    }
    close(out)
    return out
}
```

**Ответ:**

**Deadlock!**

**Проблема:**
- Небуферизированный канал
- Отправка `out <- i` блокируется пока кто-то не прочитает
- Но получателя ещё нет (функция не вернулась)
- Deadlock

**Правильно:**
```go
func generator() <-chan int {
    out := make(chan int)

    go func() {  // В отдельной горутине!
        defer close(out)
        for i := 0; i < 10; i++ {
            out <- i
        }
    }()

    return out
}
```

Или буферизированный канал:
```go
out := make(chan int, 10)
```

---

## Вопрос 2: Почему важно закрывать каналы?

**Ответ:**

Без закрытия канала:
```go
func stage(in <-chan int) <-chan int {
    out := make(chan int)

    go func() {
        // defer close(out)  ← Забыли

        for n := range in {
            out <- n * 2
        }
    }()

    return out
}
```

**Проблема:**
- Следующая стадия использует `for result := range out`
- `range` ждёт `close(out)` чтобы выйти
- Если не закрыть → горутина зависнет навсегда (goroutine leak)

**Правило:** Кто создаёт канал, тот его закрывает (обычно через `defer`)

---

## Вопрос 3: Как обработать ошибки в pipeline?

**Решение:**

**Вариант 1: Структура с ошибкой**
```go
type Result struct {
    Value int
    Err   error
}

func stage(in <-chan int) <-chan Result {
    out := make(chan Result)

    go func() {
        defer close(out)

        for n := range in {
            if n < 0 {
                out <- Result{Err: fmt.Errorf("negative number: %d", n)}
                continue
            }

            out <- Result{Value: n * 2}
        }
    }()

    return out
}

// Использование
for result := range stage(numbers) {
    if result.Err != nil {
        fmt.Printf("Error: %v\n", result.Err)
        continue
    }
    fmt.Println(result.Value)
}
```

**Вариант 2: Отдельный канал ошибок**
```go
type Pipeline struct {
    out    <-chan int
    errors <-chan error
}

func stage(in <-chan int) Pipeline {
    out := make(chan int)
    errors := make(chan error)

    go func() {
        defer close(out)
        defer close(errors)

        for n := range in {
            if n < 0 {
                errors <- fmt.Errorf("negative: %d", n)
                continue
            }
            out <- n * 2
        }
    }()

    return Pipeline{out: out, errors: errors}
}

// Использование
pipeline := stage(numbers)

go func() {
    for err := range pipeline.errors {
        fmt.Printf("Error: %v\n", err)
    }
}()

for value := range pipeline.out {
    fmt.Println(value)
}
```

---

## Best Practices

1. ✅ **Каждая стадия - в отдельной горутине**
   ```go
   go func() {
       defer close(out)
       for n := range in {
           out <- process(n)
       }
   }()
   ```

2. ✅ **Всегда закрывайте выходной канал**
   ```go
   defer close(out)
   ```

3. ✅ **Используйте однонаправленные каналы**
   ```go
   func stage(in <-chan T) <-chan T
   ```

4. ✅ **Поддерживайте cancellation через context**
   ```go
   select {
   case out <- value:
   case <-ctx.Done():
       return
   }
   ```

5. ✅ **Для нескольких workers используйте WaitGroup**
   ```go
   go func() {
       wg.Wait()
       close(out)
   }()
   ```

6. ✅ **Буферизируйте каналы если уместно**
   ```go
   out := make(chan int, 100)  // Уменьшает блокировки
   ```

7. ❌ **Не закрывайте канал несколько раз**
   ```go
   close(ch)
   close(ch)  // panic: close of closed channel
   ```

8. ❌ **Не отправляйте в закрытый канал**
   ```go
   close(ch)
   ch <- 1  // panic: send on closed channel
   ```

## Где спрашивают

- Яндекс
- Авито
- VK
- Т-Банк
- Практически на всех Go интервью

## Связанные темы

- [[Go - Каналы (channels)]]
- [[Go - Горутины (goroutines)]]
- [[Go - Select statement]]
- [[Go - Context]]
- [[Задача - Fan-in Fan-out]]
- [[Задача - Worker Pool]]
