# Задача - Valid Parentheses

Проверить правильность расстановки скобок в строке.

## Условие

Дана строка `s`, содержащая только символы `'('`, `')'`, `'{'`, `'}'`, `'['` и `']'`. Определить, является ли строка правильной.

**Правильная строка должна удовлетворять условиям:**
1. Открывающие скобки должны закрываться скобками того же типа
2. Открывающие скобки должны закрываться в правильном порядке
3. Каждой закрывающей скобке должна соответствовать открывающая того же типа

**Пример 1:**
```
Вход: s = "()"
Выход: true
```

**Пример 2:**
```
Вход: s = "()[]{}"
Выход: true
```

**Пример 3:**
```
Вход: s = "(]"
Выход: false
```

**Пример 4:**
```
Вход: s = "([)]"
Выход: false
Объяснение: скобки переплетены неправильно
```

**Пример 5:**
```
Вход: s = "{[]}"
Выход: true
```

## Анализ задачи

**Ключевые наблюдения:**
- Последняя открытая скобка должна закрыться первой (LIFO) → Стек!
- При встрече открывающей скобки - добавляем в стек
- При встрече закрывающей - проверяем соответствие с вершиной стека
- В конце стек должен быть пуст

**Edge cases:**
- Пустая строка → true
- Только открывающие скобки → false
- Только закрывающие скобки → false
- Нечётная длина → false

## Решение: Стек

**Идея:** Использовать стек для отслеживания открывающих скобок.

**Алгоритм:**
1. Создаем пустой стек
2. Проходим по каждому символу:
   - Если открывающая скобка → push в стек
   - Если закрывающая скобка:
     - Если стек пуст → false
     - Pop из стека и проверяем соответствие типов
     - Если не соответствуют → false
3. После обработки всех символов стек должен быть пуст

**Сложность:** O(n) по времени, O(n) по памяти

```go
func isValid(s string) bool {
    // Edge case: нечётная длина не может быть валидной
    if len(s) % 2 != 0 {
        return false
    }

    // Стек для хранения открывающих скобок
    stack := []rune{}

    // Карта соответствия: закрывающая → открывающая
    pairs := map[rune]rune{
        ')': '(',
        '}': '{',
        ']': '[',
    }

    for _, char := range s {
        // Проверяем, является ли символ закрывающей скобкой
        if opening, isClosing := pairs[char]; isClosing {
            // Это закрывающая скобка
            if len(stack) == 0 {
                return false  // Нет соответствующей открывающей
            }

            // Проверяем, совпадает ли с вершиной стека
            if stack[len(stack)-1] != opening {
                return false  // Неправильный тип скобки
            }

            // Pop из стека
            stack = stack[:len(stack)-1]
        } else {
            // Это открывающая скобка - push в стек
            stack = append(stack, char)
        }
    }

    // Стек должен быть пуст
    return len(stack) == 0
}
```

## Пошаговое объяснение

Рассмотрим пример: `s = "{[]}"`

### Шаг 1: char = '{'
```
Открывающая скобка
stack = ['{']
```

### Шаг 2: char = '['
```
Открывающая скобка
stack = ['{', '[']
```

### Шаг 3: char = ']'
```
Закрывающая скобка
Вершина стека: '[' ✅ Соответствует
Pop из стека
stack = ['{']
```

### Шаг 4: char = '}'
```
Закрывающая скобка
Вершина стека: '{' ✅ Соответствует
Pop из стека
stack = []
```

### Результат
```
Стек пуст ✅
Возвращаем true
```

## Визуализация

### Пример 1: `"()[]{}"`
```
Символ | Действие       | Стек
-------|----------------|-------
  (    | Push           | [(]
  )    | Pop (match)    | []
  [    | Push           | [[]
  ]    | Pop (match)    | []
  {    | Push           | [{]
  }    | Pop (match)    | []
       | Стек пуст ✅   |
```

### Пример 2: `"([)]"` (неправильно)
```
Символ | Действие       | Стек
-------|----------------|-------
  (    | Push           | [(]
  [    | Push           | [(, []
  )    | Pop: [ != ( ❌ | FAIL
```

