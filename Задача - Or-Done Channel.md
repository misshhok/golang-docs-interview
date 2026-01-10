# Задача - Or-Done Channel

Важный паттерн конкурентности для Senior уровня. Позволяет читать из канала с возможностью отмены через done-канал.

## Условие задачи

```
Реализовать функцию OrDone, которая позволяет читать из канала
с возможностью прерывания через done-канал.

Когда done закрывается — OrDone должен прекратить читать из канала
и закрыть выходной канал.
```

```go
func OrDone(done <-chan struct{}, c <-chan interface{}) <-chan interface{}
```

## Проблема без OrDone

```go
// ❌ Проблема: что если c никогда не закроется?
for val := range c {
    // Если c заблокирован — мы застряли навсегда
    process(val)
}

// ❌ Проблема: громоздкий код
for {
    select {
    case <-done:
        return
    case val, ok := <-c:
        if !ok {
            return
        }
        process(val)
    }
}
```

## Решение

```go
func OrDone(done <-chan struct{}, c <-chan interface{}) <-chan interface{} {
    out := make(chan interface{})

    go func() {
        defer close(out)

        for {
            select {
            case <-done:
                return
            case val, ok := <-c:
                if !ok {
                    return
                }
                // Отправляем в out, но тоже с проверкой done
                select {
                case out <- val:
                case <-done:
                    return
                }
            }
        }
    }()

    return out
}
```

## Использование

```go
func main() {
    done := make(chan struct{})
    values := make(chan interface{})

    // Производитель
    go func() {
        defer close(values)
        for i := 0; i < 100; i++ {
            values <- i
            time.Sleep(100 * time.Millisecond)
        }
    }()

    // Через 500ms отменяем
    go func() {
        time.Sleep(500 * time.Millisecond)
        close(done)
    }()

    // Читаем через OrDone
    for val := range OrDone(done, values) {
        fmt.Println(val)
    }

    fmt.Println("Finished")
}

// Output:
// 0
// 1
// 2
// 3
// 4
// Finished
```

## Вариация: Generic версия (Go 1.18+)

```go
func OrDone[T any](done <-chan struct{}, c <-chan T) <-chan T {
    out := make(chan T)

    go func() {
        defer close(out)

        for {
            select {
            case <-done:
                return
            case val, ok := <-c:
                if !ok {
                    return
                }
                select {
                case out <- val:
                case <-done:
                    return
                }
            }
        }
    }()

    return out
}
```

## Комбинирование с другими паттернами

### OrDone + Pipeline

```go
func Pipeline(done <-chan struct{}, input <-chan int) <-chan int {
    out := make(chan int)

    go func() {
        defer close(out)

        // Используем OrDone для безопасного чтения
        for val := range OrDone(done, toInterface(input)) {
            result := process(val.(int))

            select {
            case out <- result:
            case <-done:
                return
            }
        }
    }()

    return out
}

func toInterface(c <-chan int) <-chan interface{} {
    out := make(chan interface{})
    go func() {
        defer close(out)
        for v := range c {
            out <- v
        }
    }()
    return out
}
```

### OrDone + Fan-Out

```go
func ProcessWithWorkers(done <-chan struct{}, input <-chan int, numWorkers int) <-chan int {
    results := make(chan int)
    var wg sync.WaitGroup

    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()

            for val := range OrDone(done, toInterface(input)) {
                result := heavyProcess(val.(int))

                select {
                case results <- result:
                case <-done:
                    return
                }
            }
        }()
    }

    go func() {
        wg.Wait()
        close(results)
    }()

    return results
}
```

## Паттерн Or (объединение done-каналов)

```go
// Or возвращает канал, который закрывается когда любой из входных закрывается
func Or(channels ...<-chan struct{}) <-chan struct{} {
    switch len(channels) {
    case 0:
        return nil
    case 1:
        return channels[0]
    }

    orDone := make(chan struct{})

    go func() {
        defer close(orDone)

        switch len(channels) {
        case 2:
            select {
            case <-channels[0]:
            case <-channels[1]:
            }
        default:
            select {
            case <-channels[0]:
            case <-channels[1]:
            case <-channels[2]:
            case <-Or(append(channels[3:], orDone)...):
            }
        }
    }()

    return orDone
}
```

### Использование Or

```go
func main() {
    done1 := make(chan struct{})
    done2 := make(chan struct{})
    done3 := make(chan struct{})

    // Закроем done2 через 2 секунды
    go func() {
        time.Sleep(2 * time.Second)
        close(done2)
    }()

    // Or вернёт канал, который закроется при закрытии любого из входных
    select {
    case <-Or(done1, done2, done3):
        fmt.Println("One of the channels closed")
    case <-time.After(5 * time.Second):
        fmt.Println("Timeout")
    }
}
// Output: One of the channels closed (через 2 секунды)
```

## Типичные ошибки

### Ошибка 1: Забыть проверить done при отправке

```go
// ❌ НЕПРАВИЛЬНО: можем заблокироваться на out <- val
func OrDoneBad(done <-chan struct{}, c <-chan interface{}) <-chan interface{} {
    out := make(chan interface{})

    go func() {
        defer close(out)
        for {
            select {
            case <-done:
                return
            case val, ok := <-c:
                if !ok {
                    return
                }
                out <- val  // ❌ Блокирующая отправка!
            }
        }
    }()

    return out
}

// ✅ ПРАВИЛЬНО: проверяем done при отправке
select {
case out <- val:
case <-done:
    return
}
```

### Ошибка 2: Не закрыть выходной канал

```go
// ❌ НЕПРАВИЛЬНО: потребитель будет ждать вечно
func OrDoneBad(done <-chan struct{}, c <-chan interface{}) <-chan interface{} {
    out := make(chan interface{})

    go func() {
        // defer close(out) — забыли!
        for {
            // ...
        }
    }()

    return out
}
```

## Зачем двойной select?

```go
// Почему так?
select {
case <-done:
    return
case val, ok := <-c:
    if !ok {
        return
    }
    select {           // <-- Зачем второй select?
    case out <- val:
    case <-done:
        return
    }
}
```

**Ответ:** Между получением `val` из `c` и отправкой в `out` может пройти время. За это время `done` может закрыться. Без второго select мы заблокируемся на `out <- val`.

## Вопросы с собеседований

### Вопрос 1: Зачем нужен OrDone если есть context?

<details>
<summary>Ответ</summary>

OrDone работает с `chan struct{}` — легковесная абстракция для отмены.

Context (`context.Context`) более мощный:
- Иерархия отмены (parent-child)
- Передача значений
- Таймауты и дедлайны

OrDone проще и используется когда достаточно простого done-канала, особенно в pipeline'ах.

</details>

### Вопрос 2: Можно ли использовать небуферизированный канал в OrDone?

<details>
<summary>Ответ</summary>

Да, в базовой реализации используется небуферизированный `out`. Это означает синхронную передачу — горутина OrDone заблокируется пока потребитель не прочитает значение.

Для асинхронности можно использовать буферизированный:
```go
out := make(chan interface{}, 1)
```

</details>

### Вопрос 3: Что произойдёт если c — nil?

<details>
<summary>Ответ</summary>

Чтение из nil канала блокируется навсегда. OrDone заблокируется на `case val, ok := <-c` и сможет выйти только через done.

Это валидное поведение — позволяет "отключать" источники данных, устанавливая канал в nil.

</details>

## Связанные темы

- [[Go - Каналы (channels)]]
- [[Go - Select statement]]
- [[Go - Context]]
- [[Задача - Pipeline pattern]]
- [[Задача - Graceful Shutdown]]
- [[Задача - Fan-in Fan-out]]
