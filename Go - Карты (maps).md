# Go - Карты (maps)

Карта (map) - встроенная структура данных для хранения пар ключ-значение. Реализует hash table (хеш-таблицу).

## Объявление и создание

### Объявление

```go
// Nil map
var m map[string]int // nil, нельзя использовать!

// С make
m := make(map[string]int) // Пустая map, можно использовать

// С начальной емкостью (hint)
m := make(map[string]int, 100) // Оптимизация

// Литерал
m := map[string]int{
    "one":   1,
    "two":   2,
    "three": 3,
}

// Пустой литерал
m := map[string]int{} // НЕ nil
```

**Важно:** nil map нельзя использовать для записи!

```go
var m map[string]int // nil
m["key"] = 42        // ❌ panic: assignment to entry in nil map

m = make(map[string]int) // ✅ Инициализируем
m["key"] = 42        // ✅ Работает
```

## Операции

### Добавление и обновление

```go
m := make(map[string]int)

// Добавление
m["one"] = 1
m["two"] = 2

// Обновление (перезапись)
m["one"] = 100
```

### Чтение

```go
// Простое чтение
value := m["one"] // Возвращает zero value если ключа нет

// С проверкой существования
value, exists := m["one"]
if exists {
    fmt.Println("Найдено:", value)
} else {
    fmt.Println("Ключ не найден")
}

// Идиома проверки
if value, ok := m["key"]; ok {
    fmt.Println("Значение:", value)
}
```

**Zero value при отсутствии ключа:**
```go
m := make(map[string]int)
value := m["missing"] // value = 0 (zero value для int)

// Нельзя отличить "ключ отсутствует" от "значение = 0"
// Используйте two-value assignment!
value, ok := m["missing"]
if !ok {
    fmt.Println("Ключа нет")
}
```

### Удаление

```go
m := map[string]int{
    "one": 1,
    "two": 2,
}

// Удаление ключа
delete(m, "one")

// Удаление несуществующего ключа - безопасно, не panic
delete(m, "missing") // ✅ Ничего не происходит
```

### Длина

```go
m := map[string]int{"a": 1, "b": 2, "c": 3}

len(m) // 3

// Пустая map
empty := make(map[string]int)
len(empty) // 0

// Nil map
var nilMap map[string]int
len(nilMap) // 0
```

## Итерация

### Базовая итерация

```go
m := map[string]int{
    "one":   1,
    "two":   2,
    "three": 3,
}

// Ключ и значение
for key, value := range m {
    fmt.Printf("%s: %d\n", key, value)
}

// Только ключи
for key := range m {
    fmt.Println(key)
}

// Только значения
for _, value := range m {
    fmt.Println(value)
}
```

**⚠️ Порядок итерации НЕ гарантирован и случаен!**

```go
m := map[string]int{"a": 1, "b": 2, "c": 3}

// Может быть: a, b, c
// Или: c, a, b
// Или любой другой порядок!
for key := range m {
    fmt.Println(key)
}
```

### Сортированная итерация

```go
m := map[string]int{
    "charlie": 3,
    "alice":   1,
    "bob":     2,
}

// Собираем ключи
keys := make([]string, 0, len(m))
for key := range m {
    keys = append(keys, key)
}

// Сортируем
sort.Strings(keys)

// Итерируемся по отсортированным ключам
for _, key := range keys {
    fmt.Printf("%s: %d\n", key, m[key])
}
// Вывод:
// alice: 1
// bob: 2
// charlie: 3
```

## Типы ключей

### Допустимые типы

Ключ должен быть **comparable** (сравниваемым):

```go
// ✅ Допустимые типы ключей
map[int]string
map[string]int
map[float64]bool
map[bool]string
map[rune]int

// Структуры (если все поля comparable)
type Point struct {
    X, Y int
}
map[Point]string // ✅

// Массивы
map[[3]int]string // ✅
```

### Недопустимые типы

```go
// ❌ Нельзя использовать
map[[]int]string      // slice - not comparable
map[map[string]int]int // map - not comparable

// Структура с не-comparable полями
type Data struct {
    Slice []int
}
map[Data]string // ❌ Ошибка компиляции
```

