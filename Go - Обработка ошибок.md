# Go - Обработка ошибок

В Go ошибки - это обычные значения, реализующие интерфейс `error`. Нет исключений (exceptions), только явная обработка ошибок.

## Интерфейс error

```go
type error interface {
    Error() string
}
```

## Создание ошибок

### errors.New()

```go
import "errors"

err := errors.New("something went wrong")
fmt.Println(err) // something went wrong
```

### fmt.Errorf()

```go
err := fmt.Errorf("failed to open file: %s", filename)

// С форматированием
err := fmt.Errorf("error at line %d: %s", lineNum, msg)
```

### Кастомные ошибки

```go
type ValidationError struct {
    Field string
    Value interface{}
    Msg   string
}

func (e ValidationError) Error() string {
    return fmt.Sprintf("validation error: %s=%v: %s",
        e.Field, e.Value, e.Msg)
}

// Использование
err := ValidationError{
    Field: "email",
    Value: "invalid@",
    Msg:   "invalid email format",
}
```

## Возврат ошибок

### Базовый паттерн

```go
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("division by zero")
    }
    return a / b, nil
}

result, err := divide(10, 2)
if err != nil {
    log.Fatal(err)
}
fmt.Println(result)
```

### Множественный return

```go
// Ошибка обычно последний return value
func readConfig(filename string) (*Config, error)
func parseJSON(data []byte) (map[string]interface{}, error)
```

### Именованный return

```go
func process() (result int, err error) {
    if someCondition {
        err = errors.New("condition failed")
        return // Возвращает result=0, err
    }

    result = 42
    return // Возвращает result=42, err=nil
}
```

## Обработка ошибок

### if err != nil

Стандартный паттерн:

```go
file, err := os.Open("file.txt")
if err != nil {
    return err // Пробрасываем выше
}
defer file.Close()

// Работа с file
```

### Early return

```go
func processUser(id int) error {
    user, err := getUser(id)
    if err != nil {
        return err
    }

    if err := validateUser(user); err != nil {
        return err
    }

    if err := saveUser(user); err != nil {
        return err
    }

    return nil
}
```

### Игнорирование ошибок (редко!)

```go
// Используйте _ только если точно знаете что ошибки не будет
_ = file.Close() // В defer, где уже есть основная ошибка
```

## Error Wrapping (Go 1.13+)

### fmt.Errorf с %w

```go
func readConfig(filename string) error {
    data, err := os.ReadFile(filename)
    if err != nil {
        // Оборачиваем ошибку с контекстом
        return fmt.Errorf("read config %s: %w", filename, err)
    }
    // ...
}

// Результат:
// read config app.yaml: open app.yaml: no such file or directory
```

### errors.Is()

Проверка конкретной ошибки в цепочке:

```go
_, err := os.Open("file.txt")
if errors.Is(err, os.ErrNotExist) {
    fmt.Println("File does not exist")
}

// Работает даже если ошибка обернута!
err = fmt.Errorf("failed: %w", os.ErrNotExist)
fmt.Println(errors.Is(err, os.ErrNotExist)) // true
```

### errors.As()

Извлечение конкретного типа ошибки:

```go
type ValidationError struct {
    Field string
}

func (e ValidationError) Error() string {
    return fmt.Sprintf("validation failed: %s", e.Field)
}

// Использование
err := process()
var valErr ValidationError
if errors.As(err, &valErr) {
    fmt.Printf("Validation error in field: %s\n", valErr.Field)
}
```

### errors.Unwrap()

Получить обернутую ошибку:

```go
err := fmt.Errorf("outer: %w", errors.New("inner"))

unwrapped := errors.Unwrap(err)
fmt.Println(unwrapped) // inner
```

## Sentinel Errors

Предопределенные ошибки для сравнения:

```go
var (
    ErrNotFound    = errors.New("not found")
    ErrUnauthorized = errors.New("unauthorized")
    ErrBadRequest  = errors.New("bad request")
)

func getUser(id int) (*User, error) {
    if id < 0 {
        return nil, ErrBadRequest
    }

    user := findUser(id)
    if user == nil {
        return nil, ErrNotFound
    }

    return user, nil
}

// Проверка
user, err := getUser(123)
if errors.Is(err, ErrNotFound) {
    // Обработка not found
}
```

**Стандартные sentinel errors:**
- `io.EOF` - конец файла
- `os.ErrNotExist` - файл не существует
- `os.ErrExist` - файл уже существует
- `context.Canceled` - context отменен
- `context.DeadlineExceeded` - deadline превышен

## Паттерны обработки

### 1. Проброс с контекстом

```go
func processRequest(req Request) error {
    data, err := fetchData(req.ID)
    if err != nil {
        return fmt.Errorf("process request %d: %w", req.ID, err)
    }

    if err := validate(data); err != nil {
        return fmt.Errorf("validate data for request %d: %w", req.ID, err)
    }

    return nil
}
```

### 2. Возврат nil для успеха

```go
// ✅ Хорошо - nil означает успех
func doSomething() error {
    if err := step1(); err != nil {
        return err
    }

    if err := step2(); err != nil {
        return err
    }

    return nil // Успех
}

// ❌ Плохо - bool для успеха
func doSomething() (bool, error) {
    // Избыточно
}
```

### 3. Defer для cleanup

```go
func processFile(filename string) error {
    f, err := os.Open(filename)
    if err != nil {
        return err
    }
    defer f.Close() // Гарантирует закрытие

    // Обработка файла
    return process(f)
}
```

