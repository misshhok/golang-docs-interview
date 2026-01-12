# Задача - Custom WaitGroup

Продвинутая задача на собеседованиях в Яндекс и других BigTech компаниях. Проверяет глубокое понимание каналов и примитивов синхронизации.

## Условие задачи

```
Реализовать аналог sync.WaitGroup используя только каналы.
Должны поддерживаться методы:
- Add(delta int) — увеличить счётчик
- Done() — уменьшить счётчик на 1
- Wait() — заблокироваться до обнуления счётчика
```

## Решение 1: Через буферизированный канал (семафор)

```go
type WaitGroup struct {
    sem chan struct{}
}

func NewWaitGroup() *WaitGroup {
    return &WaitGroup{
        sem: make(chan struct{}, 1000), // Максимум 1000 горутин
    }
}

func (wg *WaitGroup) Add(delta int) {
    for i := 0; i < delta; i++ {
        wg.sem <- struct{}{}
    }
}

func (wg *WaitGroup) Done() {
    <-wg.sem
}

func (wg *WaitGroup) Wait() {
    for {
        select {
        case wg.sem <- struct{}{}:
            // Если смогли записать — канал не полон
            // Проверяем, можем ли записать до capacity
            <-wg.sem
        default:
            // Канал пуст — все Done() вызваны
            if len(wg.sem) == 0 {
                return
            }
        }
        runtime.Gosched()
    }
}
```

## Решение 2: Через отдельную горутину-координатор

```go
type WaitGroup struct {
    add  chan int
    done chan struct{}
    wait chan struct{}
}

func NewWaitGroup() *WaitGroup {
    wg := &WaitGroup{
        add:  make(chan int),
        done: make(chan struct{}),
        wait: make(chan struct{}),
    }

    go wg.coordinator()
    return wg
}

func (wg *WaitGroup) coordinator() {
    counter := 0
    waiters := []chan struct{}{}

    for {
        select {
        case delta := <-wg.add:
            counter += delta
            if counter < 0 {
                panic("negative WaitGroup counter")
            }
            if counter == 0 {
                // Разблокируем всех ожидающих
                for _, w := range waiters {
                    close(w)
                }
                waiters = nil
            }

        case <-wg.done:
            counter--
            if counter < 0 {
                panic("negative WaitGroup counter")
            }
            if counter == 0 {
                for _, w := range waiters {
                    close(w)
                }
                waiters = nil
            }

        case wg.wait <- struct{}{}:
            if counter == 0 {
                // Уже ноль — сразу разблокируем
                continue
            }
            // Создаём канал для этого waiter
            waiter := make(chan struct{})
            waiters = append(waiters, waiter)
            // Ждём пока канал закроется
            <-waiter
        }
    }
}

func (wg *WaitGroup) Add(delta int) {
    wg.add <- delta
}

func (wg *WaitGroup) Done() {
    wg.done <- struct{}{}
}

func (wg *WaitGroup) Wait() {
    wg.wait <- struct{}{}
}
```

## Решение 3: Простое и корректное

```go
type WaitGroup struct {
    counter int64
    done    chan struct{}
    mu      sync.Mutex
}

func NewWaitGroup() *WaitGroup {
    return &WaitGroup{
        done: make(chan struct{}),
    }
}

func (wg *WaitGroup) Add(delta int) {
    wg.mu.Lock()
    wg.counter += int64(delta)
    if wg.counter < 0 {
        wg.mu.Unlock()
        panic("negative WaitGroup counter")
    }
    wg.mu.Unlock()
}

func (wg *WaitGroup) Done() {
    wg.mu.Lock()
    wg.counter--
    if wg.counter == 0 {
        close(wg.done)
        wg.done = make(chan struct{}) // Для повторного использования
    } else if wg.counter < 0 {
        wg.mu.Unlock()
        panic("negative WaitGroup counter")
    }
    wg.mu.Unlock()
}

func (wg *WaitGroup) Wait() {
    wg.mu.Lock()
    if wg.counter == 0 {
        wg.mu.Unlock()
        return
    }
    done := wg.done
    wg.mu.Unlock()

    <-done
}
```

## Решение 4: Только каналы (чистое решение)