**Обходной путь:** используйте строковое представление

```go
// Вместо map[[]int]string
m := make(map[string]string)
key := fmt.Sprintf("%v", []int{1, 2, 3})
m[key] = "value"
```

## Map - reference type

```go
func modify(m map[string]int) {
    m["key"] = 100 // Изменяет оригинальную map!
}

original := map[string]int{"key": 1}
modify(original)
fmt.Println(original["key"]) // 100
```

**Копирование map:**

```go
// ❌ Нельзя просто присвоить
original := map[string]int{"a": 1, "b": 2}
copy := original // Обе переменные указывают на одну map!

copy["a"] = 100
fmt.Println(original["a"]) // 100 (изменилось!)

// ✅ Ручное копирование
original := map[string]int{"a": 1, "b": 2}
copy := make(map[string]int)
for key, value := range original {
    copy[key] = value
}

copy["a"] = 100
fmt.Println(original["a"]) // 1 (не изменилось)
```

## Многопоточность

**Map НЕ потокобезопасна!**

```go
// ❌ Race condition
m := make(map[int]int)

go func() {
    m[1] = 1 // Запись
}()

go func() {
    _ = m[1] // Чтение
}()
// Concurrent map read and write panic!
```

### Решение 1: sync.Mutex

```go
type SafeMap struct {
    mu sync.Mutex
    m  map[string]int
}

func (sm *SafeMap) Set(key string, value int) {
    sm.mu.Lock()
    defer sm.mu.Unlock()
    sm.m[key] = value
}

func (sm *SafeMap) Get(key string) (int, bool) {
    sm.mu.Lock()
    defer sm.mu.Unlock()
    value, ok := sm.m[key]
    return value, ok
}
```

### Решение 2: sync.RWMutex

```go
type SafeMap struct {
    mu sync.RWMutex
    m  map[string]int
}

func (sm *SafeMap) Set(key string, value int) {
    sm.mu.Lock()
    defer sm.mu.Unlock()
    sm.m[key] = value
}

func (sm *SafeMap) Get(key string) (int, bool) {
    sm.mu.RLock() // Только читаем
    defer sm.mu.RUnlock()
    value, ok := sm.m[key]
    return value, ok
}
```

### Решение 3: sync.Map

Встроенная потокобезопасная map для специальных случаев:

```go
var m sync.Map

// Запись
m.Store("key", 42)

// Чтение
value, ok := m.Load("key")
if ok {
    fmt.Println(value.(int))
}

// Удаление
m.Delete("key")

// Load or Store
actual, loaded := m.LoadOrStore("key", 42)
// loaded = true если ключ уже был

// Итерация
m.Range(func(key, value interface{}) bool {
    fmt.Printf("%v: %v\n", key, value)
    return true // продолжаем итерацию
})
```

**Когда использовать sync.Map:**
- Ключи записываются один раз, читаются многократно
- Несколько горутин читают/пишут непересекающиеся ключи
- Не подходит для частых обновлений одних и тех же ключей

## Паттерны

### Set (множество)

```go
// Map как Set
set := make(map[string]bool)

// Добавление
set["apple"] = true
set["banana"] = true

// Проверка наличия
if set["apple"] {
    fmt.Println("Есть apple")
}

// Удаление
delete(set, "apple")

// Итерация
for item := range set {
    fmt.Println(item)
}

// Более memory-efficient вариант
set := make(map[string]struct{})
set["apple"] = struct{}{} // struct{} не занимает памяти

if _, exists := set["apple"]; exists {
    fmt.Println("Есть apple")
}
```

### Группировка

```go
// Группировка элементов по ключу
type Person struct {
    Name string
    Age  int
}

people := []Person{
    {"Alice", 25},
    {"Bob", 25},
    {"Charlie", 30},
}

// Группируем по возрасту
byAge := make(map[int][]Person)
for _, person := range people {
    byAge[person.Age] = append(byAge[person.Age], person)
}

// byAge[25] = [Alice, Bob]
// byAge[30] = [Charlie]
```

