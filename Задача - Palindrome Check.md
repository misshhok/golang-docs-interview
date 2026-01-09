# Задача - Palindrome Check

Проверка, является ли строка палиндромом.

## Условие

Дана строка `s`. Определить, является ли она палиндромом, учитывая только буквенно-цифровые символы и игнорируя регистр.

**Палиндром** - строка, которая читается одинаково слева направо и справа налево.

**Пример 1:**
```
Вход: s = "A man, a plan, a canal: Panama"
Выход: true
Объяснение: "amanaplanacanalpanama" - палиндром
```

**Пример 2:**
```
Вход: s = "race a car"
Выход: false
Объяснение: "raceacar" - не палиндром
```

**Пример 3:**
```
Вход: s = " "
Выход: true
Объяснение: пустая строка (без буквенно-цифровых символов) считается палиндромом
```

**Пример 4:**
```
Вход: s = "0P"
Выход: false
```

## Анализ задачи

**Ключевые наблюдения:**
- Игнорируем не буквенно-цифровые символы (пробелы, знаки препинания)
- Игнорируем регистр (A = a)
- Используем два указателя с концов строки

**Подходы:**
1. **Очистка + Сравнение** - создать новую строку без лишних символов, сравнить с перевернутой
2. **Два указателя** - проверка in-place без доп. памяти ✅

## Решение 1: Очистка строки

**Идея:** Создать новую строку только с буквенно-цифровыми символами в нижнем регистре, сравнить с перевернутой.

**Сложность:** O(n) по времени, O(n) по памяти

```go
func isPalindromeClean(s string) bool {
    // Очищаем строку: только буквы и цифры, lowercase
    cleaned := ""
    for _, ch := range s {
        if isAlphanumeric(ch) {
            cleaned += string(unicode.ToLower(ch))
        }
    }

    // Сравниваем с перевернутой
    return cleaned == reverse(cleaned)
}

func isAlphanumeric(ch rune) bool {
    return unicode.IsLetter(ch) || unicode.IsDigit(ch)
}

func reverse(s string) string {
    runes := []rune(s)
    for i, j := 0, len(runes)-1; i < j; i, j = i+1, j-1 {
        runes[i], runes[j] = runes[j], runes[i]
    }
    return string(runes)
}
```

**Недостатки:**
- ❌ Дополнительная память O(n)
- ❌ Две итерации по строке

## Решение 2: Два указателя (Оптимальное)

**Идея:** Два указателя с начала и конца, пропускаем не буквенно-цифровые символы.

**Сложность:** O(n) по времени, O(1) по памяти ✅

```go
func isPalindrome(s string) bool {
    left, right := 0, len(s)-1

    for left < right {
        // Пропускаем не-алфавитно-цифровые символы слева
        for left < right && !isAlphanumeric(rune(s[left])) {
            left++
        }

        // Пропускаем не-алфавитно-цифровые символы справа
        for left < right && !isAlphanumeric(rune(s[right])) {
            right--
        }

        // Сравниваем символы (игнорируя регистр)
        if unicode.ToLower(rune(s[left])) != unicode.ToLower(rune(s[right])) {
            return false
        }

        left++
        right--
    }

    return true
}

func isAlphanumeric(ch rune) bool {
    return unicode.IsLetter(ch) || unicode.IsDigit(ch)
}
```

**Преимущества:**
- ✅ Один проход по строке
- ✅ Константная память
- ✅ Ранний выход при несовпадении

## Пошаговое объяснение

Рассмотрим пример: `s = "A man, a plan, a canal: Panama"`

### Визуализация указателей

```
Строка: "A man, a plan, a canal: Panama"
         ↑                            ↑
        left                        right

Шаг 1:
left = 'A' (буква ✓), right = 'a' (буква ✓)
toLowerCase: 'a' == 'a' ✅
left++, right--

Шаг 2:
left = ' ' (не буква), right = 'm' (буква)
Пропускаем пробелы слева до 'm'
left = 'm', right = 'm'
toLowerCase: 'm' == 'm' ✅
left++, right--

...продолжаем до середины...

Результат: true ✅
```

### Подробный пример

```
s = "race a car"
     0123456789

left=0, right=9
'r' vs 'r' ✅ → left=1, right=8

left=1, right=8
'a' vs 'a' ✅ → left=2, right=7

left=2, right=7
'c' (пропускаем ' ' и 'a' справа)
right теперь на 'c'
'c' vs 'c' ✅ → left=3, right=6

left=3, right=6
'e' (пропускаем ' ' слева)
left теперь на 'a'
'a' vs ' ' (пропускаем)
right теперь на 'r'
'a' vs 'r' ❌

Результат: false
```

