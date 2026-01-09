# Go - Строки и UTF-8

Строки в Go - неизменяемые (immutable) последовательности байт, обычно содержащие UTF-8 текст.

## Основы

### Объявление

```go
// Обычные строки (interpreted string literal)
s1 := "Hello, World!"
s2 := "Привет, мир!"

// Raw strings (обратные кавычки)
s3 := `Line 1
Line 2
Line 3`

// Escape sequences работают только в ""
s4 := "Line 1\nLine 2\tTabbed"
s5 := `Line 1\nLine 2\tTabbed` // Буквально: \n и \t
```

### Внутреннее представление

Строка - это структура:
```go
type string struct {
    ptr *byte  // Указатель на данные
    len int    // Длина в байтах
}
```

```go
import "unsafe"

s := "hello"
fmt.Println(unsafe.Sizeof(s)) // 16 bytes (на 64-bit системе)
```

## Длина строки

### len() - длина в байтах

```go
s1 := "Hello"
fmt.Println(len(s1)) // 5 (5 ASCII символов = 5 байт)

s2 := "Привет"
fmt.Println(len(s2)) // 12 (6 символов × 2 байта)

s3 := "日本語"
fmt.Println(len(s3)) // 9 (3 символа × 3 байта)
```

**len() возвращает количество БАЙТ, не символов!**

### utf8.RuneCountInString() - количество символов

```go
import "unicode/utf8"

s := "Привет"
fmt.Println(len(s))                    // 12 байт
fmt.Println(utf8.RuneCountInString(s)) // 6 символов (rune)
```

## Rune - символ Unicode

`rune` - это алиас для `int32`, представляющий Unicode code point:

```go
var r rune = 'П'
fmt.Printf("%c %d %U\n", r, r, r)
// П 1055 U+041F

// Итерация по символам
s := "Привет"
for index, r := range s {
    fmt.Printf("%d: %c\n", index, r)
}
// 0: П
// 2: р (индекс 2, т.к. 'П' занимает 2 байта!)
// 4: и
// 6: в
// 8: е
// 10: т
```

## UTF-8 кодирование

UTF-8 - переменной длины кодирование:
- ASCII (0-127): 1 байт
- Кириллица, греческий: 2 байта
- Большинство Азиатских языков: 3 байта
- Редкие символы, эмодзи: 4 байта

```go
fmt.Println(len("A"))    // 1 byte (ASCII)
fmt.Println(len("П"))    // 2 bytes (Кириллица)
fmt.Println(len("日"))   // 3 bytes (Японский)
fmt.Println(len("😀"))   // 4 bytes (Эмодзи)
```

### Декодирование UTF-8

```go
import "unicode/utf8"

s := "Hello, 世界"

// Декодирование по одному символу
for i := 0; i < len(s); {
    r, size := utf8.DecodeRuneInString(s[i:])
    fmt.Printf("%c (size: %d)\n", r, size)
    i += size
}
```

## Индексация строк

### По байтам

```go
s := "Hello"
fmt.Println(s[0])   // 72 (байт, не символ!)
fmt.Println(s[1])   // 101

// Преобразование в string
fmt.Println(string(s[0])) // "H"
```

**⚠️ Для не-ASCII это не работает как ожидается:**

```go
s := "Привет"
fmt.Println(string(s[0])) // "П" (2 байта!)
fmt.Println(string(s[1])) // "�" (невалидный UTF-8!)
```

### Срезы строк (slicing)

```go
s := "Hello, World!"
fmt.Println(s[0:5])  // "Hello"
fmt.Println(s[7:])   // "World!"
fmt.Println(s[:5])   // "Hello"

// С не-ASCII - работает по байтам!
s2 := "Привет"
fmt.Println(s2[0:2])  // "П" (первый символ = 2 байта)
fmt.Println(s2[0:4])  // "Пр" (2 символа)
```

## Итерация

### range - по Unicode символам (rune)

```go
s := "Привет"
for index, r := range s {
    fmt.Printf("%d: %c (%d bytes)\n", index, r, utf8.RuneLen(r))
}
// 0: П (2 bytes)
// 2: р (2 bytes)
// 4: и (2 bytes)
// 6: в (2 bytes)
// 8: е (2 bytes)
// 10: т (2 bytes)
```

**Индекс указывает на начало байта символа в строке!**

### Цикл for - по байтам

```go
s := "Hello"
for i := 0; i < len(s); i++ {
    fmt.Printf("%c ", s[i])
}
// H e l l o
```

