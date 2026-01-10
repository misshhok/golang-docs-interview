# Go - Внутреннее устройство slice

Slice (срез) — это динамический массив в Go. Понимание внутреннего устройства slice критически важно для написания эффективного кода и избежания распространённых ошибок.

## Структура slice

Slice — это **дескриптор** (заголовок), состоящий из трёх полей:

```go
// runtime/slice.go
type slice struct {
    array unsafe.Pointer  // указатель на underlying array
    len   int             // текущая длина
    cap   int             // ёмкость (capacity)
}
```

```
┌─────────────────────────────────────────────────────────────┐
│                      SLICE HEADER                            │
│                       (24 bytes)                             │
├─────────────────┬─────────────────┬─────────────────────────┤
│     array       │      len        │         cap             │
│   (8 bytes)     │   (8 bytes)     │      (8 bytes)          │
│   pointer       │      int        │         int             │
└────────┬────────┴─────────────────┴─────────────────────────┘
         │
         ▼
┌────┬────┬────┬────┬────┬────┬────┬────┐
│ 10 │ 20 │ 30 │ 40 │ 50 │  0 │  0 │  0 │  ← Underlying Array
└────┴────┴────┴────┴────┴────┴────┴────┘
  0    1    2    3    4    5    6    7
                     ▲              ▲
                    len=5         cap=8
```

```go
s := make([]int, 5, 8)  // len=5, cap=8
s[0], s[1], s[2], s[3], s[4] = 10, 20, 30, 40, 50

// Под капотом:
// s.array → указатель на [10, 20, 30, 40, 50, 0, 0, 0]
// s.len = 5
// s.cap = 8
```

## Создание slice

### Различные способы

```go
// 1. Литерал
s1 := []int{1, 2, 3}
// len=3, cap=3, array=[1,2,3]

// 2. make с len
s2 := make([]int, 5)
// len=5, cap=5, array=[0,0,0,0,0]

// 3. make с len и cap
s3 := make([]int, 3, 10)
// len=3, cap=10, array=[0,0,0,0,0,0,0,0,0,0]

// 4. Из массива
arr := [5]int{1, 2, 3, 4, 5}
s4 := arr[1:4]
// len=3, cap=4, array=&arr[1] (указывает на arr!)

// 5. nil slice
var s5 []int
// len=0, cap=0, array=nil

// 6. empty slice
s6 := []int{}
// len=0, cap=0, array=не nil (указывает на zerobase)
```

### nil slice vs empty slice

```go
var nilSlice []int        // nil slice
emptySlice := []int{}     // empty slice
makeSlice := make([]int, 0)  // empty slice

fmt.Println(nilSlice == nil)   // true
fmt.Println(emptySlice == nil) // false
fmt.Println(makeSlice == nil)  // false

// Но len и cap одинаковые!
fmt.Println(len(nilSlice), cap(nilSlice))     // 0 0
fmt.Println(len(emptySlice), cap(emptySlice)) // 0 0

// JSON encoding отличается!
json.Marshal(nilSlice)   // null
json.Marshal(emptySlice) // []
```

## Операция append

### Как работает append

```go
func append(slice []T, elements ...T) []T
```

**Алгоритм:**

1. Если `len + новые_элементы <= cap` → добавляем в существующий массив
2. Если `len + новые_элементы > cap` → создаём новый массив большего размера и копируем

```go
s := make([]int, 3, 5)  // len=3, cap=5
s[0], s[1], s[2] = 1, 2, 3

// Добавление без реаллокации (cap достаточно)
s = append(s, 4)  // len=4, cap=5, тот же underlying array

// Ещё одно добавление
s = append(s, 5)  // len=5, cap=5, тот же underlying array

// Реаллокация! cap исчерпан
s = append(s, 6)  // len=6, cap=10 (новый массив!)
```

### Стратегия роста capacity

```go
// До Go 1.18:
// cap < 1024: удваивается (cap * 2)
// cap >= 1024: увеличивается на 25% (cap * 1.25)

// Go 1.18+: более плавный рост
// Начинает с удвоения, постепенно замедляется
```

```go
func showGrowth() {
    var s []int
    prevCap := 0
    for i := 0; i < 20; i++ {
        s = append(s, i)
        if cap(s) != prevCap {
            fmt.Printf("len=%2d cap=%2d\n", len(s), cap(s))
            prevCap = cap(s)
        }
    }
}

// Вывод:
// len= 1 cap= 1
// len= 2 cap= 2
// len= 3 cap= 4
// len= 5 cap= 8
// len= 9 cap=16
// len=17 cap=32
```

