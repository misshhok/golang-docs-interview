# Go - Race Condition и Data Race

Race condition и data race - две разные концепции, которые часто путают. Обе приводят к непредсказуемому поведению в конкурентных программах, но имеют разную природу.

## Data Race vs Race Condition

### Data Race (гонка данных)

**Data race** - это ситуация, когда два и более потока **одновременно** обращаются к одной области памяти, и **хотя бы один из них производит запись**.

```go
// Data race пример
var counter int

func main() {
    for i := 0; i < 1000; i++ {
        go func() {
            counter++  // Одновременный доступ!
        }()
    }

    time.Sleep(time.Second)
    fmt.Println(counter)  // Непредсказуемое значение (не 1000)
}
```

**Характеристики:**
- Проблема на уровне **памяти**
- Undefined behavior
- Может привести к повреждению данных
- Детектируется race detector'ом

### Race Condition (состояние гонки)

**Race condition** - это **логическая** ошибка, возникающая когда корректность программы зависит от порядка выполнения операций.

```go
// Race condition пример
var balance = 100

func withdraw(amount int) bool {
    // Горутина 1: проверяет баланс (100 >= 80) → true
    if balance >= amount {
        time.Sleep(10 * time.Millisecond)  // Симуляция задержки
        // Горутина 2: тоже проверила и прошла
        balance -= amount
        return true
    }
    return false
}

// Две горутины пытаются снять 80
go withdraw(80)
go withdraw(80)
// Результат: balance = -60 (должно быть 20 или 100)
```

**Характеристики:**
- Проблема на уровне **алгоритма/логики**
- Программа работает, но результат неверный
- Не всегда детектируется race detector'ом
- Нужна синхронизация операций

### Главное отличие

| Data Race | Race Condition |
|-----------|----------------|
| Одновременный доступ к памяти | Неправильный порядок операций |
| Undefined behavior | Логическая ошибка |
| Детектируется `-race` | Не всегда детектируется |
| Уровень памяти | Уровень алгоритма |

## Примеры Data Race

### Пример 1: Инкремент счётчика

```go
var counter int

func increment() {
    counter++  // НЕ атомарная операция!
}

func main() {
    for i := 0; i < 1000; i++ {
        go increment()
    }

    time.Sleep(time.Second)
    fmt.Println(counter)  // Не 1000!
}
```

**Почему data race:**
- `counter++` = 3 операции: READ, INCREMENT, WRITE
- Две горутины могут прочитать одно значение → обе увеличат на 1 → запишут одинаковое

```
Горутина 1: READ counter=5
Горутина 2: READ counter=5
Горутина 1: WRITE counter=6
Горутина 2: WRITE counter=6  ← Потеря одного инкремента!
```

### Пример 2: Concurrent Map Writes (PANIC!)

```go
func main() {
    m := make(map[int]int)

    for i := 0; i < 1000; i++ {
        go func(i int) {
            m[i] = i  // Data race!
        }(i)
    }

    time.Sleep(time.Second)
    fmt.Println(len(m))
}
```

**Результат:**
```
fatal error: concurrent map writes
```

Go **специально паникует** при конкурентной записи в map, чтобы предотвратить повреждение данных.

### Пример 3: Чтение и запись слайса

```go
func main() {
    nums := []int{1, 2, 3}

    // Горутина 1: читает
    go func() {
        for {
            _ = nums[0]
        }
    }()

    // Горутина 2: пишет
    go func() {
        for {
            nums[0] = 42
        }
    }()

    time.Sleep(time.Second)
}
```

Data race! Один читает, другой пишет одновременно.

### Пример 4: Замыкание в цикле

```go
func main() {
    for i := 0; i < 5; i++ {
        go func() {
            fmt.Print(i, " ")  // Data race на переменной i
        }()
    }

    time.Sleep(time.Second)
}
```

Все горутины читают одну и ту же переменную `i`, которую модифицирует цикл.

## Обнаружение Data Race с `-race`

Go предоставляет встроенный **race detector**:

```bash
# При запуске
go run -race main.go

# При тестировании
go test -race ./...

# При сборке
go build -race
```

### Пример вывода race detector

```go
var x int

func main() {
    go func() { x = 1 }()
    go func() { x = 2 }()
    time.Sleep(time.Second)
}
```

```bash
$ go run -race main.go
==================
WARNING: DATA RACE
Write at 0x00c000014098 by goroutine 7:
  main.main.func2()
      /main.go:8 +0x38

Previous write at 0x00c000014098 by goroutine 6:
  main.main.func1()
      /main.go:7 +0x38

Goroutine 7 (running) created at:
  main.main()
      /main.go:8 +0x7c

Goroutine 6 (finished) created at:
  main.main()
      /main.go:7 +0x64
==================
```

**Race detector показывает:**
- Где произошёл data race (адрес, строка кода)
- Какие горутины участвовали
- Stack trace обеих горутин

## Вопрос 1: Есть ли здесь data race?

```go
var config Config

func init() {
    config = loadConfig()  // Инициализация при старте
}

func main() {
    // 100 горутин читают config
    for i := 0; i < 100; i++ {
        go func() {
            fmt.Println(config.Port)
        }()
    }

    time.Sleep(time.Second)
}
```

