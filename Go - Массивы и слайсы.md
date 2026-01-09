# Go - Массивы и слайсы

## Массивы

Массив - коллекция фиксированного размера элементов одного типа.

### Объявление и инициализация

```go
// Объявление с zero values
var arr [5]int // [0 0 0 0 0]

// С инициализацией
var arr = [5]int{1, 2, 3, 4, 5}

// Короткое объявление
arr := [3]string{"a", "b", "c"}

// Автоматический размер
arr := [...]int{1, 2, 3, 4, 5} // Размер 5

// Частичная инициализация
arr := [5]int{1, 2} // [1 2 0 0 0]

// С индексами
arr := [5]int{1: 10, 3: 30} // [0 10 0 30 0]
```

### Размер - часть типа!

```go
var a [3]int
var b [4]int

// a и b - РАЗНЫЕ типы!
// b = a // ❌ Ошибка компиляции

// Размер известен на этапе компиляции
func process(arr [5]int) {
    // Принимает ТОЛЬКО массивы размера 5
}
```

### Операции

```go
arr := [5]int{1, 2, 3, 4, 5}

// Длина (константа!)
len(arr) // 5

// Доступ по индексу
arr[0] = 10
value := arr[2]

// Итерация
for i := 0; i < len(arr); i++ {
    fmt.Println(arr[i])
}

// range
for index, value := range arr {
    fmt.Printf("%d: %d\n", index, value)
}

// Только значения
for _, value := range arr {
    fmt.Println(value)
}
```

### Массивы - value type

```go
// Копирование массива
a := [3]int{1, 2, 3}
b := a // Копия!

b[0] = 99
fmt.Println(a[0]) // 1 (не изменился)
fmt.Println(b[0]) // 99

// Передача в функцию - копия
func modify(arr [3]int) {
    arr[0] = 99 // Изменяет копию
}

a := [3]int{1, 2, 3}
modify(a)
fmt.Println(a[0]) // 1 (не изменился)

// Для изменения нужен указатель
func modify(arr *[3]int) {
    arr[0] = 99
}

modify(&a)
fmt.Println(a[0]) // 99
```

**На практике массивы используются редко. Используйте слайсы!**

## Слайсы

Слайс - динамическое представление части массива.

### Структура slice

```
Slice header (24 bytes):
┌──────────────┬──────────────┬──────────────┐
│   pointer    │    length    │   capacity   │
│   (8 bytes)  │   (8 bytes)  │   (8 bytes)  │
└──────────────┴──────────────┴──────────────┘
       │
       └─────> [underlying array]
```

### Создание слайсов

```go
// Nil slice
var slice []int // nil, len=0, cap=0

// Пустой slice (не nil!)
slice := []int{} // len=0, cap=0

// С элементами
slice := []int{1, 2, 3, 4, 5}

// make - создание с capacity
slice := make([]int, 5)      // len=5, cap=5, [0 0 0 0 0]
slice := make([]int, 5, 10)  // len=5, cap=10, [0 0 0 0 0]

// Из массива
arr := [5]int{1, 2, 3, 4, 5}
slice := arr[:]  // Весь массив
slice := arr[1:4] // [2 3 4]
```

### Slicing операции

```go
slice := []int{0, 1, 2, 3, 4, 5}

// slice[low:high]
slice[1:4]  // [1 2 3]
slice[:3]   // [0 1 2]
slice[3:]   // [3 4 5]
slice[:]    // [0 1 2 3 4 5]

// slice[low:high:max] - с max capacity
s := slice[1:3:5]
// len=2 (3-1), cap=4 (5-1), [1 2]
```

**Правила:**
- `0 <= low <= high <= len(slice)`
- `low <= max <= cap(slice)`

### append()

```go
slice := []int{1, 2, 3}

// Добавление элемента
slice = append(slice, 4)
// [1 2 3 4]

// Добавление нескольких
slice = append(slice, 5, 6, 7)
// [1 2 3 4 5 6 7]

// Добавление другого slice
other := []int{8, 9}
slice = append(slice, other...)
// [1 2 3 4 5 6 7 8 9]
```

