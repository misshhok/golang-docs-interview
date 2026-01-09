# Задача - Group Anagrams

Сгруппировать слова-анаграммы.

## Условие

Дан массив строк `strs`. Сгруппируйте анаграммы вместе. Результат можно вернуть в любом порядке.

**Анаграмма** - это слово или фраза, образованная путём перестановки букв другого слова или фразы, обычно используя все оригинальные буквы ровно один раз.

**Пример 1:**
```
Вход: strs = ["eat","tea","tan","ate","nat","bat"]
Выход: [["bat"],["nat","tan"],["ate","eat","tea"]]
```

**Пример 2:**
```
Вход: strs = [""]
Выход: [[""]]
```

**Пример 3:**
```
Вход: strs = ["a"]
Выход: [["a"]]
```

## Анализ задачи

**Ключевые наблюдения:**
- Анаграммы содержат одинаковые буквы в разном порядке
- Для группировки нужен уникальный ключ, одинаковый для всех анаграмм
- Варианты ключа:
  1. Отсортированная строка: "eat" → "aet"
  2. Подсчет символов: "eat" → {e:1, a:1, t:1}

**Edge cases:**
- Пустая строка
- Одна буква
- Все слова - анаграммы друг друга
- Нет анаграмм (все слова уникальны)

## Решение 1: Сортировка строк как ключ

**Идея:** Использовать отсортированную строку как ключ в HashMap.

**Алгоритм:**
1. Создаем map: отсортированная_строка → список слов
2. Для каждого слова:
   - Сортируем его буквы
   - Добавляем в группу по ключу
3. Собираем все группы в результат

**Сложность:** O(n * k log k) по времени, O(n * k) по памяти
- n - количество строк
- k - максимальная длина строки
- k log k - сортировка каждой строки

```go
func groupAnagrams(strs []string) [][]string {
    // Карта: отсортированная строка → группа анаграмм
    groups := make(map[string][]string)

    for _, str := range strs {
        // Сортируем символы строки
        key := sortString(str)

        // Добавляем в группу
        groups[key] = append(groups[key], str)
    }

    // Собираем все группы
    result := make([][]string, 0, len(groups))
    for _, group := range groups {
        result = append(result, group)
    }

    return result
}

func sortString(s string) string {
    // Конвертируем в слайс рун для сортировки
    runes := []rune(s)
    sort.Slice(runes, func(i, j int) bool {
        return runes[i] < runes[j]
    })
    return string(runes)
}
```

## Решение 2: Подсчет символов как ключ (Оптимальное)

**Идея:** Создать строковый ключ на основе подсчета символов.

**Преимущество:** Избегаем сортировки O(k log k), делаем O(k).

**Алгоритм:**
1. Для каждого слова создаем ключ: количество каждой буквы
2. Ключ в формате: "#0#1#1#0#0#..." (количества букв a-z)
3. Группируем по этому ключу

**Сложность:** O(n * k) по времени, O(n * k) по памяти

```go
func groupAnagramsOptimal(strs []string) [][]string {
    groups := make(map[string][]string)

    for _, str := range strs {
        // Создаем ключ на основе подсчета символов
        key := getCharCountKey(str)
        groups[key] = append(groups[key], str)
    }

    result := make([][]string, 0, len(groups))
    for _, group := range groups {
        result = append(result, group)
    }

    return result
}

func getCharCountKey(s string) string {
    // Массив для подсчета символов a-z
    count := [26]int{}

    for _, ch := range s {
        count[ch-'a']++
    }

    // Строим ключ: "#0#1#0#2#..." для каждой буквы
    var key strings.Builder
    for _, c := range count {
        key.WriteByte('#')
        key.WriteString(strconv.Itoa(c))
    }

    return key.String()
}
```

## Пошаговое объяснение

Рассмотрим пример: `strs = ["eat","tea","tan","ate","nat","bat"]`

### Решение 1: С сортировкой

```
Слово  | Отсортировано | Группа
-------|---------------|-------
"eat"  | "aet"         | ["eat"]
"tea"  | "aet"         | ["eat", "tea"]
"tan"  | "ant"         | ["tan"]
"ate"  | "aet"         | ["eat", "tea", "ate"]
"nat"  | "ant"         | ["tan", "nat"]
"bat"  | "abt"         | ["bat"]

Результат:
[["eat","tea","ate"], ["tan","nat"], ["bat"]]
```

### Решение 2: С подсчетом

