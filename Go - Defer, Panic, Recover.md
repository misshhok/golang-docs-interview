# Go - Defer, Panic, Recover

Три механизма для управления потоком выполнения и обработки ошибок.

## defer

Откладывает выполнение функции до выхода из окружающей функции.

### Базовое использование

```go
func readFile(filename string) error {
    file, err := os.Open(filename)
    if err != nil {
        return err
    }
    defer file.Close() // Выполнится при выходе из функции

    // Работа с файлом
    data := make([]byte, 100)
    _, err = file.Read(data)
    return err
}
```

### Порядок выполнения: LIFO

```go
func example() {
    defer fmt.Println("1")
    defer fmt.Println("2")
    defer fmt.Println("3")
    fmt.Println("Start")
}

// Вывод:
// Start
// 3
// 2
// 1
```

**Defer выполняется в обратном порядке (стек)!**

### Когда выполняется defer

```go
func test() {
    defer fmt.Println("defer 1")

    if true {
        defer fmt.Println("defer 2")
        return // defer выполнится
    }

    defer fmt.Println("defer 3") // Не выполнится
}

// Вывод:
// defer 2
// defer 1
```

**defer выполняется при:**
- Нормальном return
- panic
- Достижении конца функции

### Аргументы defer вычисляются сразу

```go
func example() {
    i := 0
    defer fmt.Println(i) // Сохраняет 0
    i++
    // Вывод: 0 (не 1!)
}

// Для захвата текущего значения используйте замыкание
func example2() {
    i := 0
    defer func() {
        fmt.Println(i) // Замыкание - захватывает переменную
    }()
    i++
    // Вывод: 1
}
```

### Изменение возвращаемых значений

```go
// С именованными возвращаемыми значениями
func increment(x int) (result int) {
    defer func() {
        result++ // Изменяет возвращаемое значение!
    }()
    return x
}

fmt.Println(increment(5)) // 6 (не 5!)

// Без именованных - не работает
func increment2(x int) int {
    defer func() {
        x++ // НЕ влияет на return!
    }()
    return x
}

fmt.Println(increment2(5)) // 5
```

### Типичные применения

**1. Освобождение ресурсов:**
```go
func process() error {
    file, err := os.Open("file.txt")
    if err != nil {
        return err
    }
    defer file.Close()

    conn, err := db.Open()
    if err != nil {
        return err
    }
    defer conn.Close()

    // Используем file и conn
    return nil
}
```

**2. Mutex unlock:**
```go
func (c *Cache) Get(key string) interface{} {
    c.mu.Lock()
    defer c.mu.Unlock()

    return c.data[key]
}
```

**3. Логирование:**
```go
func operation() {
    start := time.Now()
    defer func() {
        log.Printf("Operation took %v", time.Since(start))
    }()

    // Операция...
}
```

**4. Восстановление после panic:**
```go
func safeOperation() {
    defer func() {
        if r := recover(); r != nil {
            log.Printf("Recovered from panic: %v", r)
        }
    }()

    // Опасная операция
}
```

## panic

Останавливает нормальное выполнение функции и начинает раскрутку стека (unwinding).

### Когда возникает panic

```go
// 1. Вручную
panic("something went wrong")

// 2. Runtime errors
var p *int
*p = 42 // panic: runtime error: invalid memory address

// 3. Выход за границы
arr := [3]int{1, 2, 3}
_ = arr[10] // panic: index out of range

// 4. Закрытый канал
ch := make(chan int)
close(ch)
ch <- 1 // panic: send on closed channel

// 5. Type assertion
var i interface{} = "hello"
n := i.(int) // panic: interface conversion

// 6. Nil map
var m map[string]int
m["key"] = 1 // panic: assignment to entry in nil map
```

### Использование panic

```go
func MustLoadConfig(filename string) Config {
    config, err := LoadConfig(filename)
    if err != nil {
        panic(err) // Критическая ошибка при старте
    }
    return config
}

// Используйте panic только для unrecoverable errors!
```

### Что происходит при panic

1. Текущая функция останавливается
2. Выполняются все defer текущей функции
3. Возврат к вызывающей функции
4. Повторяются шаги 2-3 вверх по стеку
5. Программа завершается (если не recover)

```go
func level3() {
    defer fmt.Println("defer level3")
    panic("error in level3")
}

func level2() {
    defer fmt.Println("defer level2")
    level3()
    fmt.Println("after level3") // Не выполнится
}

func level1() {
    defer fmt.Println("defer level1")
    level2()
}

func main() {
    level1()
}

// Вывод:
// defer level3
// defer level2
// defer level1
// panic: error in level3
```

## recover

Перехватывает panic и восстанавливает нормальное выполнение.

### Базовое использование

```go
func safeCall(fn func()) {
    defer func() {
        if r := recover(); r != nil {
            fmt.Printf("Recovered from panic: %v\n", r)
        }
    }()

    fn() // Может вызвать panic
}

safeCall(func() {
    panic("oops!")
})

fmt.Println("Program continues")
```

**recover работает ТОЛЬКО внутри defer!**

```go
// ❌ НЕ работает
func wrong() {
    if r := recover(); r != nil { // Всегда nil
        // ...
    }
    panic("test")
}

// ✅ Работает
func correct() {
    defer func() {
        if r := recover(); r != nil {
            // ...
        }
    }()
    panic("test")
}
```

### Возврат ошибки вместо panic

