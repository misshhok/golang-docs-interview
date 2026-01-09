# Go - Deadlock

Deadlock (взаимная блокировка) - ситуация, когда две или более горутины бесконечно ждут друг друга, и программа не может продолжить выполнение. Go runtime детектирует deadlock и завершает программу с паникой.

## Что такое Deadlock

**Deadlock** возникает когда:
- Все горутины заблокированы
- Никто не может продолжить выполнение
- Программа "зависла" навсегда

### Сообщение от Go runtime

```go
fatal error: all goroutines are asleep - deadlock!
```

Go runtime **автоматически обнаруживает** deadlock, когда все горутины заблокированы и ни одна не может продолжить работу.

## Условия Coffman для Deadlock

Deadlock возможен только если **все четыре условия** выполняются одновременно:

1. **Mutual Exclusion (взаимное исключение)**
   - Ресурс может использовать только один поток
   - Пример: mutex, канал

2. **Hold and Wait (удержание и ожидание)**
   - Поток держит ресурс и ждёт другой
   - Пример: держим mutex A, ждём mutex B

3. **No Preemption (отсутствие вытеснения)**
   - Ресурс нельзя забрать силой
   - Только владелец может освободить

4. **Circular Wait (циклическое ожидание)**
   - Горутина 1 ждёт 2, 2 ждёт 3, ..., N ждёт 1
   - Образуется цикл зависимостей

**Чтобы предотвратить deadlock**, нужно нарушить хотя бы одно условие.

## Примеры Deadlock в Go

### Пример 1: Небуферизированный канал без получателя

```go
func main() {
    ch := make(chan int)
    ch <- 1  // Блокируется навсегда
    fmt.Println("Done")
}
```

**Результат:**
```
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan send]:
main.main()
    /main.go:5 +0x37
```

**Почему deadlock:**
- Небуферизированный канал требует одновременно отправителя и получателя
- `ch <- 1` блокируется в ожидании получателя
- Получателя нет → deadlock

**Исправление:**
```go
// Вариант 1: Отправка в горутине
func main() {
    ch := make(chan int)

    go func() {
        ch <- 1
    }()

    fmt.Println(<-ch)
}

// Вариант 2: Буферизированный канал
func main() {
    ch := make(chan int, 1)
    ch <- 1  // Не блокируется
    fmt.Println(<-ch)
}
```

### Пример 2: Чтение из пустого канала

```go
func main() {
    ch := make(chan int)
    val := <-ch  // Блокируется навсегда
    fmt.Println(val)
}
```

**Результат:** `fatal error: all goroutines are asleep - deadlock!`

**Почему:** Ждём значение из канала, но никто туда не пишет.

### Пример 3: Двойная блокировка одного mutex

```go
func main() {
    var mu sync.Mutex

    mu.Lock()
    fmt.Println("First lock")

    mu.Lock()  // Пытаемся заблокировать снова
    fmt.Println("Second lock")  // Никогда не выполнится

    mu.Unlock()
}
```

**Результат:** `fatal error: all goroutines are asleep - deadlock!`

**Почему:**
- Mutex уже заблокирован этой же горутиной
- Повторный `Lock()` ждёт освобождения
- Но освободить может только эта горутина → deadlock

**Важно:** `sync.Mutex` в Go **не рекурсивный**!

### Пример 4: Циклическая блокировка mutex

```go
func main() {
    var mu1, mu2 sync.Mutex

    // Горутина 1
    go func() {
        mu1.Lock()
        fmt.Println("Goroutine 1: locked mu1")
        time.Sleep(100 * time.Millisecond)

        fmt.Println("Goroutine 1: waiting for mu2")
        mu2.Lock()  // Ждёт mu2
        fmt.Println("Goroutine 1: locked mu2")

        mu2.Unlock()
        mu1.Unlock()
    }()

    // Горутина 2
    go func() {
        mu2.Lock()
        fmt.Println("Goroutine 2: locked mu2")
        time.Sleep(100 * time.Millisecond)

        fmt.Println("Goroutine 2: waiting for mu1")
        mu1.Lock()  // Ждёт mu1
        fmt.Println("Goroutine 2: locked mu1")

        mu1.Unlock()
        mu2.Unlock()
    }()

    time.Sleep(time.Second)
}
```

**Вывод:**
```
Goroutine 1: locked mu1
Goroutine 2: locked mu2
Goroutine 1: waiting for mu2
Goroutine 2: waiting for mu1
fatal error: all goroutines are asleep - deadlock!
```

**Почему deadlock:**
- Горутина 1 держит `mu1`, ждёт `mu2`
- Горутина 2 держит `mu2`, ждёт `mu1`
- Circular wait → deadlock