<details>
<summary>Ответ</summary>

**Нет data race**, если `loadConfig()` вызывается до запуска горутин.

**Почему безопасно:**
- `init()` выполняется **до** `main()`
- Все горутины только **читают**
- Нет одновременных записей

**Но осторожно:** если где-то в коде есть `config = newConfig()`, то это data race!

**Правильно для изменяемого config:**
```go
var (
    config Config
    mu     sync.RWMutex
)

func getConfig() Config {
    mu.RLock()
    defer mu.RUnlock()
    return config
}

func setConfig(c Config) {
    mu.Lock()
    defer mu.Unlock()
    config = c
}
```
</details>

## Способы исправления Data Race

### 1. Mutex (взаимное исключение)

```go
var (
    counter int
    mu      sync.Mutex
)

func increment() {
    mu.Lock()
    counter++
    mu.Unlock()
}

func main() {
    for i := 0; i < 1000; i++ {
        go increment()
    }

    time.Sleep(time.Second)

    mu.Lock()
    fmt.Println(counter)  // 1000
    mu.Unlock()
}
```

**Преимущества:** Простота, универсальность
**Недостатки:** Может стать узким местом

### 2. Atomic операции

```go
var counter int64

func increment() {
    atomic.AddInt64(&counter, 1)
}

func main() {
    for i := 0; i < 1000; i++ {
        go increment()
    }

    time.Sleep(time.Second)
    fmt.Println(atomic.LoadInt64(&counter))  // 1000
}
```

**Преимущества:** Быстрее mutex для простых операций
**Недостатки:** Только для примитивных типов

### 3. Channels

```go
func main() {
    counter := 0
    done := make(chan bool)

    // Одна горутина владеет данными
    go func() {
        for i := 0; i < 1000; i++ {
            <-done
            counter++
        }
    }()

    // Остальные отправляют сигналы
    for i := 0; i < 1000; i++ {
        go func() {
            done <- true
        }()
    }

    time.Sleep(time.Second)
    fmt.Println(counter)
}
```

**Философия Go:** "Don't communicate by sharing memory; share memory by communicating"

### 4. sync.Map для конкурентных map

```go
func main() {
    var m sync.Map

    for i := 0; i < 1000; i++ {
        go func(i int) {
            m.Store(i, i)  // Безопасно
        }(i)
    }

    time.Sleep(time.Second)

    count := 0
    m.Range(func(key, value interface{}) bool {
        count++
        return true
    })
    fmt.Println(count)
}
```

### 5. Один владелец данных

```go
type Counter struct {
    value int
    ops   chan func(*Counter)
}

func NewCounter() *Counter {
    c := &Counter{
        ops: make(chan func(*Counter)),
    }

    go func() {
        for op := range c.ops {
            op(c)  // Только эта горутина изменяет value
        }
    }()

    return c
}

func (c *Counter) Increment() {
    c.ops <- func(c *Counter) {
        c.value++
    }
}

func (c *Counter) Value() int {
    result := make(chan int)
    c.ops <- func(c *Counter) {
        result <- c.value
    }
    return <-result
}

func main() {
    counter := NewCounter()

    for i := 0; i < 1000; i++ {
        go counter.Increment()
    }

    time.Sleep(time.Second)
    fmt.Println(counter.Value())
}
```

**Паттерн "Single Owner"**: только одна горутина имеет доступ к данным напрямую.

## Примеры Race Condition (без data race)

### Пример 1: Check-then-act

```go
var (
    balance = 100
    mu      sync.Mutex
)

func withdraw(amount int) bool {
    mu.Lock()
    canWithdraw := balance >= amount
    mu.Unlock()

    if canWithdraw {
        time.Sleep(10 * time.Millisecond)  // Симуляция обработки

        mu.Lock()
        balance -= amount  // ← Race condition!
        mu.Unlock()

        return true
    }
    return false
}
```

**Data race нет** (используем mutex), но **race condition есть**:
- Горутина 1: проверяет balance=100, canWithdraw=true
- Горутина 2: проверяет balance=100, canWithdraw=true
- Горутина 1: снимает 80, balance=20
- Горутина 2: снимает 80, balance=-60 ← Ошибка!

**Правильно:**
```go
func withdraw(amount int) bool {
    mu.Lock()
    defer mu.Unlock()

    if balance >= amount {
        balance -= amount
        return true
    }
    return false
}
```

Критическая секция должна включать **всю последовательность** операций.

### Пример 2: Двойная проверка (Double-checked locking)

```go
var (
    instance *Singleton
    mu       sync.Mutex
)

func GetInstance() *Singleton {
    if instance == nil {  // Первая проверка без блокировки
        mu.Lock()
        if instance == nil {  // Вторая проверка с блокировкой
            instance = &Singleton{}
        }
        mu.Unlock()
    }
    return instance
}
```

**Проблема:** Компилятор/CPU может переупорядочить операции. Другая горутина может увидеть `instance != nil`, но с неинициализированными полями.

