# Go - Errors wrapping

Обертывание ошибок для сохранения контекста (Go 1.13+).

## Зачем нужно wrapping

```go
// ❌ Без wrapping - теряем оригинальную ошибку
func ReadConfig(filename string) error {
    _, err := os.Open(filename)
    if err != nil {
        return fmt.Errorf("failed to read config")
        // Потеряли информацию что именно пошло не так!
    }
    return nil
}

// ✅ С wrapping - сохраняем оригинальную ошибку
func ReadConfig(filename string) error {
    _, err := os.Open(filename)
    if err != nil {
        return fmt.Errorf("failed to read config: %w", err)
        // %w - оборачивает ошибку
    }
    return nil
}
```

## fmt.Errorf с %w

```go
func processFile(filename string) error {
    data, err := readFile(filename)
    if err != nil {
        return fmt.Errorf("process file %s: %w", filename, err)
    }
    return nil
}

// Цепочка ошибок:
// "process file config.json: open config.json: no such file or directory"
```

## errors.Is - проверка типа

```go
import "errors"

var ErrNotFound = errors.New("not found")

func Find(id int) error {
    // ...
    return fmt.Errorf("find user %d: %w", id, ErrNotFound)
}

// Проверка
err := Find(123)
if errors.Is(err, ErrNotFound) {
    // Обработка не найденной записи
}

// До Go 1.13 так не работало:
// if err == ErrNotFound { // false! (обернутая ошибка)
```

## errors.As - извлечение ошибки

```go
type ValidationError struct {
    Field string
    Issue string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("%s: %s", e.Field, e.Issue)
}

func Validate(user User) error {
    if user.Email == "" {
        return &ValidationError{
            Field: "email",
            Issue: "required",
        }
    }
    return nil
}

// Использование
err := Validate(user)
var validErr *ValidationError
if errors.As(err, &validErr) {
    fmt.Printf("Validation failed: %s\n", validErr.Field)
}
```

## errors.Unwrap

```go
// Извлечь обернутую ошибку
wrapped := fmt.Errorf("outer: %w", innerErr)
unwrapped := errors.Unwrap(wrapped)
// unwrapped == innerErr
```

## Кастомные wrapping errors

```go
type QueryError struct {
    Query string
    Err   error
}

func (e *QueryError) Error() string {
    return fmt.Sprintf("query failed: %s", e.Query)
}

func (e *QueryError) Unwrap() error {
    return e.Err // Для работы errors.Is и errors.As
}

// Использование
func Execute(query string) error {
    err := db.Query(query)
    if err != nil {
        return &QueryError{
            Query: query,
            Err:   err,
        }
    }
    return nil
}
```

## Паттерн: обогащение ошибки

```go
func (r *Repository) GetUser(id int) (*User, error) {
    user, err := r.db.QueryUser(id)
    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, fmt.Errorf("user %d: %w", id, ErrNotFound)
        }
        return nil, fmt.Errorf("get user %d: %w", id, err)
    }
    return user, nil
}
```

## Sentinel errors

```go
var (
    ErrNotFound    = errors.New("not found")
    ErrInvalidInput = errors.New("invalid input")
    ErrUnauthorized = errors.New("unauthorized")
)

func GetUser(id int) error {
    if id < 0 {
        return fmt.Errorf("get user: %w", ErrInvalidInput)
    }
    // ...
    return fmt.Errorf("get user %d: %w", id, ErrNotFound)
}

// Проверка
err := GetUser(-1)
if errors.Is(err, ErrInvalidInput) {
    // Обработка невалидного ввода
}
```

## Множественное wrapping (Go 1.20+)

```go
err1 := errors.New("error 1")
err2 := errors.New("error 2")

// Обернуть несколько ошибок
err := fmt.Errorf("multiple errors: %w, %w", err1, err2)

// Проверка любой из ошибок
errors.Is(err, err1) // true
errors.Is(err, err2) // true
```

## Best Practices

1. ✅ Используйте `%w` для wrapping ошибок
2. ✅ Добавляйте контекст при wrapping
3. ✅ Используйте `errors.Is` вместо `==`
4. ✅ Используйте `errors.As` для извлечения типа
5. ✅ Определяйте sentinel errors для публичных ошибок
6. ❌ Не оборачивайте дважды одну и ту же ошибку
7. ❌ Не используйте `%v` когда нужно `%w`

## Связанные темы

- [[Go - Обработка ошибок]]
- [[Go - Defer, Panic, Recover]]