**Важно:** `append` возвращает новый slice, может перевыделить память!

```go
slice := []int{1, 2, 3}
fmt.Println(cap(slice)) // 3

slice = append(slice, 4)
fmt.Println(cap(slice)) // 6 (удвоилась!)

// ⚠️ НЕ присваивайте результат - потеряете данные
slice := []int{1, 2, 3}
append(slice, 4) // ❌ Результат потерян!

slice = append(slice, 4) // ✅ Правильно
```

### copy()

```go
src := []int{1, 2, 3, 4, 5}
dst := make([]int, 5)

// Копирование
n := copy(dst, src)
// dst = [1 2 3 4 5], n = 5

// Частичное копирование
dst := make([]int, 3)
copy(dst, src)
// dst = [1 2 3] (скопировано min(len(dst), len(src)))

// Копирование overlapping slices - безопасно
slice := []int{1, 2, 3, 4, 5}
copy(slice[1:], slice[:])
// [1 1 2 3 4]
```

### len() и cap()

```go
slice := make([]int, 5, 10)

len(slice) // 5 - количество элементов
cap(slice) // 10 - емкость

slice = append(slice, 1, 2, 3)
len(slice) // 8
cap(slice) // 10

slice = append(slice, 4, 5, 6)
len(slice) // 11
cap(slice) // 20 (перевыделение!)
```

### Слайсы - reference type

```go
// Разные слайсы могут ссылаться на один массив
slice1 := []int{1, 2, 3, 4, 5}
slice2 := slice1[1:4] // [2 3 4]

// Изменение в slice2 влияет на slice1!
slice2[0] = 99

fmt.Println(slice1) // [1 99 3 4 5]
fmt.Println(slice2) // [99 3 4]
```

**Для независимой копии используйте copy:**
```go
slice1 := []int{1, 2, 3, 4, 5}
slice2 := make([]int, len(slice1))
copy(slice2, slice1)

slice2[0] = 99
fmt.Println(slice1) // [1 2 3 4 5] (не изменился)
```

### nil slice vs empty slice

```go
// nil slice
var nilSlice []int
nilSlice == nil         // true
len(nilSlice)           // 0
cap(nilSlice)           // 0
nilSlice = append(nilSlice, 1) // ✅ Работает!

// Empty slice (not nil)
emptySlice := []int{}
emptySlice == nil       // false
len(emptySlice)         // 0
cap(emptySlice)         // 0

// Или
emptySlice := make([]int, 0)

// Оба можно использовать одинаково
// Предпочитайте nil slice
```

### Рост capacity

При `append` capacity растет по формуле:
- Если `cap < 256`: новый cap = 2 * старый cap
- Если `cap >= 256`: новый cap ≈ 1.25 * старый cap

```go
s := []int{}
for i := 0; i < 20; i++ {
    s = append(s, i)
    fmt.Printf("len=%d cap=%d\n", len(s), cap(s))
}

// len=1 cap=1
// len=2 cap=2
// len=3 cap=4
// len=4 cap=4
// len=5 cap=8
// len=9 cap=16
// len=17 cap=32
```

### Pre-allocation для производительности

```go
// ❌ Медленно - многократное перевыделение
var result []int
for i := 0; i < 10000; i++ {
    result = append(result, i)
}

// ✅ Быстро - одно выделение
result := make([]int, 0, 10000)
for i := 0; i < 10000; i++ {
    result = append(result, i)
}

// Еще быстрее - известна длина
result := make([]int, 10000)
for i := 0; i < 10000; i++ {
    result[i] = i
}
```

## Многомерные слайсы

```go
// 2D slice
matrix := [][]int{
    {1, 2, 3},
    {4, 5, 6},
    {7, 8, 9},
}

// Создание 2D slice
rows, cols := 3, 4
matrix := make([][]int, rows)
for i := range matrix {
    matrix[i] = make([]int, cols)
}
```