**Исправление:** Всегда блокируйте mutex'ы в одном порядке:
```go
// Всегда: сначала mu1, потом mu2
func operation1() {
    mu1.Lock()
    mu2.Lock()
    // ...
    mu2.Unlock()
    mu1.Unlock()
}

func operation2() {
    mu1.Lock()  // Тот же порядок!
    mu2.Lock()
    // ...
    mu2.Unlock()
    mu1.Unlock()
}
```

### Пример 5: Deadlock с WaitGroup

```go
func main() {
    var wg sync.WaitGroup

    wg.Add(1)

    go func() {
        fmt.Println("Working...")
        // Забыли вызвать wg.Done()
    }()

    wg.Wait()  // Ждёт вечно
    fmt.Println("Done")
}
```

**Результат:** `fatal error: all goroutines are asleep - deadlock!`

**Исправление:**
```go
go func() {
    defer wg.Done()  // Всегда используйте defer
    fmt.Println("Working...")
}()
```

### Пример 6: Взаимное ожидание через каналы

```go
func main() {
    ch1 := make(chan int)
    ch2 := make(chan int)

    go func() {
        val := <-ch1  // Ждёт данные из ch1
        ch2 <- val    // Отправляет в ch2
    }()

    go func() {
        val := <-ch2  // Ждёт данные из ch2
        ch1 <- val    // Отправляет в ch1
    }()

    time.Sleep(time.Second)
}
```

**Результат:** Deadlock

**Почему:**
- Горутина 1 ждёт данные из `ch1`
- Горутина 2 ждёт данные из `ch2`
- Обе горутины заблокированы → никто не может начать → deadlock

### Пример 7: Закрытый канал в select

```go
func main() {
    ch := make(chan int)
    close(ch)

    for {
        select {
        case val, ok := <-ch:
            if !ok {
                // Канал закрыт, но select продолжает выбирать этот case!
                fmt.Println("Channel closed")
                // Нет break из for!
            } else {
                fmt.Println(val)
            }
        }
    }
}
```

**Проблема:** Бесконечный цикл, но не deadlock (горутина не заблокирована).

Чтение из закрытого канала возвращает zero value немедленно.

**Правильно:**
```go
for {
    select {
    case val, ok := <-ch:
        if !ok {
            return  // Выход из функции
        }
        fmt.Println(val)
    }
}
```

## Вопрос 1: Будет ли здесь deadlock?

```go
func main() {
    ch := make(chan int, 1)

    ch <- 1
    ch <- 2  // Эта строка

    fmt.Println(<-ch)
    fmt.Println(<-ch)
}
```

<details>
<summary>Ответ</summary>

**Да, deadlock!**

**Почему:**
- Канал буферизирован размером 1
- `ch <- 1` успешно → буфер заполнен
- `ch <- 2` блокируется → буфер полон, никто не читает
- Все горутины заблокированы → deadlock

**Правильно:**
```go
ch := make(chan int, 2)  // Увеличить буфер

// Или
ch <- 1
fmt.Println(<-ch)
ch <- 2
fmt.Println(<-ch)
```
</details>

## Вопрос 2: Deadlock или нет?

```go
func main() {
    var wg sync.WaitGroup
    ch := make(chan int)

    wg.Add(2)

    go func() {
        defer wg.Done()
        ch <- 1
    }()

    go func() {
        defer wg.Done()
        ch <- 2
    }()

    wg.Wait()
    fmt.Println(<-ch)
    fmt.Println(<-ch)
}
```

<details>
<summary>Ответ</summary>

**Да, deadlock!**

**Почему:**
- Обе горутины пытаются отправить в небуферизированный канал
- Обе блокируются (нет получателя)
- `wg.Wait()` ждёт пока горутины завершатся
- Но горутины не могут завершиться пока кто-то не прочитает из канала
- `<-ch` не выполнится пока `wg.Wait()` не завершится
- Circular dependency → deadlock

**Правильно:**
```go
go func() {
    defer wg.Done()
    ch <- 1
}()

go func() {
    defer wg.Done()
    ch <- 2
}()

// Читаем ДО Wait
val1 := <-ch
val2 := <-ch

wg.Wait()
fmt.Println(val1, val2)
```

Или используйте буферизированный канал:
```go
ch := make(chan int, 2)  // Горутины не блокируются
```
</details>

## Вопрос 3: Найдите потенциальный deadlock

```go
type Database struct {
    mu   sync.RWMutex
    data map[string]string
}

func (db *Database) Get(key string) string {
    db.mu.RLock()
    defer db.mu.RUnlock()

    if val, ok := db.data[key]; ok {
        return val
    }

    // Если нет в кэше, загрузить
    return db.LoadFromDisk(key)
}

func (db *Database) LoadFromDisk(key string) string {
    db.mu.Lock()  // ← Потенциальный deadlock!
    defer db.mu.Unlock()

    val := readFromDisk(key)
    db.data[key] = val
    return val
}
```