**Правильно в Go:**
```go
var (
    instance *Singleton
    once     sync.Once
)

func GetInstance() *Singleton {
    once.Do(func() {
        instance = &Singleton{}
    })
    return instance
}
```

## Вопрос 2: Какая проблема в этом коде?

```go
type Cache struct {
    data map[string]string
    mu   sync.RWMutex
}

func (c *Cache) Get(key string) (string, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()

    val, ok := c.data[key]
    if !ok {
        c.mu.RUnlock()  // Освобождаем read lock
        c.mu.Lock()     // Берём write lock
        c.data[key] = fetchFromDB(key)
        val = c.data[key]
        c.mu.Unlock()
        c.mu.RLock()    // Снова берём read lock для defer
    }

    return val, ok
}
```

<details>
<summary>Ответ</summary>

**Проблема:** Race condition + излишняя сложность

**Что не так:**
1. Между `RUnlock()` и `Lock()` другая горутина может записать значение → дублирование запросов в БД
2. Манипуляции с локами внутри критической секции - легко ошибиться
3. `defer c.mu.RUnlock()` в начале вызовется дважды

**Правильный паттерн (Cache-aside with double-check):**
```go
func (c *Cache) Get(key string) string {
    // Быстрый путь: проверка с read lock
    c.mu.RLock()
    val, ok := c.data[key]
    c.mu.RUnlock()

    if ok {
        return val
    }

    // Медленный путь: загрузка с write lock
    c.mu.Lock()
    defer c.mu.Unlock()

    // Двойная проверка: может уже загрузили
    val, ok = c.data[key]
    if ok {
        return val
    }

    val = fetchFromDB(key)
    c.data[key] = val
    return val
}
```
</details>

## Best Practices

1. ✅ **Всегда запускайте тесты с `-race`**
   ```bash
   go test -race -short ./...
   ```

2. ✅ **Используйте правильный инструмент для задачи**
   - Простые счётчики → `sync/atomic`
   - Защита секции кода → `sync.Mutex`
   - Конкурентный доступ к map → `sync.Map`
   - Передача данных между горутинами → channels

3. ✅ **"Share memory by communicating"**
   ```go
   // ❌ Sharing memory
   var data []int
   var mu sync.Mutex

   // ✅ Communicating
   dataChan := make(chan []int)
   ```

4. ✅ **Держите критические секции короткими**
   ```go
   // ❌ Плохо: долгая операция под замком
   mu.Lock()
   result := expensiveComputation()
   cache[key] = result
   mu.Unlock()

   // ✅ Хорошо
   result := expensiveComputation()
   mu.Lock()
   cache[key] = result
   mu.Unlock()
   ```

5. ✅ **Используйте `defer` для unlock**
   ```go
   mu.Lock()
   defer mu.Unlock()
   // Код не может забыть разблокировать
   ```

6. ✅ **Документируйте какой mutex защищает какие данные**
   ```go
   type Server struct {
       mu       sync.Mutex
       clients  map[int]*Client  // protected by mu
       nextID   int              // protected by mu
   }
   ```

7. ❌ **Не копируйте sync примитивы**
   ```go
   type Config struct {
       mu sync.Mutex
   }

   c := Config{}
   c2 := c  // ❌ Копирует mutex!

   // ✅ Используйте указатель
   c := &Config{}
   ```

8. ✅ **Race detector не находит всё**
   - Детектирует только data races
   - Не детектирует race conditions
   - Требует выполнения проблемного кода
   - Не гарантия отсутствия проблем

## Вопросы с собеседований

**Вопрос 1:** В чём разница между race condition и data race?

**Ответ:**
- **Data race** - одновременный доступ к памяти (хотя бы один пишет), undefined behavior, детектируется `-race`
- **Race condition** - логическая ошибка из-за порядка операций, программа работает но неправильно, не всегда детектируется

**Вопрос 2:** Безопасно ли читать переменную из нескольких горутин без синхронизации?

**Ответ:**
- Если **никто не пишет** → безопасно
- Если **кто-то пишет** → data race (даже если остальные только читают)
- Используйте `sync.RWMutex` (RLock для чтения, Lock для записи) или `atomic.Load/Store`

**Вопрос 3:** Почему map не thread-safe в Go?

**Ответ:**
- Производительность: внутренняя синхронизация замедлила бы все операции
- Философия Go: явная синхронизация лучше неявной
- Когда нужна thread-safety → используйте `sync.Map` или защитите mutex'ом
- Map специально паникует при конкурентном доступе для раннего обнаружения проблем

**Вопрос 4:** Что делает race detector и как его использовать?

**Ответ:**
- Инструментирует код для отслеживания доступа к памяти
- Детектирует одновременный доступ где хотя бы один - запись
- Использование: `go test -race ./...` или `go run -race main.go`
- Overhead: ~10x медленнее, увеличение памяти в 5-10 раз
- Важно: находит только data races которые выполнились во время теста

## Связанные темы

- [[Go - Mutex и RWMutex]]
- [[Go - Атомарные операции]]
- [[Go - Deadlock]]
- [[Go - Горутины (goroutines)]]
- [[Go - Каналы (channels)]]
- [[Go - WaitGroup и Once]]