## Slicing (взятие подсреза)

### Синтаксис

```go
s[low:high]      // от low до high-1
s[low:high:max]  // трёх-индексный slice (контроль capacity)
```

### Примеры

```go
arr := [8]int{0, 1, 2, 3, 4, 5, 6, 7}

s1 := arr[2:5]
// s1 = [2, 3, 4]
// len = 3, cap = 6 (до конца массива)

s2 := arr[2:5:6]
// s2 = [2, 3, 4]
// len = 3, cap = 4 (ограничено max=6)
```

```
Original array:
┌────┬────┬────┬────┬────┬────┬────┬────┐
│  0 │  1 │  2 │  3 │  4 │  5 │  6 │  7 │
└────┴────┴────┴────┴────┴────┴────┴────┘
  0    1    2    3    4    5    6    7

arr[2:5]:
              ┌────┬────┬────┐
              │  2 │  3 │  4 │ len=3
              └────┴────┴────┘
              ▲              cap=6 (до конца)
              │
           array

arr[2:5:6]:
              ┌────┬────┬────┐
              │  2 │  3 │  4 │ len=3
              └────┴────┴────┘
              ▲         cap=4 (ограничено)
              │
           array
```

## Sharing и копирование

### Проблема: общий underlying array

```go
original := []int{1, 2, 3, 4, 5}
slice := original[1:4]  // [2, 3, 4]

// Изменение slice влияет на original!
slice[0] = 999

fmt.Println(original)  // [1, 999, 3, 4, 5]
fmt.Println(slice)     // [999, 3, 4]
```

```
До изменения:
original: ──▶ [1, 2, 3, 4, 5]
                  ▲
slice: ───────────┘ (указывает на тот же массив)

После slice[0] = 999:
original: ──▶ [1, 999, 3, 4, 5]
                  ▲
slice: ───────────┘
```

### Решение: явное копирование

```go
original := []int{1, 2, 3, 4, 5}

// Способ 1: copy
slice := make([]int, 3)
copy(slice, original[1:4])

// Способ 2: append к nil slice
slice := append([]int(nil), original[1:4]...)

// Способ 3: full slice expression + append
slice := append(original[1:4:4], []int{}...)

// Теперь изменения независимы
slice[0] = 999
fmt.Println(original)  // [1, 2, 3, 4, 5] — не изменился
```

### Функция copy

```go
func copy(dst, src []T) int  // возвращает количество скопированных элементов

src := []int{1, 2, 3, 4, 5}
dst := make([]int, 3)

n := copy(dst, src)  // n = 3 (min(len(dst), len(src)))
fmt.Println(dst)     // [1, 2, 3]

// copy копирует min(len(dst), len(src)) элементов
```

## Подводные камни

### 1. Append может изменить исходный slice

```go
s1 := make([]int, 3, 6)  // len=3, cap=6
s1[0], s1[1], s1[2] = 1, 2, 3

s2 := append(s1, 4)  // s2 использует тот же underlying array!

s2[0] = 999
fmt.Println(s1)  // [999, 2, 3] — s1 тоже изменился!
```

**Решение:** Используйте full slice expression

```go
s2 := append(s1[:len(s1):len(s1)], 4)  // cap = len, форсирует реаллокацию
```

### 2. Утечка памяти при slicing

```go
func getFirstTwo(data []byte) []byte {
    return data[:2]  // ❌ Держит ссылку на весь data!
}

// Если data = 1GB, то 1GB не освободится
```

**Решение:** Копируйте

```go
func getFirstTwo(data []byte) []byte {
    result := make([]byte, 2)
    copy(result, data[:2])
    return result  // ✅ Только 2 байта
}
```

### 3. Передача slice в функцию

```go
func modify(s []int) {
    s[0] = 999      // ✅ Изменит оригинал (тот же underlying array)
    s = append(s, 4) // ❌ НЕ изменит оригинал (копия header)
}

original := []int{1, 2, 3}
modify(original)
fmt.Println(original)  // [999, 2, 3] — длина не изменилась!
```

**Решение:** Возвращайте slice или передавайте указатель

```go
// Вариант 1: возвращать
func modify(s []int) []int {
    s[0] = 999
    s = append(s, 4)
    return s
}
original = modify(original)

// Вариант 2: указатель на slice
func modify(s *[]int) {
    (*s)[0] = 999
    *s = append(*s, 4)
}
modify(&original)
```

### 4. Range и модификация