<details>
<summary>Ответ</summary>

**Deadlock при `Get()` → `LoadFromDisk()`**

**Почему:**
1. `Get()` берёт read lock (`RLock`)
2. `Get()` вызывает `LoadFromDisk()`
3. `LoadFromDisk()` пытается взять write lock (`Lock`)
4. Write lock ждёт пока все read lock'и освободятся
5. Но read lock держит та же горутина → deadlock

**Правильно:**
```go
func (db *Database) Get(key string) string {
    db.mu.RLock()
    val, ok := db.data[key]
    db.mu.RUnlock()  // Освободить ДО вызова LoadFromDisk

    if ok {
        return val
    }

    return db.LoadFromDisk(key)
}
```

Или с double-check:
```go
func (db *Database) Get(key string) string {
    // Быстрая проверка
    db.mu.RLock()
    val, ok := db.data[key]
    db.mu.RUnlock()

    if ok {
        return val
    }

    // Загрузка с write lock
    db.mu.Lock()
    defer db.mu.Unlock()

    // Double-check: может уже загрузили
    if val, ok := db.data[key]; ok {
        return val
    }

    val = readFromDisk(key)
    db.data[key] = val
    return val
}
```
</details>

## Обнаружение Deadlock

### 1. Go runtime детектор

Go автоматически обнаруживает deadlock когда **все** горутины заблокированы:

```go
fatal error: all goroutines are asleep - deadlock!
```

**Ограничения:**
- Детектирует только "полный" deadlock (все горутины спят)
- Не детектирует частичный deadlock (часть горутин работает)
- Срабатывает только в runtime, не на этапе компиляции

### 2. Частичный deadlock (не детектируется)

```go
func main() {
    ch := make(chan int)

    // Эта горутина работает вечно
    go func() {
        for {
            time.Sleep(time.Second)
        }
    }()

    // Эта заблокирована
    ch <- 1  // Deadlock, но Go runtime не обнаружит!
}
```

**Программа зависнет**, но не упадёт с паникой, т.к. есть активная горутина.

### 3. Инструменты для анализа

```bash
# Пр запуске с race detector (может помочь найти проблемы)
go run -race main.go

# Профилирование блокировок
go test -blockprofile=block.out
go tool pprof block.out
```

## Как избежать Deadlock

### 1. Упорядочение блокировок

```go
// ✅ Всегда блокировать в одном порядке
func transfer(from, to *Account, amount int) {
    // Определить порядок по ID
    first, second := from, to
    if from.ID > to.ID {
        first, second = to, from
    }

    first.mu.Lock()
    second.mu.Lock()
    defer second.mu.Unlock()
    defer first.mu.Unlock()

    from.balance -= amount
    to.balance += amount
}
```

### 2. Таймауты

```go
select {
case ch <- value:
    // Успешно отправили
case <-time.After(5 * time.Second):
    // Таймаут
    return errors.New("timeout")
}
```

### 3. TryLock (с Go 1.18+)

```go
if mu.TryLock() {
    defer mu.Unlock()
    // Работа
} else {
    // Не удалось заблокировать, попробуем позже
    return errors.New("resource busy")
}
```

### 4. Использование context для отмены

```go
func worker(ctx context.Context, ch chan int) {
    for {
        select {
        case val := <-ch:
            process(val)
        case <-ctx.Done():
            return  // Выход при отмене
        }
    }
}
```

### 5. Буферизированные каналы

```go
// ❌ Легко получить deadlock
ch := make(chan int)

// ✅ Буфер даёт "люфт"
ch := make(chan int, 100)
```

### 6. Неблокирующие операции с select

```go
select {
case ch <- value:
    // Отправили
default:
    // Канал занят, делаем что-то другое
}

select {
case val := <-ch:
    // Получили
default:
    // Канала пуст, продолжаем
}
```

### 7. Один владелец канала

```go
// Правило: кто создаёт канал, тот его закрывает
func producer() <-chan int {
    ch := make(chan int)

    go func() {
        defer close(ch)  // Закрываем здесь
        for i := 0; i < 10; i++ {
            ch <- i
        }
    }()

    return ch
}

func consumer(ch <-chan int) {
    for val := range ch {  // range автоматически выходит при close
        fmt.Println(val)
    }
}
```

### 8. Избегайте держать lock во время I/O