## Паттерны

### Удаление элемента

```go
slice := []int{1, 2, 3, 4, 5}

// Удалить элемент по индексу (с сохранением порядка)
index := 2
slice = append(slice[:index], slice[index+1:]...)
// [1 2 4 5]

// Удалить элемент (без сохранения порядка) - быстрее
index := 2
slice[index] = slice[len(slice)-1]
slice = slice[:len(slice)-1]
// [1 2 5 4]
```

### Фильтрация

```go
// In-place фильтрация
slice := []int{1, 2, 3, 4, 5, 6}
n := 0
for _, v := range slice {
    if v%2 == 0 { // Оставляем четные
        slice[n] = v
        n++
    }
}
slice = slice[:n]
// [2 4 6]

// Новый slice
var result []int
for _, v := range slice {
    if v%2 == 0 {
        result = append(result, v)
    }
}
```

### Вставка

```go
slice := []int{1, 2, 5, 6}
index := 2
value := 99

// Вставка value на позицию index
slice = append(slice[:index], append([]int{value}, slice[index:]...)...)
// [1 2 99 5 6]

// Более эффективно
slice = append(slice, 0)        // Расширяем
copy(slice[index+1:], slice[index:]) // Сдвигаем
slice[index] = value            // Вставляем
```

### Reverse

```go
func reverse(s []int) {
    for i, j := 0, len(s)-1; i < j; i, j = i+1, j-1 {
        s[i], s[j] = s[j], s[i]
    }
}

slice := []int{1, 2, 3, 4, 5}
reverse(slice)
// [5 4 3 2 1]
```

## Частые ошибки

### 1. Игнорирование результата append

```go
// ❌ Результат потерян
slice := []int{1, 2, 3}
append(slice, 4)
fmt.Println(slice) // [1 2 3]

// ✅ Присваиваем результат
slice = append(slice, 4)
fmt.Println(slice) // [1 2 3 4]
```

### 2. Неожиданное поведение при slicing

```go
slice1 := []int{1, 2, 3, 4, 5}
slice2 := slice1[1:3] // [2 3]

// Изменение slice2 влияет на slice1!
slice2[0] = 99
fmt.Println(slice1) // [1 99 3 4 5]

// append может перевыделить память
slice2 = append(slice2, 100, 200)
slice2[0] = 77
fmt.Println(slice1) // Может быть [1 99 3 4 5] или [1 77 3 4 5]!
```

### 3. Утечка памяти при slicing большого slice

```go
// ❌ Удерживает весь большой массив в памяти
func getFirst100(bigSlice []byte) []byte {
    return bigSlice[:100] // Ссылается на весь bigSlice!
}

// ✅ Копируем нужную часть
func getFirst100(bigSlice []byte) []byte {
    result := make([]byte, 100)
    copy(result, bigSlice[:100])
    return result // bigSlice может быть освобожден GC
}
```

### 4. Модификация slice в range

```go
// ❌ Изменяет копии
slice := []Point{{X: 1}, {X: 2}, {X: 3}}
for _, p := range slice {
    p.X = 99 // Изменяет копию!
}
// slice не изменился

// ✅ Через индекс
for i := range slice {
    slice[i].X = 99
}

// ✅ Slice указателей
slice := []*Point{{X: 1}, {X: 2}, {X: 3}}
for _, p := range slice {
    p.X = 99 // Изменяет через указатель
}
```

## Best Practices

1. ✅ Используйте слайсы вместо массивов
2. ✅ Pre-allocate когда знаете размер
3. ✅ Всегда присваивайте результат append
4. ✅ Копируйте slice когда нужна независимая копия
5. ✅ Используйте nil slice для пустых коллекций
6. ❌ Не передавайте массивы по значению (используйте слайсы)

## Связанные темы

- [[Go - Типы данных]]
- [[Go - Указатели и ссылки]]
- [[Go - Карты (maps)]]
- [[Алгоритмическая сложность (Big O)]]