### Подсчет частоты

```go
words := []string{"apple", "banana", "apple", "cherry", "banana", "apple"}

// Подсчет вхождений
frequency := make(map[string]int)
for _, word := range words {
    frequency[word]++
}

// frequency["apple"] = 3
// frequency["banana"] = 2
// frequency["cherry"] = 1
```

### Кэш/Мемоизация

```go
// Кэш вычислений
cache := make(map[int]int)

func fibonacci(n int) int {
    if n <= 1 {
        return n
    }

    // Проверяем кэш
    if result, ok := cache[n]; ok {
        return result
    }

    // Вычисляем и сохраняем
    result := fibonacci(n-1) + fibonacci(n-2)
    cache[n] = result
    return result
}
```

### Инвертирование map

```go
original := map[string]int{
    "one":   1,
    "two":   2,
    "three": 3,
}

// Инвертируем: ключи ⇄ значения
inverted := make(map[int]string)
for key, value := range original {
    inverted[value] = key
}

// inverted[1] = "one"
// inverted[2] = "two"
```

## Производительность

### Выделение памяти

```go
// ❌ Без hint - многократные перевыделения
m := make(map[string]int)
for i := 0; i < 10000; i++ {
    m[fmt.Sprintf("key%d", i)] = i
}

// ✅ С hint - одно выделение
m := make(map[string]int, 10000)
for i := 0; i < 10000; i++ {
    m[fmt.Sprintf("key%d", i)] = i
}
```

### Сложность операций

- **Вставка**: O(1) средняя, O(n) худшая (при коллизиях)
- **Поиск**: O(1) средняя, O(n) худшая
- **Удаление**: O(1) средняя, O(n) худшая
- **Итерация**: O(n)

Подробнее: [[Алгоритмическая сложность (Big O)]]

## Частые ошибки

### 1. Использование nil map

```go
// ❌ panic
var m map[string]int
m["key"] = 42 // panic: assignment to entry in nil map

// ✅ Инициализируем
m = make(map[string]int)
m["key"] = 42
```

### 2. Игнорирование порядка итерации

```go
// ❌ Надежда на порядок
m := map[string]int{"a": 1, "b": 2, "c": 3}
for key := range m {
    // Порядок НЕ гарантирован!
}

// ✅ Явная сортировка
keys := make([]string, 0, len(m))
for key := range m {
    keys = append(keys, key)
}
sort.Strings(keys)
```

### 3. Concurrent access без синхронизации

```go
// ❌ Race condition
m := make(map[int]int)
go func() { m[1] = 1 }()
go func() { _ = m[1] }()
// fatal error: concurrent map read and map write

// ✅ С мьютексом
var mu sync.RWMutex
m := make(map[int]int)
go func() {
    mu.Lock()
    m[1] = 1
    mu.Unlock()
}()
go func() {
    mu.RLock()
    _ = m[1]
    mu.RUnlock()
}()
```

### 4. Модификация во время итерации

```go
// ✅ Удаление безопасно
m := map[string]int{"a": 1, "b": 2, "c": 3}
for key := range m {
    if key == "b" {
        delete(m, key) // Безопасно
    }
}

// ✅ Добавление безопасно (но новые ключи могут не попасть в итерацию)
for key, value := range m {
    if value > 10 {
        m[key+"_copy"] = value // Безопасно
    }
}
```

## Best Practices

1. ✅ Инициализируйте map перед использованием (make или литерал)
2. ✅ Используйте hint при известном размере
3. ✅ Проверяйте существование ключа через two-value assignment
4. ✅ Не полагайтесь на порядок итерации
5. ✅ Используйте sync.RWMutex для concurrent access
6. ✅ Используйте `map[K]struct{}` для Set вместо `map[K]bool`
7. ❌ Не используйте не-comparable типы как ключи

## Связанные темы

- [[Go - Типы данных]]
- [[Go - Массивы и слайсы]]
- [[HashMap - Реализация и особенности]]
- [[Go - Mutex и RWMutex]]
- [[Go - Пакет sync]]
- [[Алгоритмическая сложность (Big O)]]