```
Слово  | Ключ                        | Группа
-------|-----------------------------|---------
"eat"  | "#1#0#0#0#1#0...#1#0..."   | ["eat"]
"tea"  | "#1#0#0#0#1#0...#1#0..."   | ["eat","tea"]
"tan"  | "#1#0#0#0#0#0...#1#0..."   | ["tan"]
"ate"  | "#1#0#0#0#1#0...#1#0..."   | ["eat","tea","ate"]
...

(Одинаковый ключ для анаграмм)
```

## Визуализация

```
Входной массив:
["eat", "tea", "tan", "ate", "nat", "bat"]

        ↓ Группировка по отсортированным буквам

Группа "aet":    Группа "ant":    Группа "abt":
  eat               tan               bat
  tea               nat
  ate

Результат:
[["eat","tea","ate"], ["tan","nat"], ["bat"]]
```

## Полное решение

```go
package main

import (
    "fmt"
    "sort"
    "strconv"
    "strings"
)

// Решение 1: С сортировкой
func groupAnagrams(strs []string) [][]string {
    groups := make(map[string][]string)

    for _, str := range strs {
        key := sortString(str)
        groups[key] = append(groups[key], str)
    }

    result := make([][]string, 0, len(groups))
    for _, group := range groups {
        result = append(result, group)
    }

    return result
}

func sortString(s string) string {
    runes := []rune(s)
    sort.Slice(runes, func(i, j int) bool {
        return runes[i] < runes[j]
    })
    return string(runes)
}

// Решение 2: С подсчетом символов (оптимальное)
func groupAnagramsOptimal(strs []string) [][]string {
    groups := make(map[string][]string)

    for _, str := range strs {
        key := getCharCountKey(str)
        groups[key] = append(groups[key], str)
    }

    result := make([][]string, 0, len(groups))
    for _, group := range groups {
        result = append(result, group)
    }

    return result
}

func getCharCountKey(s string) string {
    count := [26]int{}

    for _, ch := range s {
        count[ch-'a']++
    }

    var key strings.Builder
    for i, c := range count {
        if c > 0 {
            key.WriteByte(byte('a' + i))
            key.WriteString(strconv.Itoa(c))
        }
    }

    return key.String()
}

func main() {
    strs := []string{"eat", "tea", "tan", "ate", "nat", "bat"}

    fmt.Println("Входные слова:", strs)

    result := groupAnagrams(strs)
    fmt.Println("\nГруппы анаграмм:")
    for i, group := range result {
        fmt.Printf("Группа %d: %v\n", i+1, group)
    }
}
```

## Тестовые случаи

```go
func TestGroupAnagrams(t *testing.T) {
    tests := []struct {
        input    []string
        expected int  // Количество групп
    }{
        {
            input:    []string{"eat", "tea", "tan", "ate", "nat", "bat"},
            expected: 3,
        },
        {
            input:    []string{""},
            expected: 1,
        },
        {
            input:    []string{"a"},
            expected: 1,
        },
        {
            input:    []string{"abc", "bca", "cab", "xyz", "zyx", "yxz"},
            expected: 2,
        },
        {
            input:    []string{"a", "b", "c"},
            expected: 3,  // Нет анаграмм
        },
        {
            input:    []string{"aaa", "aaa", "aaa"},
            expected: 1,  // Все одинаковые
        },
    }

    for _, tt := range tests {
        result := groupAnagrams(tt.input)

        if len(result) != tt.expected {
            t.Errorf("groupAnagrams(%v) returned %d groups, expected %d",
                tt.input, len(result), tt.expected)
        }
    }
}
```

## Альтернативная реализация с массивом как ключом

```go
func groupAnagramsArray(strs []string) [][]string {
    // Используем массив [26]int как ключ напрямую
    // Преобразуем в строку для использования в map

    type Key [26]int

    groups := make(map[Key][]string)

    for _, str := range strs {
        var key Key
        for _, ch := range str {
            key[ch-'a']++
        }
        groups[key] = append(groups[key], str)
    }

    result := make([][]string, 0, len(groups))
    for _, group := range groups {
        result = append(result, group)
    }

    return result
}
```

**Преимущество:** Не нужно конвертировать массив в строку, массив используется как ключ напрямую.

## Сравнение подходов

| Подход | Время | Память | Преимущества |
|--------|-------|--------|--------------|
| Сортировка | O(n * k log k) | O(n * k) | Простая реализация |
| Подсчет символов | O(n * k) | O(n * k) | Быстрее, оптимально |
| Массив как ключ | O(n * k) | O(n * k) | Не нужна конвертация |