## Неизменяемость строк

Строки immutable - нельзя изменить:

```go
s := "Hello"
// s[0] = 'h' // ❌ Ошибка компиляции!

// Создание новой строки
s = "h" + s[1:] // "hello"
```

**Для изменения используйте []rune или []byte:**

```go
s := "Hello"

// Через []rune (для Unicode)
runes := []rune(s)
runes[0] = 'h'
s = string(runes) // "hello"

// Через []byte (для ASCII)
bytes := []byte(s)
bytes[0] = 'H'
s = string(bytes) // "Hello"
```

## Конкатенация строк

### Оператор +

```go
s1 := "Hello"
s2 := "World"
result := s1 + ", " + s2 + "!" // "Hello, World!"
```

**⚠️ Неэффективно в цикле:**

```go
// ❌ Медленно - многократное копирование
var result string
for i := 0; i < 1000; i++ {
    result += "text"
}
```

### strings.Builder (эффективно)

```go
import "strings"

// ✅ Быстро
var builder strings.Builder
for i := 0; i < 1000; i++ {
    builder.WriteString("text")
}
result := builder.String()
```

**С предварительным резервированием:**

```go
var builder strings.Builder
builder.Grow(4000) // Резервируем память

for i := 0; i < 1000; i++ {
    builder.WriteString("text")
}
result := builder.String()
```

### fmt.Sprintf

```go
name := "Alice"
age := 25
s := fmt.Sprintf("Name: %s, Age: %d", name, age)
// "Name: Alice, Age: 25"
```

## Пакет strings

### Основные функции

```go
import "strings"

s := "Hello, World!"

// Поиск
strings.Contains(s, "World")     // true
strings.HasPrefix(s, "Hello")    // true
strings.HasSuffix(s, "!")        // true
strings.Index(s, "World")        // 7
strings.LastIndex(s, "o")        // 8
strings.Count(s, "l")            // 3

// Изменение
strings.ToUpper(s)               // "HELLO, WORLD!"
strings.ToLower(s)               // "hello, world!"
strings.TrimSpace("  text  ")    // "text"
strings.Trim("---text---", "-")  // "text"
strings.Replace(s, "World", "Go", 1) // "Hello, Go!"
strings.ReplaceAll(s, "l", "L")  // "HeLLo, WorLd!"

// Разделение/объединение
strings.Split("a,b,c", ",")      // []string{"a", "b", "c"}
strings.Join([]string{"a", "b"}, ",") // "a,b"
strings.Fields("a  b\tc\nd")     // []string{"a", "b", "c", "d"}

// Повторение
strings.Repeat("Go", 3)          // "GoGoGo"
```

### Сравнение

```go
s1 := "Hello"
s2 := "hello"

// Регистрозависимое
s1 == s2 // false

// Регистронезависимое
strings.EqualFold(s1, s2) // true

// Лексикографическое
strings.Compare("abc", "def") // -1 (abc < def)
strings.Compare("abc", "abc") // 0
strings.Compare("def", "abc") // 1 (def > abc)
```

## Пакет unicode/utf8

```go
import "unicode/utf8"

s := "Привет, 世界!"

// Проверка валидности UTF-8
utf8.ValidString(s) // true

// Количество символов
utf8.RuneCountInString(s) // 10

// Декодирование
r, size := utf8.DecodeRuneInString(s)
// r = 'П', size = 2

// Длина rune в байтах
utf8.RuneLen('П')  // 2
utf8.RuneLen('世') // 3
utf8.RuneLen('A')  // 1
```

## Преобразование типов

### string ↔ []byte

```go
s := "Hello"

// string → []byte
bytes := []byte(s)
bytes[0] = 'h'
fmt.Println(bytes) // [104 101 108 108 111]

// []byte → string
s2 := string(bytes)
fmt.Println(s2) // "hello"
```

### string ↔ []rune

```go
s := "Привет"

// string → []rune
runes := []rune(s)
runes[0] = 'п'
fmt.Println(runes) // [1087 1088 1080 1074 1077 1090]

// []rune → string
s2 := string(runes)
fmt.Println(s2) // "привет"
```

### string ↔ числа

```go
import "strconv"

// string → int
i, err := strconv.Atoi("42")

// int → string
s := strconv.Itoa(42)

// ParseInt, ParseFloat
i64, err := strconv.ParseInt("42", 10, 64)
f64, err := strconv.ParseFloat("3.14", 64)

// FormatInt, FormatFloat
s = strconv.FormatInt(42, 10)
s = strconv.FormatFloat(3.14, 'f', 2, 64)
```