```go
// ❌ Плохо: держим lock во время I/O
mu.Lock()
data := cache[key]
if data == "" {
    data = fetchFromDB(key)  // Долгая операция!
    cache[key] = data
}
mu.Unlock()

// ✅ Хорошо: lock только для доступа к данным
mu.Lock()
data, ok := cache[key]
mu.Unlock()

if !ok {
    data = fetchFromDB(key)  // Без lock
    mu.Lock()
    cache[key] = data
    mu.Unlock()
}
```

## Практический пример: Dining Philosophers

Классическая задача на deadlock:

```go
type Fork struct {
    mu sync.Mutex
}

type Philosopher struct {
    id          int
    leftFork    *Fork
    rightFork   *Fork
}

// ❌ Возможен deadlock
func (p *Philosopher) dine() {
    p.leftFork.mu.Lock()
    p.rightFork.mu.Lock()

    fmt.Printf("Philosopher %d is eating\n", p.id)
    time.Sleep(time.Second)

    p.rightFork.mu.Unlock()
    p.leftFork.mu.Unlock()
}

// ✅ Решение: упорядочение
func (p *Philosopher) dine() {
    // Всегда берём вилки в порядке возрастания адресов
    first, second := p.leftFork, p.rightFork
    if uintptr(unsafe.Pointer(p.leftFork)) > uintptr(unsafe.Pointer(p.rightFork)) {
        first, second = p.rightFork, p.leftFork
    }

    first.mu.Lock()
    second.mu.Lock()

    fmt.Printf("Philosopher %d is eating\n", p.id)
    time.Sleep(time.Second)

    second.mu.Unlock()
    first.mu.Unlock()
}
```

## Best Practices

1. ✅ **Блокируйте mutex'ы в консистентном порядке**
   ```go
   // Всегда в порядке: A → B → C
   muA.Lock()
   muB.Lock()
   muC.Lock()
   defer muC.Unlock()
   defer muB.Unlock()
   defer muA.Unlock()
   ```

2. ✅ **Используйте defer для unlock**
   ```go
   mu.Lock()
   defer mu.Unlock()
   // Даже при panic unlock выполнится
   ```

3. ✅ **Не вызывайте внешние функции под lock'ом**
   ```go
   // ❌ Опасно
   mu.Lock()
   externalFunction()  // Может попытаться взять тот же lock
   mu.Unlock()
   ```

4. ✅ **Используйте буферизированные каналы**
   ```go
   ch := make(chan int, cap)  // Reduce blocking
   ```

5. ✅ **Применяйте таймауты**
   ```go
   ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
   defer cancel()
   ```

6. ✅ **Кто создаёт канал, тот его закрывает**
   ```go
   func produce() <-chan int {
       ch := make(chan int)
       go func() {
           defer close(ch)  // Владелец закрывает
           // ...
       }()
       return ch
   }
   ```

7. ✅ **Проверяйте WaitGroup counter**
   ```go
   wg.Add(1)
   go func() {
       defer wg.Done()  // ВСЕГДА через defer
       // ...
   }()
   ```

8. ❌ **Не используйте рекурсивные lock'и**
   ```go
   mu.Lock()
   mu.Lock()  // Deadlock! sync.Mutex не рекурсивный
   ```

## Вопросы с собеседований

**Вопрос 1:** Что такое deadlock и как Go его обнаруживает?

**Ответ:** Deadlock - взаимная блокировка, когда горутины бесконечно ждут друг друга. Go runtime обнаруживает deadlock когда **все** горутины заблокированы (никто не может продолжить). Выдаёт: `fatal error: all goroutines are asleep - deadlock!`

**Вопрос 2:** Назовите 4 условия Coffman для deadlock.

**Ответ:**
1. Mutual Exclusion - ресурс используется только одним потоком
2. Hold and Wait - держим ресурс и ждём другой
3. No Preemption - ресурс нельзя забрать силой
4. Circular Wait - циклическое ожидание

Чтобы предотвратить deadlock, нужно нарушить хотя бы одно условие.

**Вопрос 3:** Как избежать deadlock при работе с несколькими mutex'ами?

**Ответ:**
- Всегда блокировать mutex'ы в **одном и том же порядке**
- Например, по ID объекта или по адресу в памяти
- Это нарушает условие Circular Wait

**Вопрос 4:** В чём разница между deadlock и livelock?

**Ответ:**
- **Deadlock** - горутины заблокированы и ждут (не выполняются)
- **Livelock** - горутины активны, но не прогрессируют (например, постоянно пытаются взять lock и откатываются)

## Связанные темы

- [[Go - Каналы (channels)]]
- [[Go - Mutex и RWMutex]]
- [[Go - Race Condition и Data Race]]
- [[Go - Горутины (goroutines)]]
- [[Go - WaitGroup и Once]]
- [[Go - Context]]
- [[Go - Select statement]]