```go
type WaitGroup struct {
    tokens chan struct{}
    waiter chan chan struct{}
}

func NewWaitGroup() *WaitGroup {
    wg := &WaitGroup{
        tokens: make(chan struct{}, 10000),
        waiter: make(chan chan struct{}),
    }

    // Горутина для обработки Wait
    go func() {
        for waitCh := range wg.waiter {
            // Ждём пока tokens станет пустым
            for len(wg.tokens) > 0 {
                runtime.Gosched()
            }
            close(waitCh)
        }
    }()

    return wg
}

func (wg *WaitGroup) Add(delta int) {
    for i := 0; i < delta; i++ {
        wg.tokens <- struct{}{}
    }
}

func (wg *WaitGroup) Done() {
    <-wg.tokens
}

func (wg *WaitGroup) Wait() {
    waitCh := make(chan struct{})
    wg.waiter <- waitCh
    <-waitCh
}
```

## Пример использования

```go
func main() {
    wg := NewWaitGroup()

    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func(n int) {
            defer wg.Done()
            time.Sleep(time.Duration(n) * 100 * time.Millisecond)
            fmt.Printf("Worker %d done\n", n)
        }(i)
    }

    wg.Wait()
    fmt.Println("All workers completed")
}

// Output:
// Worker 0 done
// Worker 1 done
// Worker 2 done
// Worker 3 done
// Worker 4 done
// All workers completed
```

## Сравнение с sync.WaitGroup

| Аспект | sync.WaitGroup | Custom (каналы) |
|--------|----------------|-----------------|
| Производительность | Высокая (атомарные операции) | Ниже (каналы) |
| Сложность | Скрыта в runtime | Явная |
| Повторное использование | После Wait нужен новый Add | Зависит от реализации |
| Thread-safety | Гарантирована | Нужно обеспечить |

## Типичные ошибки

### Ошибка 1: Гонка между Add и Wait

```go
// ❌ НЕПРАВИЛЬНО: Wait может начаться раньше Add
go func() {
    wg.Add(1)  // Может выполниться после Wait!
    // ...
}()
wg.Wait()

// ✅ ПРАВИЛЬНО: Add до запуска горутины
wg.Add(1)
go func() {
    defer wg.Done()
    // ...
}()
wg.Wait()
```

### Ошибка 2: Отрицательный счётчик

```go
// ❌ НЕПРАВИЛЬНО: больше Done чем Add
wg.Add(1)
wg.Done()
wg.Done() // panic: negative counter!
```

### Ошибка 3: Deadlock при неправильной реализации

```go
// ❌ НЕПРАВИЛЬНО: Wait блокирует сам себя
func (wg *WaitGroup) Wait() {
    wg.mu.Lock()
    for wg.counter > 0 {
        // Держим lock и ждём — Done тоже ждёт lock!
    }
    wg.mu.Unlock()
}
```

## Вопросы с собеседований

### Вопрос 1: Почему sync.WaitGroup быстрее канальной реализации?

**Ответ:**

sync.WaitGroup использует:
- Атомарные операции (atomic.AddInt64)
- Семафор уровня runtime
- Минимум системных вызовов

Каналы требуют:
- Захват mutex канала
- Копирование данных
- Возможное переключение контекста горутины

---

### Вопрос 2: Можно ли реализовать Add с отрицательным delta?

**Ответ:**

Да, sync.WaitGroup поддерживает `Add(-1)` как эквивалент `Done()`.

Но нужно проверять, что счётчик не становится отрицательным — это паника.

```go
func (wg *WaitGroup) Add(delta int) {
    // ...
    if counter < 0 {
        panic("negative WaitGroup counter")
    }
}
```

---

### Вопрос 3: Как сделать WaitGroup переиспользуемым?

**Ответ:**

После Wait нужно сбросить состояние:

```go
func (wg *WaitGroup) Done() {
    wg.mu.Lock()
    wg.counter--
    if wg.counter == 0 {
        close(wg.done)
        wg.done = make(chan struct{}) // Новый канал для следующего цикла
    }
    wg.mu.Unlock()
}
```

---

## Связанные темы

- [[Go - WaitGroup и Once]]
- [[Go - Каналы (channels)]]
- [[Go - Mutex и RWMutex]]
- [[Go - Атомарные операции]]
- [[Задача - Semaphore pattern]]