```go
s := []int{1, 2, 3}

// ❌ Опасно: модификация во время итерации
for i, v := range s {
    if v == 2 {
        s = append(s, 4)  // Может вызвать реаллокацию!
    }
}

// Range работает с копией header, но может быть неожиданное поведение
```

## Производительность

### Предвыделение capacity

```go
// ❌ Много реаллокаций
func bad(n int) []int {
    var s []int
    for i := 0; i < n; i++ {
        s = append(s, i)  // ~log2(n) реаллокаций
    }
    return s
}

// ✅ Одна аллокация
func good(n int) []int {
    s := make([]int, 0, n)  // заранее известен размер
    for i := 0; i < n; i++ {
        s = append(s, i)  // 0 реаллокаций
    }
    return s
}
```

### Benchmark

```go
func BenchmarkWithoutCap(b *testing.B) {
    for i := 0; i < b.N; i++ {
        var s []int
        for j := 0; j < 10000; j++ {
            s = append(s, j)
        }
    }
}

func BenchmarkWithCap(b *testing.B) {
    for i := 0; i < b.N; i++ {
        s := make([]int, 0, 10000)
        for j := 0; j < 10000; j++ {
            s = append(s, j)
        }
    }
}

// Результат:
// BenchmarkWithoutCap   5000   300000 ns/op   386000 B/op   20 allocs
// BenchmarkWithCap     20000    80000 ns/op    80000 B/op    1 allocs
```

## Вопросы с собеседований

### Вопрос 1: Из чего состоит slice под капотом?

<details>
<summary>Ответ</summary>

Slice — это структура из трёх полей (24 байта на 64-bit системе):
- `array` — указатель на underlying array
- `len` — текущая длина
- `cap` — ёмкость (сколько элементов может вместить без реаллокации)

```go
type slice struct {
    array unsafe.Pointer
    len   int
    cap   int
}
```

</details>

### Вопрос 2: В чём разница между nil slice и empty slice?

<details>
<summary>Ответ</summary>

```go
var nilSlice []int     // nil slice: array=nil, len=0, cap=0
emptySlice := []int{}  // empty slice: array≠nil, len=0, cap=0
```

- `nilSlice == nil` → true
- `emptySlice == nil` → false
- `len()` и `cap()` одинаковые для обоих
- JSON: nil → `null`, empty → `[]`

На практике ведут себя одинаково для append, range и т.д.

</details>

### Вопрос 3: Что произойдёт при append, если capacity исчерпана?

<details>
<summary>Ответ</summary>

1. Создаётся новый underlying array с большей capacity
2. Данные копируются из старого массива
3. Добавляются новые элементы
4. Возвращается slice с новым указателем

Стратегия роста: примерно удвоение для маленьких slice, постепенное замедление для больших.

**Важно:** старый slice продолжает указывать на старый массив!

</details>

### Вопрос 4: Почему изменение sub-slice влияет на оригинал?

<details>
<summary>Ответ</summary>

Потому что sub-slice и оригинал **делят один underlying array**:

```go
original := []int{1, 2, 3, 4, 5}
sub := original[1:4]  // указывает на ту же память

sub[0] = 999
fmt.Println(original)  // [1, 999, 3, 4, 5]
```

Чтобы избежать — используйте `copy()` или full slice expression `s[low:high:max]`.

</details>

### Вопрос 5: Как передать slice в функцию, чтобы append внутри функции изменил оригинал?

<details>
<summary>Ответ</summary>

Slice передаётся по значению (копия header). Append внутри функции может изменить len локальной копии, но не оригинала.

Два решения:

```go
// 1. Возвращать slice
func add(s []int, v int) []int {
    return append(s, v)
}
s = add(s, 42)

// 2. Передавать указатель на slice
func add(s *[]int, v int) {
    *s = append(*s, v)
}
add(&s, 42)
```

</details>

### Вопрос 6: Как избежать утечки памяти при slicing большого slice?

<details>
<summary>Ответ</summary>

Sub-slice держит ссылку на весь underlying array. Решение — копировать:

```go
// ❌ Утечка: держит весь bigData
func bad(bigData []byte) []byte {
    return bigData[:100]
}

// ✅ Копируем только нужное
func good(bigData []byte) []byte {
    result := make([]byte, 100)
    copy(result, bigData[:100])
    return result
}
```

</details>

## Связанные темы

- [[Go - Массивы и слайсы]]
- [[Go - Escape Analysis]]
- [[Go - Управление памятью]]
- [[Go - Внутреннее устройство map]]
- [[Go - Указатели и ссылки]]