## Вариации задачи

### 1. Найти все анаграммы подстроки в строке

```go
func findAnagrams(s string, p string) []int {
    if len(s) < len(p) {
        return []int{}
    }

    result := []int{}
    pCount := [26]int{}
    windowCount := [26]int{}

    // Подсчет символов в p
    for _, ch := range p {
        pCount[ch-'a']++
    }

    // Скользящее окно
    for i := 0; i < len(s); i++ {
        // Добавляем символ в окно
        windowCount[s[i]-'a']++

        // Удаляем символ за окном
        if i >= len(p) {
            windowCount[s[i-len(p)]-'a']--
        }

        // Проверяем совпадение
        if i >= len(p)-1 && pCount == windowCount {
            result = append(result, i-len(p)+1)
        }
    }

    return result
}
```

### 2. Проверка, являются ли две строки анаграммами

```go
func isAnagram(s string, t string) bool {
    if len(s) != len(t) {
        return false
    }

    count := [26]int{}

    for i := 0; i < len(s); i++ {
        count[s[i]-'a']++
        count[t[i]-'a']--
    }

    for _, c := range count {
        if c != 0 {
            return false
        }
    }

    return true
}
```

## Где спрашивают

- **LeetCode:** #49 - Group Anagrams
- **Яндекс:** Часто на алгоритмических раундах
- **Авито:** Могут спросить оптимизацию
- **Т-Банк:** Просят объяснить сложность разных подходов
- **Озон:** Классическая задача на строки и HashMap

## Вопросы с собеседований

**Вопрос 1:** Почему подход с подсчетом быстрее сортировки?

**Ответ:** Подсчет символов O(k), а сортировка O(k log k). Для алфавита из 26 букв подсчет всегда быстрее.

---

**Вопрос 2:** Можно ли использовать просто map[[]int][]string?

**Ответ:** Нет, слайсы в Go не сравниваемы и не могут быть ключами map. Нужно использовать массив [26]int или конвертировать в строку.

---

**Вопрос 3:** Что если строки содержат Unicode символы, не только a-z?

**Ответ:** Нужно использовать `map[rune]int` для подсчета вместо `[26]int`. Ключ строится аналогично, но для всех уникальных символов.

---

**Вопрос 4:** Как оптимизировать для очень длинных строк?

**Ответ:**
1. Использовать подсчет символов вместо сортировки
2. Можно сначала сравнивать длины строк
3. Использовать хеш вместо полного ключа (с учетом коллизий)

## Best Practices

1. ✅ Для английских букв используйте подсчет символов [26]int
2. ✅ Массив как ключ быстрее, чем конвертация в строку
3. ✅ Предаллоцируйте capacity для result если знаете примерный размер
4. ✅ Для Unicode используйте map[rune]int
5. ❌ Не забывайте про edge cases (пустая строка)
6. ❌ Помните, что слайсы нельзя использовать как ключи map

## Применение в реальности

### Поиск похожих документов

```go
type Document struct {
    ID      string
    Content string
}

func findSimilarDocuments(docs []Document) map[string][]Document {
    groups := make(map[string][]Document)

    for _, doc := range docs {
        // Нормализуем: lowercase, убираем пробелы
        normalized := strings.ToLower(strings.ReplaceAll(doc.Content, " ", ""))

        // Используем как ключ отсортированные символы
        key := sortString(normalized)

        groups[key] = append(groups[key], doc)
    }

    return groups
}
```

### Обнаружение плагиата

```go
func detectPlagiarism(texts []string) [][]int {
    // Группируем по "отпечатку" - подсчету слов
    fingerprints := make(map[string][]int)

    for i, text := range texts {
        fp := getTextFingerprint(text)
        fingerprints[fp] = append(fingerprints[fp], i)
    }

    // Возвращаем группы с более чем одним документом
    var duplicates [][]int
    for _, group := range fingerprints {
        if len(group) > 1 {
            duplicates = append(duplicates, group)
        }
    }

    return duplicates
}

func getTextFingerprint(text string) string {
    words := strings.Fields(strings.ToLower(text))
    sort.Strings(words)
    return strings.Join(words, ",")
}
```

## Связанные темы

- [[Go - Карты (maps)]]
- [[HashMap - Реализация и особенности]]
- [[Алгоритмы на строках]]
- [[Алгоритмическая сложность (Big O)]]
- [[Сортировки - Быстрая, слиянием, пузырьком]]
- [[Go - Строки и UTF-8]]