## Решение 3: Рекурсивный подход

**Идея:** Рекурсивная проверка с концов.

**Сложность:** O(n) по времени, O(n) по памяти (стек)

```go
func isPalindromeRecursive(s string) bool {
    return isPalindromeHelper(s, 0, len(s)-1)
}

func isPalindromeHelper(s string, left, right int) bool {
    // Базовый случай
    if left >= right {
        return true
    }

    // Пропускаем не-алфавитно-цифровые
    if !isAlphanumeric(rune(s[left])) {
        return isPalindromeHelper(s, left+1, right)
    }

    if !isAlphanumeric(rune(s[right])) {
        return isPalindromeHelper(s, left, right-1)
    }

    // Сравниваем
    if unicode.ToLower(rune(s[left])) != unicode.ToLower(rune(s[right])) {
        return false
    }

    // Рекурсивный вызов
    return isPalindromeHelper(s, left+1, right-1)
}
```

**Недостатки:**
- ❌ Дополнительная память для стека O(n)

## Тестовые случаи

```go
func TestIsPalindrome(t *testing.T) {
    tests := []struct {
        input    string
        expected bool
    }{
        {
            input:    "A man, a plan, a canal: Panama",
            expected: true,
        },
        {
            input:    "race a car",
            expected: false,
        },
        {
            input:    " ",
            expected: true,
        },
        {
            input:    "0P",
            expected: false,
        },
        {
            input:    "abba",
            expected: true,
        },
        {
            input:    "aba",
            expected: true,
        },
        {
            input:    "abc",
            expected: false,
        },
        {
            input:    "",
            expected: true,
        },
        {
            input:    ".,",
            expected: true,  // Только спецсимволы
        },
        {
            input:    "a.",
            expected: true,
        },
    }

    for _, tt := range tests {
        result := isPalindrome(tt.input)
        if result != tt.expected {
            t.Errorf("isPalindrome(%q) = %v, expected %v",
                tt.input, result, tt.expected)
        }
    }
}
```

## Полное решение

```go
package main

import (
    "fmt"
    "unicode"
)

func isPalindrome(s string) bool {
    left, right := 0, len(s)-1

    for left < right {
        // Пропускаем не-буквенно-цифровые слева
        for left < right && !isAlphanumeric(rune(s[left])) {
            left++
        }

        // Пропускаем не-буквенно-цифровые справа
        for left < right && !isAlphanumeric(rune(s[right])) {
            right--
        }

        // Сравниваем символы
        if unicode.ToLower(rune(s[left])) != unicode.ToLower(rune(s[right])) {
            return false
        }

        left++
        right--
    }

    return true
}

func isAlphanumeric(ch rune) bool {
    return unicode.IsLetter(ch) || unicode.IsDigit(ch)
}

func main() {
    testCases := []string{
        "A man, a plan, a canal: Panama",
        "race a car",
        " ",
        "abba",
        "0P",
    }

    for _, s := range testCases {
        result := isPalindrome(s)
        fmt.Printf("\"%s\" → %v\n", s, result)
    }
}
```

## Вариации задачи

### 1. Палиндром без игнорирования символов

Проверка всех символов, включая пробелы и знаки:

```go
func isStrictPalindrome(s string) bool {
    runes := []rune(s)
    left, right := 0, len(runes)-1

    for left < right {
        if runes[left] != runes[right] {
            return false
        }
        left++
        right--
    }

    return true
}
```

### 2. Найти самый длинный палиндром в строке

```go
func longestPalindrome(s string) string {
    if len(s) < 1 {
        return ""
    }

    start, end := 0, 0

    for i := 0; i < len(s); i++ {
        // Нечетная длина (центр - один символ)
        len1 := expandAroundCenter(s, i, i)

        // Четная длина (центр - два символа)
        len2 := expandAroundCenter(s, i, i+1)

        maxLen := max(len1, len2)

        if maxLen > end-start {
            start = i - (maxLen-1)/2
            end = i + maxLen/2
        }
    }

    return s[start : end+1]
}

func expandAroundCenter(s string, left, right int) int {
    for left >= 0 && right < len(s) && s[left] == s[right] {
        left--
        right++
    }

    return right - left - 1
}
```

### 3. Проверка палиндрома связного списка

