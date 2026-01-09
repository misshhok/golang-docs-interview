# HashMap - Реализация и особенности

Хеш-таблица (HashMap) - структура данных для хранения пар ключ-значение с O(1) доступом.

## Принцип работы

```
Ключ → Hash Function → Index в массиве → Значение

"name" → hash("name") → 42 → "John"
```

### Хеш-функция

```go
func hash(key string) int {
    h := 0
    for _, ch := range key {
        h = 31*h + int(ch)
    }
    return h
}

// В реальности используется более сложная функция
```

## Коллизии

Когда разные ключи дают одинаковый хеш.

### 1. Метод цепочек (Chaining)

```go
type Entry struct {
    key   string
    value int
    next  *Entry  // Связный список
}

type HashMap struct {
    buckets []*Entry
    size    int
}

func (hm *HashMap) Put(key string, value int) {
    index := hash(key) % len(hm.buckets)

    // Проверить существующие записи
    for entry := hm.buckets[index]; entry != nil; entry = entry.next {
        if entry.key == key {
            entry.value = value  // Обновить
            return
        }
    }

    // Добавить новую запись
    newEntry := &Entry{
        key:   key,
        value: value,
        next:  hm.buckets[index],
    }
    hm.buckets[index] = newEntry
    hm.size++

    // Проверить load factor
    if float64(hm.size)/float64(len(hm.buckets)) > 0.75 {
        hm.resize()
    }
}

func (hm *HashMap) Get(key string) (int, bool) {
    index := hash(key) % len(hm.buckets)

    for entry := hm.buckets[index]; entry != nil; entry = entry.next {
        if entry.key == key {
            return entry.value, true
        }
    }

    return 0, false
}

func (hm *HashMap) Delete(key string) bool {
    index := hash(key) % len(hm.buckets)

    if hm.buckets[index] == nil {
        return false
    }

    // Удаление первого элемента
    if hm.buckets[index].key == key {
        hm.buckets[index] = hm.buckets[index].next
        hm.size--
        return true
    }

    // Поиск в цепочке
    prev := hm.buckets[index]
    for entry := prev.next; entry != nil; entry = entry.next {
        if entry.key == key {
            prev.next = entry.next
            hm.size--
            return true
        }
        prev = entry
    }

    return false
}
```

### 2. Открытая адресация (Open Addressing)

```go
type HashMapOpen struct {
    keys   []string
    values []int
    size   int
}

// Linear Probing
func (hm *HashMapOpen) Put(key string, value int) {
    index := hash(key) % len(hm.keys)

    // Найти свободное место
    for hm.keys[index] != "" && hm.keys[index] != key {
        index = (index + 1) % len(hm.keys)
    }

    if hm.keys[index] == "" {
        hm.size++
    }

    hm.keys[index] = key
    hm.values[index] = value
}

// Quadratic Probing
func quadraticProbe(index, i, size int) int {
    return (index + i*i) % size
}

// Double Hashing
func doubleHash(key string, i, size int) int {
    h1 := hash(key) % size
    h2 := 1 + (hash(key) % (size - 1))
    return (h1 + i*h2) % size
}
```

## Load Factor и Resize

**Load Factor** = количество элементов / размер массива

```go
func (hm *HashMap) resize() {
    oldBuckets := hm.buckets
    hm.buckets = make([]*Entry, len(oldBuckets)*2)
    hm.size = 0

    // Перехешировать все элементы
    for _, entry := range oldBuckets {
        for e := entry; e != nil; e = e.next {
            hm.Put(e.key, e.value)
        }
    }
}
```

**Когда делать resize:**
- Load factor > 0.75 (рекомендуется)
- В Go map: загрузка ~6.5 элементов на bucket

## Сложность операций

| Операция | Best Case | Average Case | Worst Case |
|----------|-----------|--------------|------------|
| Get | O(1) | O(1) | O(n) |
| Put | O(1) | O(1) | O(n) |
| Delete | O(1) | O(1) | O(n) |

**Worst case:** все ключи в одном bucket (плохая хеш-функция)

## Go map

### Внутреннее устройство

```go
// Упрощенная структура runtime map
type hmap struct {
    count     int       // количество элементов
    B         uint8     // log₂(количество buckets)
    buckets   unsafe.Pointer
    oldbuckets unsafe.Pointer  // для постепенного resize
}

type bmap struct {
    tophash [8]uint8      // Старшие 8 бит хеша
    keys    [8]keytype
    values  [8]valuetype
    overflow *bmap        // Указатель на overflow bucket
}
```

### Особенности Go map

```go
// Создание
m := make(map[string]int)
m := make(map[string]int, 100)  // С capacity

// Вставка
m["key"] = 42

// Получение
value := m["key"]           // 0 если нет
value, ok := m["key"]       // Проверка существования

// Удаление
delete(m, "key")

// Итерация (неупорядоченная!)
for key, value := range m {
    fmt.Println(key, value)
}
```

### Нулевые значения

```go
m := make(map[string]int)

// Получение несуществующего ключа → zero value
value := m["nonexistent"]  // 0

// Как отличить?
value, ok := m["key"]
if ok {
    fmt.Println("Найден:", value)
} else {
    fmt.Println("Не найден")
}
```

