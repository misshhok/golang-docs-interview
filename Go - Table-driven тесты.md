# Go - Table-driven тесты

Table-driven тесты — это паттерн тестирования в Go, при котором тестовые случаи описываются в виде таблицы (слайса структур). Это позволяет легко добавлять новые тестовые сценарии без дублирования кода.

## Зачем нужны Table-driven тесты

### Проблема: дублирование кода

```go
// ❌ Плохо - много повторяющегося кода
func TestAdd(t *testing.T) {
    result := Add(2, 3)
    if result != 5 {
        t.Errorf("Add(2, 3) = %d; want 5", result)
    }
}

func TestAddNegative(t *testing.T) {
    result := Add(-1, -1)
    if result != -2 {
        t.Errorf("Add(-1, -1) = %d; want -2", result)
    }
}

func TestAddZero(t *testing.T) {
    result := Add(0, 0)
    if result != 0 {
        t.Errorf("Add(0, 0) = %d; want 0", result)
    }
}
```

### Решение: таблица тестов

```go
// ✅ Хорошо - компактно и легко расширять
func TestAdd(t *testing.T) {
    tests := []struct {
        name     string
        a, b     int
        expected int
    }{
        {"положительные числа", 2, 3, 5},
        {"отрицательные числа", -1, -1, -2},
        {"нули", 0, 0, 0},
        {"смешанные", -5, 5, 0},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result := Add(tt.a, tt.b)
            if result != tt.expected {
                t.Errorf("Add(%d, %d) = %d; want %d",
                    tt.a, tt.b, result, tt.expected)
            }
        })
    }
}
```

## Структура table-driven теста

### Базовый паттерн

```go
func TestФункция(t *testing.T) {
    // 1. Определяем таблицу тестов
    tests := []struct {
        name     string  // название теста
        input    Type    // входные данные
        expected Type    // ожидаемый результат
    }{
        {"описание 1", input1, expected1},
        {"описание 2", input2, expected2},
    }

    // 2. Проходим по всем тестам
    for _, tt := range tests {
        // 3. Запускаем каждый тест как подтест
        t.Run(tt.name, func(t *testing.T) {
            // 4. Выполняем тестируемую функцию
            result := Функция(tt.input)

            // 5. Проверяем результат
            if result != tt.expected {
                t.Errorf("got %v, want %v", result, tt.expected)
            }
        })
    }
}
```

## Примеры

### Пример 1: Простая функция

```go
// Функция валидации email
func IsValidEmail(email string) bool {
    // Упрощённая проверка
    return strings.Contains(email, "@") && strings.Contains(email, ".")
}

// Table-driven тест
func TestIsValidEmail(t *testing.T) {
    tests := []struct {
        name  string
        email string
        want  bool
    }{
        {"валидный email", "user@example.com", true},
        {"без @", "userexample.com", false},
        {"без домена", "user@", false},
        {"пустая строка", "", false},
        {"только @", "@", false},
        {"несколько @", "user@@example.com", true}, // потенциальный баг!
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := IsValidEmail(tt.email)
            if got != tt.want {
                t.Errorf("IsValidEmail(%q) = %v; want %v",
                    tt.email, got, tt.want)
            }
        })
    }
}
```

### Пример 2: Функция с ошибками

```go
// Функция деления
func Divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("division by zero")
    }
    return a / b, nil
}

// Table-driven тест
func TestDivide(t *testing.T) {
    tests := []struct {
        name      string
        a, b      float64
        want      float64
        wantError bool
    }{
        {"обычное деление", 10, 2, 5, false},
        {"деление на ноль", 10, 0, 0, true},
        {"отрицательные числа", -10, 2, -5, false},
        {"дробные числа", 7, 2, 3.5, false},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := Divide(tt.a, tt.b)

            // Проверка ошибки
            if (err != nil) != tt.wantError {
                t.Fatalf("Divide(%v, %v) error = %v; wantError %v",
                    tt.a, tt.b, err, tt.wantError)
            }

            // Проверка результата (только если нет ошибки)
            if !tt.wantError && got != tt.want {
                t.Errorf("Divide(%v, %v) = %v; want %v",
                    tt.a, tt.b, got, tt.want)
            }
        })
    }
}
```

### Пример 3: Сложные структуры

```go
type User struct {
    ID    int
    Name  string
    Email string
}

func ValidateUser(u User) error {
    if u.Name == "" {
        return errors.New("name is required")
    }
    if u.Email == "" {
        return errors.New("email is required")
    }
    return nil
}

func TestValidateUser(t *testing.T) {
    tests := []struct {
        name      string
        user      User
        wantError string  // ожидаемый текст ошибки
    }{
        {
            name: "валидный пользователь",
            user: User{ID: 1, Name: "John", Email: "john@example.com"},
            wantError: "",
        },
        {
            name: "пустое имя",
            user: User{ID: 1, Name: "", Email: "john@example.com"},
            wantError: "name is required",
        },
        {
            name: "пустой email",
            user: User{ID: 1, Name: "John", Email: ""},
            wantError: "email is required",
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            err := ValidateUser(tt.user)

            if tt.wantError == "" {
                // Ожидаем отсутствие ошибки
                if err != nil {
                    t.Errorf("ValidateUser() unexpected error: %v", err)
                }
            } else {
                // Ожидаем конкретную ошибку
                if err == nil {
                    t.Errorf("ValidateUser() expected error %q, got nil", tt.wantError)
                } else if err.Error() != tt.wantError {
                    t.Errorf("ValidateUser() error = %q; want %q",
                        err.Error(), tt.wantError)
                }
            }
        })
    }
}
```

