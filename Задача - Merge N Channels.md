# Задача - Merge N Channels

Классическая задача на собеседованиях в Яндекс, Авито, МТС. Проверяет понимание каналов, WaitGroup и паттерна мультиплексирования.

## Условие задачи

```
Написать функцию, которая объединяет N входных каналов в один выходной.
Данные из всех входных каналов должны попадать в выходной канал.
Выходной канал должен закрыться, когда все входные каналы закроются.
```

```go
func MergeChannels(channels ...<-chan int) <-chan int
```

## Решение 1: WaitGroup

```go
func MergeChannels(channels ...<-chan int) <-chan int {
    out := make(chan int)

    var wg sync.WaitGroup

    // Для каждого входного канала запускаем горутину
    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan int) {
            defer wg.Done()
            for val := range c {
                out <- val
            }
        }(ch)
    }

    // Закрываем выходной канал когда все входные закрылись
    go func() {
        wg.Wait()
        close(out)
    }()

    return out
}
```

### Использование

```go
func main() {
    ch1 := make(chan int)
    ch2 := make(chan int)
    ch3 := make(chan int)

    // Запускаем производителей
    go func() {
        for i := 0; i < 5; i++ {
            ch1 <- i
        }
        close(ch1)
    }()

    go func() {
        for i := 10; i < 15; i++ {
            ch2 <- i
        }
        close(ch2)
    }()

    go func() {
        for i := 100; i < 103; i++ {
            ch3 <- i
        }
        close(ch3)
    }()

    // Объединяем каналы
    merged := MergeChannels(ch1, ch2, ch3)

    // Читаем все значения
    for val := range merged {
        fmt.Println(val)
    }
}

// Вывод (порядок может отличаться):
// 0, 10, 100, 1, 11, 101, 2, 12, 102, 3, 13, 4, 14
```

## Решение 2: С использованием reflect.Select

Для случаев когда количество каналов неизвестно на этапе компиляции:

```go
import "reflect"

func MergeChannelsReflect(channels ...<-chan int) <-chan int {
    out := make(chan int)

    go func() {
        defer close(out)

        // Создаём slice SelectCase для reflect.Select
        cases := make([]reflect.SelectCase, len(channels))
        for i, ch := range channels {
            cases[i] = reflect.SelectCase{
                Dir:  reflect.SelectRecv,
                Chan: reflect.ValueOf(ch),
            }
        }

        // Пока есть открытые каналы
        for len(cases) > 0 {
            // Выбираем готовый канал
            chosen, value, ok := reflect.Select(cases)
            if !ok {
                // Канал закрыт — удаляем его из списка
                cases = append(cases[:chosen], cases[chosen+1:]...)
                continue
            }

            out <- int(value.Int())
        }
    }()

    return out
}
```

## Решение 3: С буферизацией (оптимизация)

```go
func MergeChannelsBuffered(channels ...<-chan int) <-chan int {
    // Буферизированный канал снижает блокировки
    out := make(chan int, len(channels))

    var wg sync.WaitGroup

    output := func(c <-chan int) {
        defer wg.Done()
        for val := range c {
            out <- val
        }
    }

    wg.Add(len(channels))
    for _, ch := range channels {
        go output(ch)
    }

    go func() {
        wg.Wait()
        close(out)
    }()

    return out
}
```

## Решение 4: С контекстом для отмены

```go
func MergeChannelsWithContext(ctx context.Context, channels ...<-chan int) <-chan int {
    out := make(chan int)

    var wg sync.WaitGroup

    output := func(c <-chan int) {
        defer wg.Done()
        for {
            select {
            case <-ctx.Done():
                return
            case val, ok := <-c:
                if !ok {
                    return
                }
                select {
                case out <- val:
                case <-ctx.Done():
                    return
                }
            }
        }
    }

    wg.Add(len(channels))
    for _, ch := range channels {
        go output(ch)
    }

    go func() {
        wg.Wait()
        close(out)
    }()

    return out
}
```