### 4. Defer с named return для error handling

```go
func doWork() (err error) {
    resource, err := acquire()
    if err != nil {
        return err
    }

    defer func() {
        if releaseErr := release(resource); releaseErr != nil {
            if err == nil {
                err = releaseErr // Возвращаем release error если нет других
            }
        }
    }()

    return work(resource)
}
```

### 5. Multi-error

```go
type MultiError []error

func (m MultiError) Error() string {
    if len(m) == 0 {
        return "no errors"
    }
    if len(m) == 1 {
        return m[0].Error()
    }

    var b strings.Builder
    b.WriteString(fmt.Sprintf("%d errors:", len(m)))
    for i, err := range m {
        b.WriteString(fmt.Sprintf("\n  [%d] %v", i+1, err))
    }
    return b.String()
}

func validateAll(users []User) error {
    var errs MultiError

    for i, user := range users {
        if err := validate(user); err != nil {
            errs = append(errs, fmt.Errorf("user %d: %w", i, err))
        }
    }

    if len(errs) > 0 {
        return errs
    }
    return nil
}
```

## Panic и Recover

**Panic** - для невосстановимых ошибок:

```go
// ❌ Не используйте panic для обычных ошибок!
if err != nil {
    panic(err) // Плохо
}

// ✅ panic только для programmer errors
func getConfig() *Config {
    cfg, err := loadConfig()
    if err != nil {
        panic("config must be valid") // Cannot continue
    }
    return cfg
}
```

**Recover** - перехват panic:

```go
func safeDivide(a, b int) (result int, err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("panic recovered: %v", r)
        }
    }()

    return a / b, nil // Может вызвать panic при b=0
}
```

Подробнее: [[Go - Defer, Panic, Recover]]

## Best Practices

### 1. Не игнорируйте ошибки

```go
// ❌ Плохо
file, _ := os.Open("file.txt")

// ✅ Хорошо
file, err := os.Open("file.txt")
if err != nil {
    return err
}
```

### 2. Добавляйте контекст при оборачивании

```go
// ❌ Теряем контекст
return err

// ✅ Добавляем контекст
return fmt.Errorf("failed to process user %d: %w", userID, err)
```

### 3. Не оборачивайте дважды

```go
// ❌ Двойное оборачивание
if err != nil {
    return fmt.Errorf("error: %w", err)
}
// ...где-то выше...
if err != nil {
    return fmt.Errorf("outer error: %w", err)
}

// Результат: outer error: error: original error (избыточно)
```

### 4. Проверяйте sentinel errors через errors.Is

```go
// ❌ Не работает с wrapped errors
if err == os.ErrNotExist {
    // ...
}

// ✅ Работает с wrapped errors
if errors.Is(err, os.ErrNotExist) {
    // ...
}
```

### 5. Используйте errors.As для type assertions

```go
// ❌ Не работает с wrapped errors
if valErr, ok := err.(ValidationError); ok {
    // ...
}

// ✅ Работает с wrapped errors
var valErr ValidationError
if errors.As(err, &valErr) {
    // ...
}
```

### 6. Документируйте возвращаемые ошибки

```go
// GetUser returns ErrNotFound if the user doesn't exist,
// ErrUnauthorized if the caller lacks permissions,
// or a wrapped database error.
func GetUser(ctx context.Context, id int) (*User, error) {
    // ...
}
```

## Error Groups

Для конкурентных операций используйте `errgroup`:

```go
import "golang.org/x/sync/errgroup"

func fetchAll(urls []string) error {
    g := new(errgroup.Group)

    for _, url := range urls {
        url := url // Захват переменной
        g.Go(func() error {
            return fetch(url)
        })
    }

    // Ждет всех и возвращает первую ошибку
    return g.Wait()
}

// С context для отмены
func fetchAll(ctx context.Context, urls []string) error {
    g, ctx := errgroup.WithContext(ctx)

    for _, url := range urls {
        url := url
        g.Go(func() error {
            return fetchWithContext(ctx, url)
        })
    }

    return g.Wait()
}
```

## Частые ошибки

### 1. Сравнение error через ==

```go
// ❌ Не работает с wrapped errors
if err == ErrNotFound {
    // ...
}

// ✅ Правильно
if errors.Is(err, ErrNotFound) {
    // ...
}
```

### 2. Возврат error без контекста

```go
// ❌ Теряем информацию
func processUser(id int) error {
    user, err := db.GetUser(id)
    return err // Не знаем для какого ID!
}

// ✅ Добавляем контекст
func processUser(id int) error {
    user, err := db.GetUser(id)
    if err != nil {
        return fmt.Errorf("get user %d: %w", id, err)
    }
    // ...
}
```

### 3. Логирование и возврат ошибки

```go
// ❌ Логируем и возвращаем - дублирование!
if err != nil {
    log.Printf("error: %v", err)
    return err // Вызывающий тоже залогирует!
}

// ✅ Либо логируем, либо возвращаем
if err != nil {
    return fmt.Errorf("context: %w", err) // Только возврат
}

// ✅ Логируем только на верхнем уровне
if err := process(); err != nil {
    log.Printf("process failed: %v", err)
    // Обрабатываем
}
```

### 4. Пустая ошибка

```go
// ❌ Неинформативно
return errors.New("error")

// ✅ Описательно
return errors.New("failed to connect to database: timeout exceeded")
```

## Связанные темы

- [[Go - Интерфейсы]]
- [[Go - Defer, Panic, Recover]]
- [[Go - Context]]