### Пример 3: `"{[}]"` (неправильно)
```
Символ | Действие       | Стек
-------|----------------|-------
  {    | Push           | [{]
  [    | Push           | [{, []
  }    | Pop: [ != { ❌ | FAIL
```

## Альтернативная реализация (более компактная)

```go
func isValid(s string) bool {
    stack := []rune{}

    for _, c := range s {
        switch c {
        case '(', '[', '{':
            // Открывающие - в стек
            stack = append(stack, c)
        case ')':
            if len(stack) == 0 || stack[len(stack)-1] != '(' {
                return false
            }
            stack = stack[:len(stack)-1]
        case ']':
            if len(stack) == 0 || stack[len(stack)-1] != '[' {
                return false
            }
            stack = stack[:len(stack)-1]
        case '}':
            if len(stack) == 0 || stack[len(stack)-1] != '{' {
                return false
            }
            stack = stack[:len(stack)-1]
        }
    }

    return len(stack) == 0
}
```

## Тестовые случаи

```go
func TestIsValid(t *testing.T) {
    tests := []struct {
        input    string
        expected bool
    }{
        {"()", true},
        {"()[]{}", true},
        {"{[]}", true},
        {"(]", false},
        {"([)]", false},
        {"((", false},
        {"))", false},
        {"", true},  // Пустая строка - валидна
        {"{[()]}", true},
        {"((()))", true},
        {"(((", false},
        {"}}}", false},
        {"({[}])", false},
    }

    for _, tt := range tests {
        result := isValid(tt.input)
        if result != tt.expected {
            t.Errorf("isValid(%q) = %v, expected %v",
                tt.input, result, tt.expected)
        }
    }
}
```

## Оптимизация с использованием массива фиксированного размера

Если известно, что входная строка не очень длинная, можно использовать массив вместо слайса:

```go
func isValidOptimized(s string) bool {
    if len(s) % 2 != 0 {
        return false
    }

    stack := make([]rune, 0, len(s)/2)
    pairs := map[rune]rune{')': '(', '}': '{', ']': '['}

    for _, char := range s {
        if opening, isClosing := pairs[char]; isClosing {
            if len(stack) == 0 || stack[len(stack)-1] != opening {
                return false
            }
            stack = stack[:len(stack)-1]
        } else {
            stack = append(stack, char)
        }
    }

    return len(stack) == 0
}
```

**Преимущества:**
- Предаллоцированная память
- Меньше аллокаций

## Вариации задачи

### 1. Удалить минимальное количество скобок для валидности

```go
func minRemoveToMakeValid(s string) string {
    // Первый проход: помечаем невалидные ')'
    stack := []int{}
    toRemove := make(map[int]bool)

    for i, c := range s {
        if c == '(' {
            stack = append(stack, i)
        } else if c == ')' {
            if len(stack) == 0 {
                toRemove[i] = true  // Невалидная ')'
            } else {
                stack = stack[:len(stack)-1]
            }
        }
    }

    // Оставшиеся '(' тоже невалидные
    for _, idx := range stack {
        toRemove[idx] = true
    }

    // Строим результат
    result := []rune{}
    for i, c := range s {
        if !toRemove[i] {
            result = append(result, c)
        }
    }

    return string(result)
}
```

### 2. Проверка со специальными символами

Что если в строке есть другие символы, не только скобки?

```go
func isValidWithOtherChars(s string) bool {
    stack := []rune{}
    pairs := map[rune]rune{')': '(', '}': '{', ']': '['}
    openings := map[rune]bool{'(': true, '[': true, '{': true}

    for _, char := range s {
        if openings[char] {
            // Открывающая скобка
            stack = append(stack, char)
        } else if opening, isClosing := pairs[char]; isClosing {
            // Закрывающая скобка
            if len(stack) == 0 || stack[len(stack)-1] != opening {
                return false
            }
            stack = stack[:len(stack)-1]
        }
        // Другие символы игнорируем
    }

    return len(stack) == 0
}
```

### 3. Вернуть индексы несовпадающих скобок