## Типичные ошибки

### Ошибка 1: Забыть закрыть выходной канал

```go
// ❌ НЕПРАВИЛЬНО: deadlock — читатель будет ждать вечно
func MergeBad(channels ...<-chan int) <-chan int {
    out := make(chan int)

    for _, ch := range channels {
        go func(c <-chan int) {
            for val := range c {
                out <- val
            }
        }(ch)
    }

    // out никогда не закроется!
    return out
}
```

### Ошибка 2: Захват переменной цикла

```go
// ❌ НЕПРАВИЛЬНО: все горутины читают из последнего канала
func MergeBad(channels ...<-chan int) <-chan int {
    out := make(chan int)
    var wg sync.WaitGroup

    for _, ch := range channels {
        wg.Add(1)
        go func() {  // ch захвачена по ссылке!
            defer wg.Done()
            for val := range ch {  // ❌ ch меняется!
                out <- val
            }
        }()
    }
    // ...
}

// ✅ ПРАВИЛЬНО: передаём как параметр
go func(c <-chan int) {
    // ...
}(ch)
```

### Ошибка 3: Закрытие канала раньше времени

```go
// ❌ НЕПРАВИЛЬНО: close до завершения горутин
func MergeBad(channels ...<-chan int) <-chan int {
    out := make(chan int)

    for _, ch := range channels {
        go func(c <-chan int) {
            for val := range c {
                out <- val  // panic: send on closed channel
            }
        }(ch)
    }

    close(out)  // ❌ Закрываем сразу!
    return out
}
```

## Визуализация

```
ch1: [1, 2, 3] ──────┐
                     │
ch2: [10, 20] ───────┼────▶ merged: [1, 10, 100, 2, 20, 3, ...]
                     │
ch3: [100] ──────────┘

Горутины:           WaitGroup
┌──────────┐          │
│ reader 1 │──────────┤
└──────────┘          │
┌──────────┐          │──▶ wg.Wait() ──▶ close(out)
│ reader 2 │──────────┤
└──────────┘          │
┌──────────┐          │
│ reader 3 │──────────┘
└──────────┘
```

## Сложность

| Операция | Время | Память |
|----------|-------|--------|
| Merge N каналов | O(total elements) | O(N) горутин |

## Вопросы с собеседований

### Вопрос 1: Почему нужен WaitGroup?

<details>
<summary>Ответ</summary>

WaitGroup нужен чтобы узнать, когда все входные каналы закрылись и можно закрыть выходной канал.

Без WaitGroup:
- Либо выходной канал никогда не закроется (deadlock)
- Либо закроется раньше времени (panic при записи)

</details>

### Вопрос 2: Что если один канал заблокируется?

<details>
<summary>Ответ</summary>

Если один входной канал заблокируется (никто не пишет), его горутина будет ждать на `range`. Это не заблокирует другие каналы — каждый читается своей горутиной.

Для защиты от зависания можно использовать версию с context и таймаутом.

</details>

### Вопрос 3: Можно ли сделать без горутин?

<details>
<summary>Ответ</summary>

Без горутин нельзя эффективно читать из N каналов одновременно.

Можно использовать `reflect.Select` в одной горутине, но это медленнее и сложнее. Стандартный `select` не поддерживает динамическое количество case.

</details>

### Вопрос 4: Гарантируется ли порядок?

<details>
<summary>Ответ</summary>

Нет. Порядок значений в выходном канале не гарантируется — зависит от планировщика горутин.

Если нужен порядок — используйте heap или сортировку, но это другая задача (Merge K Sorted Streams).

</details>

## Связанные темы

- [[Go - Каналы (channels)]]
- [[Go - WaitGroup и Once]]
- [[Задача - Fan-in Fan-out]]
- [[Задача - Pipeline pattern]]
- [[Go - Select statement]]