### Map не потокобезопасна!

```go
// ❌ Race condition
m := make(map[string]int)

go func() {
    m["key"] = 1
}()

go func() {
    m["key"] = 2  // Race!
}()

// ✅ Используйте sync.Map или мьютекс
type SafeMap struct {
    mu sync.RWMutex
    m  map[string]int
}

func (sm *SafeMap) Get(key string) (int, bool) {
    sm.mu.RLock()
    defer sm.mu.RUnlock()
    val, ok := sm.m[key]
    return val, ok
}

func (sm *SafeMap) Set(key string, value int) {
    sm.mu.Lock()
    defer sm.mu.Unlock()
    sm.m[key] = value
}
```

### sync.Map

```go
var m sync.Map

// Store
m.Store("key", 42)

// Load
value, ok := m.Load("key")

// LoadOrStore (атомарно)
actual, loaded := m.LoadOrStore("key", 42)

// Delete
m.Delete("key")

// Range
m.Range(func(key, value interface{}) bool {
    fmt.Println(key, value)
    return true  // продолжить итерацию
})
```

**Когда использовать sync.Map:**
- Много чтений, мало записей
- Непересекающиеся множества ключей в разных горутинах

## Хеш-функции

### Свойства хорошей хеш-функции

1. **Детерминированность** - одинаковый вход → одинаковый хеш
2. **Равномерность** - распределение по всему диапазону
3. **Лавинный эффект** - малое изменение входа → большое изменение хеша

### FNV-1a (используется в Go)

```go
func fnv1a(data []byte) uint32 {
    hash := uint32(2166136261)
    for _, b := range data {
        hash ^= uint32(b)
        hash *= 16777619
    }
    return hash
}
```

## Применение

### 1. Кэш

```go
type Cache struct {
    data map[string]interface{}
    mu   sync.RWMutex
}

func (c *Cache) Get(key string) (interface{}, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    val, ok := c.data[key]
    return val, ok
}

func (c *Cache) Set(key string, value interface{}) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.data[key] = value
}
```

### 2. Подсчет частоты

```go
func countFrequency(nums []int) map[int]int {
    freq := make(map[int]int)
    for _, num := range nums {
        freq[num]++
    }
    return freq
}

// countFrequency([1,2,2,3,3,3]) → {1:1, 2:2, 3:3}
```

### 3. Группировка

```go
func groupAnagrams(strs []string) [][]string {
    groups := make(map[string][]string)

    for _, str := range strs {
        // Сортировка как ключ
        key := sortString(str)
        groups[key] = append(groups[key], str)
    }

    result := [][]string{}
    for _, group := range groups {
        result = append(result, group)
    }

    return result
}

// groupAnagrams(["eat","tea","tan","ate","nat","bat"])
// → [["eat","tea","ate"],["tan","nat"],["bat"]]
```

### 4. Two Sum

```go
func twoSum(nums []int, target int) []int {
    seen := make(map[int]int)  // value → index

    for i, num := range nums {
        complement := target - num
        if j, ok := seen[complement]; ok {
            return []int{j, i}
        }
        seen[num] = i
    }

    return nil
}

// twoSum([2,7,11,15], 9) → [0,1]
```

## Сравнение с другими структурами

| Структура | Поиск | Вставка | Удаление | Порядок |
|-----------|-------|---------|----------|---------|
| HashMap | O(1)* | O(1)* | O(1)* | Нет |
| Array | O(n) | O(n) | O(n) | Индекс |
| Sorted Array | O(log n) | O(n) | O(n) | Сортировка |
| BST | O(log n) | O(log n) | O(log n) | Сортировка |

\* Амортизированное

## Best Practices

1. ✅ Pre-allocate с make(map[K]V, capacity)
2. ✅ Используйте sync.Map для concurrent доступа
3. ✅ Проверяйте существование через `value, ok := m[key]`
4. ✅ delete() безопасна для несуществующих ключей
5. ✅ Итерация по map неупорядоченная (намеренно!)
6. ❌ Не полагайтесь на порядок итерации
7. ❌ Map не сравнима с ==, используйте reflect.DeepEqual
8. ❌ Не храните указатели на элементы map (GC может их переместить)

## Типичные ошибки

### 1. Concurrent writes

```go
// ❌ Паника: concurrent map writes
m := make(map[int]int)
for i := 0; i < 100; i++ {
    go func(n int) {
        m[n] = n
    }(i)
}
```

### 2. Порядок итерации

```go
// ❌ Порядок может меняться!
m := map[string]int{"a": 1, "b": 2, "c": 3}
for k := range m {
    fmt.Print(k)  // Может быть: "abc", "bca", "cab", ...
}
```

### 3. Модификация во время итерации

```go
// ✅ Безопасно добавлять
for k, v := range m {
    if v == 0 {
        m[k+"_new"] = 1  // OK
    }
}

// ✅ Безопасно удалять
for k, v := range m {
    if v == 0 {
        delete(m, k)  // OK
    }
}
```

## Связанные темы

- [[HashSet]]
- [[Алгоритмическая сложность (Big O)]]
- [[Амортизированная сложность]]
- [[Go - Карты (maps)]]
- [[Go - Пакет sync]]
- [[Задача - Two Sum]]
