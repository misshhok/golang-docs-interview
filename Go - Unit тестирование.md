# Go - Unit тестирование

Unit тестирование в Go — это процесс тестирования отдельных функций и методов для проверки их корректной работы. Go предоставляет встроенный пакет `testing` для написания тестов без необходимости использования сторонних библиотек.

## Основы пакета testing

### Структура тестов

Тесты в Go должны находиться в файлах с суффиксом `_test.go`. Для каждой функции создаётся тестовая функция с префиксом `Test`:

```go
// math.go
package math

func Add(a, b int) int {
    return a + b
}

// math_test.go
package math

import "testing"

func TestAdd(t *testing.T) {
    result := Add(2, 3)
    expected := 5

    if result != expected {
        t.Errorf("Add(2, 3) = %d; expected %d", result, expected)
    }
}
```

### Naming Convention

- **Файлы:** `имя_файла_test.go`
- **Функции:** `TestИмяФункции(t *testing.T)`
- Тестовая функция должна принимать единственный параметр `*testing.T`

```go
// Правильные имена тестов
func TestCalculateTotal(t *testing.T) { }
func TestUserValidation(t *testing.T) { }
func TestParseJSON(t *testing.T) { }

// ❌ Неправильно - тест не будет запущен
func testAdd(t *testing.T) { }  // нет префикса Test
func Test_add(t *testing.T) { }  // начинается с маленькой буквы после Test
```

## t.Error vs t.Fatal

### t.Error / t.Errorf

Сообщает об ошибке, но **продолжает выполнение теста**:

```go
func TestMultipleChecks(t *testing.T) {
    if Add(1, 1) != 2 {
        t.Error("1 + 1 должно быть 2")  // тест продолжится
    }

    if Add(2, 2) != 4 {
        t.Error("2 + 2 должно быть 4")  // эта проверка тоже выполнится
    }
}
```

### t.Fatal / t.Fatalf

Сообщает об ошибке и **немедленно останавливает тест**:

```go
func TestWithFatal(t *testing.T) {
    config, err := LoadConfig()
    if err != nil {
        t.Fatalf("Не удалось загрузить конфиг: %v", err)
        // код ниже не выполнится
    }

    // используем config дальше
    result := ProcessWithConfig(config)
    // ...
}
```

**Когда использовать:**
- ✅ `t.Error` — когда можно продолжить проверку других условий
- ✅ `t.Fatal` — когда дальнейшее выполнение невозможно (например, не удалось инициализировать тестовые данные)

## Подтесты с t.Run

`t.Run` позволяет группировать связанные тесты и запускать их изолированно:

```go
func TestCalculator(t *testing.T) {
    // Тест сложения
    t.Run("Addition", func(t *testing.T) {
        if Add(2, 3) != 5 {
            t.Error("Addition failed")
        }
    })

    // Тест вычитания
    t.Run("Subtraction", func(t *testing.T) {
        if Subtract(5, 3) != 2 {
            t.Error("Subtraction failed")
        }
    })

    // Тест с edge case
    t.Run("AdditionWithNegative", func(t *testing.T) {
        if Add(-1, 1) != 0 {
            t.Error("Addition with negative failed")
        }
    })
}
```

**Преимущества t.Run:**
- Логическая группировка тестов
- Возможность запустить конкретный подтест: `go test -run TestCalculator/Addition`
- Каждый подтест изолирован
- Параллельное выполнение с `t.Parallel()`

## Флаги go test

### Основные флаги

```bash
# Вывести подробную информацию о всех тестах
go test -v

# Запустить конкретный тест
go test -run TestAdd

# Запустить тесты по паттерну (regexp)
go test -run TestCalculator/Add

# Показать покрытие кода
go test -cover

# Подробная информация о покрытии
go test -coverprofile=coverage.out
go tool cover -html=coverage.out

# Запуск с таймаутом
go test -timeout 30s

# Параллельное выполнение
go test -parallel 4

# Показать, какие тесты запускаются
go test -v -run .
```

### Практический пример