## Многострочные строки

### Raw string literals

```go
query := `
SELECT id, name, email
FROM users
WHERE active = true
  AND age > 18
ORDER BY name
`

html := `
<html>
  <body>
    <h1>Title</h1>
  </body>
</html>
`
```

### С интерполяцией (template)

```go
import "text/template"

tmpl := `Hello, {{.Name}}! You are {{.Age}} years old.`
t := template.Must(template.New("greeting").Parse(tmpl))

data := struct {
    Name string
    Age  int
}{
    Name: "Alice",
    Age:  25,
}

var buf bytes.Buffer
t.Execute(&buf, data)
fmt.Println(buf.String())
// Hello, Alice! You are 25 years old.
```

## Паттерны

### Эффективная конкатенация в цикле

```go
// ❌ Неэффективно
var result string
for _, item := range items {
    result += item + "\n"
}

// ✅ Эффективно
var builder strings.Builder
builder.Grow(len(items) * 10) // Примерный размер
for _, item := range items {
    builder.WriteString(item)
    builder.WriteByte('\n')
}
result := builder.String()
```

### Проверка пустой строки

```go
s := "  "

// Пустая строка
if s == "" {
    // ...
}

// Пустая или только пробелы
if strings.TrimSpace(s) == "" {
    // ...
}

// Длина 0
if len(s) == 0 {
    // ...
}
```

### Обратная строка (reverse)

```go
func Reverse(s string) string {
    runes := []rune(s)
    for i, j := 0, len(runes)-1; i < j; i, j = i+1, j-1 {
        runes[i], runes[j] = runes[j], runes[i]
    }
    return string(runes)
}

fmt.Println(Reverse("Hello"))  // "olleH"
fmt.Println(Reverse("Привет")) // "тевирП"
```

## Частые ошибки

### 1. len() для подсчета символов

```go
// ❌ Неправильно для не-ASCII
s := "Привет"
fmt.Println(len(s)) // 12 (байт, не символов!)

// ✅ Правильно
fmt.Println(utf8.RuneCountInString(s)) // 6 символов
```

### 2. Индексация не-ASCII строк

```go
// ❌ Неправильно
s := "Привет"
fmt.Println(s[1]) // Байт, не символ!

// ✅ Правильно
runes := []rune(s)
fmt.Println(runes[1]) // 'р'
```

### 3. Конкатенация в цикле

```go
// ❌ Медленно - O(n²)
var result string
for i := 0; i < 10000; i++ {
    result += "text"
}

// ✅ Быстро - O(n)
var builder strings.Builder
for i := 0; i < 10000; i++ {
    builder.WriteString("text")
}
result := builder.String()
```

### 4. Изменение строки

```go
// ❌ Нельзя изменить напрямую
s := "Hello"
// s[0] = 'h' // Ошибка компиляции!

// ✅ Через []rune
runes := []rune(s)
runes[0] = 'h'
s = string(runes)
```

## Производительность

### Сложность операций

| Операция | Сложность |
|----------|-----------|
| len(s) | O(1) |
| s[i] | O(1) |
| s + t | O(n+m) |
| strings.Contains | O(n×m) |
| strings.Index | O(n×m) |
| utf8.RuneCountInString | O(n) |

### Оптимизация

```go
// ❌ Многократный поиск
for i := 0; i < 1000; i++ {
    if strings.Contains(longString, "pattern") {
        // ...
    }
}

// ✅ Один поиск
found := strings.Contains(longString, "pattern")
for i := 0; i < 1000; i++ {
    if found {
        // ...
    }
}
```

## Best Practices

1. ✅ Используйте `utf8.RuneCountInString()` для подсчета символов
2. ✅ Используйте `strings.Builder` для конкатенации в цикле
3. ✅ Используйте raw strings для многострочного текста
4. ✅ Преобразуйте в `[]rune` для работы с отдельными символами
5. ✅ Используйте `range` для итерации по символам (не байтам)
6. ❌ Не используйте `len()` для подсчета символов в Unicode
7. ❌ Не конкатенируйте строки в цикле через `+`
8. ❌ Не индексируйте не-ASCII строки напрямую

## Связанные темы

- [[Go - Типы данных]]
- [[Go - Массивы и слайсы]]
- [[Go - Пакет fmt]]
- [[Алгоритмы на строках]]