```go
func SafeDivide(a, b float64) (result float64, err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("panic: %v", r)
        }
    }()

    return divide(a, b), nil
}

func divide(a, b float64) float64 {
    if b == 0 {
        panic("division by zero")
    }
    return a / b
}

// Использование
result, err := SafeDivide(10, 0)
if err != nil {
    log.Println(err) // panic: division by zero
}
```

### Web сервер - recover middleware

```go
func RecoverMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                log.Printf("panic: %v\n%s", err, debug.Stack())
                http.Error(w, "Internal Server Error", 500)
            }
        }()

        next.ServeHTTP(w, r)
    })
}
```

### Горутины и recover

**Каждая горутина нуждается в своем recover!**

```go
// ❌ Recover в main не поймает panic из горутины
func main() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("Recovered:", r) // НЕ выполнится!
        }
    }()

    go func() {
        panic("panic in goroutine")
    }()

    time.Sleep(time.Second)
}

// ✅ Recover в каждой горутине
func main() {
    go func() {
        defer func() {
            if r := recover(); r != nil {
                fmt.Println("Recovered:", r) // Выполнится
            }
        }()

        panic("panic in goroutine")
    }()

    time.Sleep(time.Second)
}
```

## Паттерны

### 1. Cleanup resources

```go
func processFile(filename string) error {
    file, err := os.Open(filename)
    if err != nil {
        return err
    }
    defer file.Close()

    // Множественные defer для cleanup
    lock.Lock()
    defer lock.Unlock()

    conn, err := db.Connect()
    if err != nil {
        return err
    }
    defer conn.Close()

    // Работа...
    return nil
}
```

### 2. Измерение времени выполнения

```go
func track(name string) func() {
    start := time.Now()
    return func() {
        log.Printf("%s took %v", name, time.Since(start))
    }
}

func operation() {
    defer track("operation")()

    // Операция...
}
```

### 3. Изменение возвращаемой ошибки

```go
func ReadConfig(filename string) (cfg Config, err error) {
    defer func() {
        if err != nil {
            err = fmt.Errorf("read config %s: %w", filename, err)
        }
    }()

    file, err := os.Open(filename)
    if err != nil {
        return Config{}, err
    }
    defer file.Close()

    // Парсинг...
    return cfg, nil
}
```

### 4. Safe goroutine wrapper

```go
func SafeGo(fn func()) {
    go func() {
        defer func() {
            if r := recover(); r != nil {
                log.Printf("Goroutine panic: %v\n%s", r, debug.Stack())
            }
        }()
        fn()
    }()
}

// Использование
SafeGo(func() {
    // Код который может паниковать
})
```

## Частые ошибки

### 1. defer в цикле

```go
// ❌ Все defer выполнятся только в конце функции!
func processFiles(files []string) error {
    for _, filename := range files {
        file, err := os.Open(filename)
        if err != nil {
            return err
        }
        defer file.Close() // Закроется только в конце!

        // Обработка...
    }
    return nil
}

// ✅ Используйте отдельную функцию
func processFiles(files []string) error {
    for _, filename := range files {
        if err := processFile(filename); err != nil {
            return err
        }
    }
    return nil
}

func processFile(filename string) error {
    file, err := os.Open(filename)
    if err != nil {
        return err
    }
    defer file.Close() // Закроется в конце processFile

    // Обработка...
    return nil
}
```

### 2. Игнорирование ошибки defer

```go
// ❌ Ошибка Close игнорируется
defer file.Close()

// ✅ Обработка ошибки
defer func() {
    if err := file.Close(); err != nil {
        log.Printf("Failed to close file: %v", err)
    }
}()

// ✅ С именованным return
func process() (err error) {
    file, err := os.Open("file.txt")
    if err != nil {
        return err
    }
    defer func() {
        if cerr := file.Close(); cerr != nil && err == nil {
            err = cerr
        }
    }()

    // Обработка...
    return nil
}
```

### 3. recover вне defer

```go
// ❌ НЕ работает
func wrong() {
    if r := recover(); r != nil {
        // Всегда nil!
    }
}

// ✅ Работает
func correct() {
    defer func() {
        if r := recover(); r != nil {
            // Ловит panic
        }
    }()
}
```

## Когда использовать

### defer

✅ **Используйте для:**
- Освобождения ресурсов (Close, Unlock)
- Логирования
- Восстановления после panic (с recover)
- Изменения именованных возвращаемых значений

❌ **Избегайте:**
- В критичных по производительности участках (небольшой overhead)
- В циклах (используйте отдельную функцию)

### panic

✅ **Используйте для:**
- Критических ошибок при инициализации (MustXxx функции)
- Программных ошибок (bugs)
- Невозможных состояний

❌ **Не используйте для:**
- Обычной обработки ошибок (используйте error)
- Валидации входных данных
- Ошибок I/O

### recover

✅ **Используйте для:**
- Middleware в web серверах
- Обертки горутин
- Граничных слоев (API границ)

❌ **Не используйте для:**
- Обычного control flow
- Замены нормальной обработки ошибок

## Best Practices

1. ✅ Используйте defer для гарантированного освобождения ресурсов
2. ✅ Помните о порядке выполнения (LIFO)
3. ✅ Избегайте defer в циклах (используйте функцию)
4. ✅ panic только для программных ошибок
5. ✅ recover в middleware и границах модулей
6. ✅ recover в каждой горутине отдельно
7. ❌ Не используйте panic для обычных ошибок
8. ❌ Не игнорируйте ошибки в defer

## Связанные темы

- [[Go - Обработка ошибок]]
- [[Go - Функции и методы]]
- [[Go - Горутины (goroutines)]]