## Параллельное выполнение с t.Parallel()

Table-driven тесты можно выполнять параллельно для ускорения:

```go
func TestAdd(t *testing.T) {
    tests := []struct {
        name     string
        a, b     int
        expected int
    }{
        {"test 1", 2, 3, 5},
        {"test 2", -1, -1, -2},
        {"test 3", 0, 0, 0},
    }

    for _, tt := range tests {
        tt := tt  // ВАЖНО: захватываем переменную цикла!
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel()  // тест будет выполняться параллельно

            result := Add(tt.a, tt.b)
            if result != tt.expected {
                t.Errorf("Add(%d, %d) = %d; want %d",
                    tt.a, tt.b, result, tt.expected)
            }
        })
    }
}
```

**⚠️ Важно:** При использовании `t.Parallel()` необходимо захватывать переменную цикла `tt := tt`, иначе все горутины будут использовать последнее значение из цикла.

## Best Practices

### 1. Описательные имена тестов

```go
// ✅ Хорошо - понятно что тестируется
tests := []struct {
    name string
    // ...
}{
    {"пустая строка возвращает ошибку", ...},
    {"валидный input возвращает результат", ...},
    {"специальные символы обрабатываются корректно", ...},
}

// ❌ Плохо - непонятные названия
tests := []struct {
    name string
    // ...
}{
    {"test1", ...},
    {"case 2", ...},
    {"тест", ...},
}
```

### 2. Группировка связанных тестов

```go
func TestCalculator(t *testing.T) {
    t.Run("Addition", func(t *testing.T) {
        tests := []struct {
            name     string
            a, b     int
            expected int
        }{
            {"положительные", 2, 3, 5},
            {"отрицательные", -2, -3, -5},
        }
        // ...
    })

    t.Run("Multiplication", func(t *testing.T) {
        tests := []struct {
            name     string
            a, b     int
            expected int
        }{
            {"положительные", 2, 3, 6},
            {"с нулём", 0, 5, 0},
        }
        // ...
    })
}
```

### 3. Использование вспомогательных функций

```go
func TestProcessData(t *testing.T) {
    tests := []struct {
        name  string
        input string
        want  Result
    }{
        // тесты
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := ProcessData(tt.input)
            assertResultEqual(t, got, tt.want)  // helper функция
        })
    }
}

// Helper функция для сравнения
func assertResultEqual(t *testing.T, got, want Result) {
    t.Helper()
    if !reflect.DeepEqual(got, want) {
        t.Errorf("got %+v, want %+v", got, want)
    }
}
```

## Типичные ошибки

### Ошибка 1: Забыли t.Run

```go
// ❌ Плохо - тесты выполнятся, но не будут изолированы
for _, tt := range tests {
    result := Add(tt.a, tt.b)
    if result != tt.expected {
        t.Errorf("failed")  // непонятно какой именно тест упал
    }
}

// ✅ Правильно - каждый тест изолирован
for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        result := Add(tt.a, tt.b)
        if result != tt.expected {
            t.Errorf("Add(%d, %d) = %d; want %d",
                tt.a, tt.b, result, tt.expected)
        }
    })
}
```

### Ошибка 2: Не захватили переменную при t.Parallel()

```go
// ❌ Опасно - race condition!
for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        t.Parallel()
        result := Add(tt.a, tt.b)  // tt может измениться!
        // ...
    })
}

// ✅ Правильно
for _, tt := range tests {
    tt := tt  // захватываем переменную
    t.Run(tt.name, func(t *testing.T) {
        t.Parallel()
        result := Add(tt.a, tt.b)
        // ...
    })
}
```

## Запуск тестов

```bash
# Запустить все тесты
go test -v

# Запустить конкретный тест
go test -run TestAdd

# Запустить конкретный подтест
go test -run TestAdd/"положительные_числа"

# С покрытием
go test -cover -v
```

## Вопросы с собеседований

**Вопрос:** В чём преимущества table-driven тестов?

**Ответ:**
1. Минимизация дублирования кода
2. Легко добавлять новые тестовые случаи
3. Все тесты используют одну логику проверки
4. Улучшенная читаемость — все случаи видны в одном месте
5. Простая поддержка

**Вопрос:** Зачем нужна строка `tt := tt` перед `t.Parallel()`?

**Ответ:** Это захват переменной цикла. Без неё все горутины будут использовать одну и ту же переменную `tt`, которая будет указывать на последнее значение из цикла. Захват создаёт новую переменную для каждой итерации, предотвращая race condition.

**Вопрос:** Можно ли комбинировать table-driven тесты с обычными тестами?

**Ответ:** Да, это распространённая практика. Используйте table-driven подход для проверки множества входных данных, а отдельные тесты — для сложных сценариев, требующих специфической настройки или проверки.

## Связанные темы

- [[Go - Unit тестирование]]
- [[Go - Моки и стабы]]
- [[Test Coverage]]
- [[Go - Бенчмаркинг]]