```go
type ListNode struct {
    Val  int
    Next *ListNode
}

func isPalindromeList(head *ListNode) bool {
    // Найти середину
    slow, fast := head, head
    for fast != nil && fast.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
    }

    // Перевернуть вторую половину
    var prev *ListNode
    for slow != nil {
        next := slow.Next
        slow.Next = prev
        prev = slow
        slow = next
    }

    // Сравнить
    left, right := head, prev
    for right != nil {
        if left.Val != right.Val {
            return false
        }
        left = left.Next
        right = right.Next
    }

    return true
}
```

### 4. Сделать палиндром за минимальное количество удалений

```go
func minDeletionsToMakePalindrome(s string) int {
    n := len(s)

    // dp[i][j] = минимальное количество удалений для s[i:j+1]
    dp := make([][]int, n)
    for i := range dp {
        dp[i] = make([]int, n)
    }

    for length := 2; length <= n; length++ {
        for i := 0; i <= n-length; i++ {
            j := i + length - 1

            if s[i] == s[j] {
                dp[i][j] = dp[i+1][j-1]
            } else {
                dp[i][j] = 1 + min(dp[i+1][j], dp[i][j-1])
            }
        }
    }

    return dp[0][n-1]
}

func min(a, b int) int {
    if a < b {
        return a
    }
    return b
}
```

## Где спрашивают

- **LeetCode:** #125 - Valid Palindrome
- **LeetCode:** #5 - Longest Palindromic Substring
- **LeetCode:** #234 - Palindrome Linked List
- **Яндекс:** Базовая задача, часто в первом раунде
- **Авито:** Могут усложнить (найти самый длинный)
- **Т-Банк:** Просят оптимизацию по памяти
- **Озон:** Классическая задача на строки

## Вопросы с собеседований

**Вопрос 1:** Почему используем два указателя?

**Ответ:** Два указателя позволяют проверить симметрию строки за один проход O(n) без дополнительной памяти O(1). Мы одновременно двигаемся с обоих концов к центру.

---

**Вопрос 2:** Как обрабатывать Unicode символы?

**Ответ:** Конвертируем строку в `[]rune` вместо работы с байтами. В Go строки - это UTF-8, `rune` представляет Unicode code point.

```go
runes := []rune(s)
left, right := 0, len(runes)-1
```

---

**Вопрос 3:** Можно ли проверить палиндром за O(n/2)?

**Ответ:** O(n/2) = O(n), поэтому да - мы итерируемся до середины, но асимптотическая сложность остается O(n).

---

**Вопрос 4:** Как проверить, можно ли сделать строку палиндромом удалив один символ?

**Ответ:** При первом несовпадении пробуем удалить либо левый, либо правый символ и проверяем оставшуюся часть:

```go
func validPalindrome(s string) bool {
    left, right := 0, len(s)-1

    for left < right {
        if s[left] != s[right] {
            // Попробовать удалить левый или правый
            return isPalindromeRange(s, left+1, right) ||
                   isPalindromeRange(s, left, right-1)
        }
        left++
        right--
    }

    return true
}
```

## Best Practices

1. ✅ Используйте два указателя для O(1) памяти
2. ✅ Работайте с rune для Unicode символов
3. ✅ Используйте `unicode.ToLower` для игнорирования регистра
4. ✅ Пропускайте не-буквенно-цифровые в цикле
5. ❌ Не создавайте промежуточные строки если не нужно
6. ❌ Не забывайте про пустую строку (она палиндром)

## Применение в реальности

### Валидация данных

```go
func isValidCode(code string) bool {
    // Некоторые коды должны быть палиндромами
    cleaned := strings.ToLower(strings.ReplaceAll(code, "-", ""))
    return isPalindrome(cleaned)
}
```

### Проверка симметрии ДНК последовательности

```go
func isDNAPalindrome(sequence string) bool {
    // ДНК: A-T, C-G комплементарны
    complement := map[rune]rune{'A': 'T', 'T': 'A', 'C': 'G', 'G': 'C'}

    left, right := 0, len(sequence)-1

    for left < right {
        if complement[rune(sequence[left])] != rune(sequence[right]) {
            return false
        }
        left++
        right--
    }

    return true
}
```

### Поиск симметричных паттернов

```go
func findPalindromes(text string) []string {
    words := strings.Fields(text)
    palindromes := []string{}

    for _, word := range words {
        cleaned := cleanWord(word)
        if isPalindrome(cleaned) && len(cleaned) > 1 {
            palindromes = append(palindromes, word)
        }
    }

    return palindromes
}
```

## Связанные темы

- [[Два указателя (Two Pointers)]]
- [[Go - Строки и UTF-8]]
- [[Алгоритмы на строках]]
- [[Алгоритмическая сложность (Big O)]]