```go
func findInvalidBrackets(s string) []int {
    stack := []int{}  // Храним индексы
    invalid := []int{}

    for i, char := range s {
        if char == '(' || char == '[' || char == '{' {
            stack = append(stack, i)
        } else {
            if len(stack) == 0 {
                invalid = append(invalid, i)
            } else {
                stack = stack[:len(stack)-1]
            }
        }
    }

    // Добавляем оставшиеся открывающие
    invalid = append(invalid, stack...)

    return invalid
}
```

## Где спрашивают

- **LeetCode:** #20 - Valid Parentheses
- **Яндекс:** Базовая задача на собеседованиях
- **Авито:** Часто в первом раунде
- **Т-Банк:** Могут усложнить (спросить про вариации)
- **Озон:** Классическая задача на стек

## Вопросы с собеседований

**Вопрос 1:** Почему используется именно стек?

**Ответ:** Скобки подчиняются принципу LIFO (Last In, First Out) - последняя открытая скобка должна закрыться первой. Это идеально соответствует структуре данных "стек".

---

**Вопрос 2:** Какая сложность по памяти в худшем случае?

**Ответ:** O(n) - когда все скобки открывающие, например `"((((("`. Все они попадут в стек.

---

**Вопрос 3:** Можно ли решить без стека?

**Ответ:** Для простой проверки баланса одного типа скобок можно использовать счетчик. Но для нескольких типов скобок, которые могут переплетаться, нужен стек для отслеживания порядка.

---

**Вопрос 4:** Что если скобок очень много типов?

**Ответ:** Алгоритм масштабируется - просто добавляем новые пары в map. Сложность остается O(n).

## Best Practices

1. ✅ Используйте стек для решения задач со скобками
2. ✅ Проверяйте нечетную длину сразу (оптимизация)
3. ✅ Используйте map для соответствия типов скобок
4. ✅ Предаллоцируйте память для стека если знаете верхнюю границу
5. ❌ Не забывайте проверить, что стек пуст в конце
6. ❌ Не забывайте проверить пустоту стека перед pop

## Применение в реальности

### Валидация синтаксиса

```go
func validateCodeSyntax(code string) (bool, error) {
    stack := []rune{}
    pairs := map[rune]rune{')': '(', '}': '{', ']': '['}
    line := 1

    for i, char := range code {
        if char == '\n' {
            line++
        }

        if char == '(' || char == '[' || char == '{' {
            stack = append(stack, char)
        } else if opening, isClosing := pairs[char]; isClosing {
            if len(stack) == 0 {
                return false, fmt.Errorf("unexpected '%c' at position %d (line %d)",
                    char, i, line)
            }
            if stack[len(stack)-1] != opening {
                return false, fmt.Errorf("mismatched brackets at position %d (line %d)",
                    i, line)
            }
            stack = stack[:len(stack)-1]
        }
    }

    if len(stack) > 0 {
        return false, fmt.Errorf("unclosed brackets: %d remaining", len(stack))
    }

    return true, nil
}
```

### Балансировка HTML тегов

```go
func validateHTMLTags(html string) bool {
    stack := []string{}
    i := 0

    for i < len(html) {
        if html[i] == '<' {
            // Найти закрывающую >
            j := i
            for j < len(html) && html[j] != '>' {
                j++
            }

            tag := html[i+1:j]

            if strings.HasPrefix(tag, "/") {
                // Закрывающий тег
                tagName := tag[1:]
                if len(stack) == 0 || stack[len(stack)-1] != tagName {
                    return false
                }
                stack = stack[:len(stack)-1]
            } else {
                // Открывающий тег (игнорируем самозакрывающиеся)
                if !strings.HasSuffix(tag, "/") {
                    stack = append(stack, tag)
                }
            }

            i = j + 1
        } else {
            i++
        }
    }

    return len(stack) == 0
}
```

## Связанные темы

- [[Стек и очередь]]
- [[Алгоритмическая сложность (Big O)]]
- [[Go - Массивы и слайсы]]
- [[Go - Строки и UTF-8]]
- [[Go - Карты (maps)]]