```go
package calculator

import "testing"

func TestDivide(t *testing.T) {
    t.Run("NormalDivision", func(t *testing.T) {
        result, err := Divide(10, 2)
        if err != nil {
            t.Fatalf("unexpected error: %v", err)
        }
        if result != 5 {
            t.Errorf("Divide(10, 2) = %f; expected 5", result)
        }
    })

    t.Run("DivisionByZero", func(t *testing.T) {
        _, err := Divide(10, 0)
        if err == nil {
            t.Error("expected error for division by zero")
        }
    })
}
```

Запуск:
```bash
# Запустить все тесты
go test -v

# Запустить только тест деления
go test -v -run TestDivide

# Запустить только проверку деления на ноль
go test -v -run TestDivide/DivisionByZero
```

## Типичные ошибки

### Ошибка 1: Неправильное именование

```go
// ❌ Неправильно - тест не запустится
func testAdd(t *testing.T) {
    // ...
}

// ✅ Правильно
func TestAdd(t *testing.T) {
    // ...
}
```

### Ошибка 2: Использование t.Fatal вместо t.Error

```go
// ❌ Плохо - после первой ошибки остальные проверки не выполнятся
func TestValidation(t *testing.T) {
    if !ValidateName("") {
        t.Fatal("Name validation failed")
    }
    if !ValidateEmail("") {
        t.Fatal("Email validation failed")  // может не дойти сюда
    }
}

// ✅ Лучше - все проверки выполнятся
func TestValidation(t *testing.T) {
    if !ValidateName("") {
        t.Error("Name validation failed")
    }
    if !ValidateEmail("") {
        t.Error("Email validation failed")  // выполнится в любом случае
    }
}
```

### Ошибка 3: Тесты зависят друг от друга

```go
// ❌ Плохо - глобальное состояние
var counter int

func TestIncrement(t *testing.T) {
    counter++
    if counter != 1 {
        t.Error("expected 1")
    }
}

func TestDouble(t *testing.T) {
    // зависит от порядка выполнения TestIncrement!
    counter *= 2
}

// ✅ Хорошо - тесты изолированы
func TestIncrement(t *testing.T) {
    counter := 0
    counter++
    if counter != 1 {
        t.Error("expected 1")
    }
}
```

## Best Practices

1. ✅ **Один тест = одна проверка** (или используй подтесты)
2. ✅ **Называй тесты описательно**: `TestUserCreationWithInvalidEmail`
3. ✅ **Изолируй тесты** - не используй глобальное состояние
4. ✅ **Используй t.Helper()** для вспомогательных функций
5. ✅ **Проверяй как успешные, так и ошибочные сценарии**

```go
// Helper функция
func assertEqual(t *testing.T, got, want int) {
    t.Helper()  // при ошибке покажет строку вызова, а не эту функцию
    if got != want {
        t.Errorf("got %d, want %d", got, want)
    }
}

func TestWithHelper(t *testing.T) {
    result := Add(2, 3)
    assertEqual(t, result, 5)  // ошибка укажет на эту строку
}
```

## Вопросы с собеседований

**Вопрос:** В чём разница между `t.Error` и `t.Fatal`?

**Ответ:** `t.Error` сообщает об ошибке, но продолжает выполнение теста, позволяя выполнить остальные проверки. `t.Fatal` немедленно останавливает выполнение текущего теста. Используйте `t.Fatal`, когда дальнейшие проверки не имеют смысла (например, не удалось инициализировать тестовые данные).

**Вопрос:** Как запустить только один тест из пакета?

**Ответ:** Используйте флаг `-run` с именем теста: `go test -run TestAdd`. Можно использовать регулярные выражения для более точного выбора, например `go test -run TestCalculator/Addition` запустит только подтест Addition внутри TestCalculator.

**Вопрос:** Зачем нужна функция `t.Helper()`?

**Ответ:** `t.Helper()` помечает функцию как вспомогательную. Когда происходит ошибка в такой функции, Go покажет в отчёте строку, где эта функция была вызвана, а не строку внутри самой вспомогательной функции. Это делает отладку проще.

## Связанные темы

- [[Go - Table-driven тесты]]
- [[Go - Моки и стабы]]
- [[Test Coverage]]
- [[Go - Бенчмаркинг]]
- [[E2E тестирование]]