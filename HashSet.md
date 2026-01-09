# HashSet

Множество (Set) - коллекция уникальных элементов без определенного порядка.

## Концепция

**Set** хранит только ключи (без значений), гарантирует уникальность.

```
Set: {1, 2, 3, 4, 5}
Операции: Add, Remove, Contains
```

## Реализация через HashMap

### В Go нет встроенного Set

```go
// Используем map[T]bool
type IntSet map[int]bool

func NewIntSet() IntSet {
    return make(IntSet)
}

func (s IntSet) Add(value int) {
    s[value] = true
}

func (s IntSet) Remove(value int) {
    delete(s, value)
}

func (s IntSet) Contains(value int) bool {
    return s[value]
}

func (s IntSet) Size() int {
    return len(s)
}

func (s IntSet) Values() []int {
    values := make([]int, 0, len(s))
    for v := range s {
        values = append(values, v)
    }
    return values
}
```

### Оптимизация памяти: map[T]struct{}

```go
// struct{} занимает 0 байт!
type IntSet map[int]struct{}

func (s IntSet) Add(value int) {
    s[value] = struct{}{}  // Пустая структура
}

func (s IntSet) Contains(value int) bool {
    _, ok := s[value]
    return ok
}
```

**Почему struct{}?**
- `bool` занимает 1 байт
- `struct{}` занимает 0 байт
- Для 1,000,000 элементов экономия ~1 MB

## Обобщенная реализация (Go 1.18+)

```go
type Set[T comparable] map[T]struct{}

func NewSet[T comparable]() Set[T] {
    return make(Set[T])
}

func (s Set[T]) Add(value T) {
    s[value] = struct{}{}
}

func (s Set[T]) Remove(value T) {
    delete(s, value)
}

func (s Set[T]) Contains(value T) bool {
    _, ok := s[value]
    return ok
}

func (s Set[T]) Size() int {
    return len(s)
}

func (s Set[T]) Clear() {
    for k := range s {
        delete(s, k)
    }
}

func (s Set[T]) Values() []T {
    values := make([]T, 0, len(s))
    for v := range s {
        values = append(values, v)
    }
    return values
}

// Использование
intSet := NewSet[int]()
intSet.Add(1)
intSet.Add(2)

stringSet := NewSet[string]()
stringSet.Add("hello")
```

## Операции над множествами

### Union (объединение)

```go
func (s Set[T]) Union(other Set[T]) Set[T] {
    result := NewSet[T]()

    for v := range s {
        result.Add(v)
    }
    for v := range other {
        result.Add(v)
    }

    return result
}

// {1,2,3} ∪ {3,4,5} = {1,2,3,4,5}
```

### Intersection (пересечение)

```go
func (s Set[T]) Intersection(other Set[T]) Set[T] {
    result := NewSet[T]()

    // Итерируем по меньшему множеству
    smaller, larger := s, other
    if len(other) < len(s) {
        smaller, larger = other, s
    }

    for v := range smaller {
        if larger.Contains(v) {
            result.Add(v)
        }
    }

    return result
}

// {1,2,3} ∩ {2,3,4} = {2,3}
```

### Difference (разность)

```go
func (s Set[T]) Difference(other Set[T]) Set[T] {
    result := NewSet[T]()

    for v := range s {
        if !other.Contains(v) {
            result.Add(v)
        }
    }

    return result
}

// {1,2,3} - {2,3,4} = {1}
```

### Symmetric Difference (симметричная разность)

```go
func (s Set[T]) SymmetricDifference(other Set[T]) Set[T] {
    result := NewSet[T]()

    for v := range s {
        if !other.Contains(v) {
            result.Add(v)
        }
    }

    for v := range other {
        if !s.Contains(v) {
            result.Add(v)
        }
    }

    return result
}

// {1,2,3} △ {2,3,4} = {1,4}
```

### Subset и Superset

```go
func (s Set[T]) IsSubset(other Set[T]) bool {
    if len(s) > len(other) {
        return false
    }

    for v := range s {
        if !other.Contains(v) {
            return false
        }
    }

    return true
}

func (s Set[T]) IsSuperset(other Set[T]) bool {
    return other.IsSubset(s)
}

// {1,2} ⊆ {1,2,3} → true
// {1,2,3} ⊇ {1,2} → true
```

## Сложность операций

| Операция | Сложность |
|----------|-----------|
| Add | O(1)* |
| Remove | O(1)* |
| Contains | O(1)* |
| Size | O(1) |
| Union | O(n + m) |
| Intersection | O(min(n, m)) |
| Difference | O(n) |

\* Амортизированное

## Применение

### 1. Удаление дубликатов

```go
func removeDuplicates(nums []int) []int {
    set := NewSet[int]()
    for _, num := range nums {
        set.Add(num)
    }
    return set.Values()
}

// removeDuplicates([1,2,2,3,3,3]) → [1,2,3]
```

### 2. Проверка уникальности

```go
func hasUniqueChars(s string) bool {
    set := NewSet[rune]()
    for _, ch := range s {
        if set.Contains(ch) {
            return false
        }
        set.Add(ch)
    }
    return true
}

// hasUniqueChars("abcdef") → true
// hasUniqueChars("hello") → false (два 'l')
```

### 3. Intersection of Two Arrays

```go
func intersection(nums1, nums2 []int) []int {
    set1 := NewSet[int]()
    for _, num := range nums1 {
        set1.Add(num)
    }

    result := NewSet[int]()
    for _, num := range nums2 {
        if set1.Contains(num) {
            result.Add(num)
        }
    }

    return result.Values()
}

// intersection([1,2,2,1], [2,2]) → [2]
```

### 4. Happy Number (LeetCode)

```go
func isHappy(n int) bool {
    seen := NewSet[int]()

    for n != 1 {
        if seen.Contains(n) {
            return false  // Цикл обнаружен
        }
        seen.Add(n)
        n = sumOfSquares(n)
    }

    return true
}

func sumOfSquares(n int) int {
    sum := 0
    for n > 0 {
        digit := n % 10
        sum += digit * digit
        n /= 10
    }
    return sum
}

// isHappy(19) → true: 19 → 82 → 68 → 100 → 1
```

### 5. Visited Set (графы)

```go
func bfs(graph map[int][]int, start int) []int {
    visited := NewSet[int]()
    queue := []int{start}
    result := []int{}

    for len(queue) > 0 {
        node := queue[0]
        queue = queue[1:]

        if visited.Contains(node) {
            continue
        }

        visited.Add(node)
        result = append(result, node)

        for _, neighbor := range graph[node] {
            if !visited.Contains(neighbor) {
                queue = append(queue, neighbor)
            }
        }
    }

    return result
}
```

## Set vs Slice vs Array

| Операция | Set | Slice | Array |
|----------|-----|-------|-------|
| Add | O(1) | O(1)* | Fixed |
| Contains | O(1) | O(n) | O(n) |
| Remove | O(1) | O(n) | O(n) |
| Уникальность | ✅ | ❌ | ❌ |
| Порядок | ❌ | ✅ | ✅ |

\* Амортизированное для append

## Ordered Set (упорядоченное множество)

Go не имеет встроенного ordered set, но можно реализовать через:

### 1. Map + Slice

```go
type OrderedSet[T comparable] struct {
    m     map[T]struct{}
    order []T
}

func (os *OrderedSet[T]) Add(value T) {
    if _, ok := os.m[value]; !ok {
        os.m[value] = struct{}{}
        os.order = append(os.order, value)
    }
}

func (os *OrderedSet[T]) Values() []T {
    return os.order
}
```

**Проблема:** Remove будет O(n)

### 2. Используйте container/list + map

```go
import "container/list"

type OrderedSet[T comparable] struct {
    m map[T]*list.Element
    l *list.List
}

func NewOrderedSet[T comparable]() *OrderedSet[T] {
    return &OrderedSet[T]{
        m: make(map[T]*list.Element),
        l: list.New(),
    }
}

func (os *OrderedSet[T]) Add(value T) {
    if _, ok := os.m[value]; !ok {
        elem := os.l.PushBack(value)
        os.m[value] = elem
    }
}

func (os *OrderedSet[T]) Remove(value T) {
    if elem, ok := os.m[value]; ok {
        os.l.Remove(elem)
        delete(os.m, value)
    }
}
```

## BitSet для малых значений

Для множеств целых чисел 0-63:

```go
type BitSet uint64

func (bs *BitSet) Add(n uint) {
    *bs |= 1 << n
}

func (bs *BitSet) Remove(n uint) {
    *bs &^= 1 << n
}

func (bs *BitSet) Contains(n uint) bool {
    return (*bs & (1 << n)) != 0
}

// Пример
var bs BitSet
bs.Add(5)
bs.Add(10)
fmt.Println(bs.Contains(5))   // true
fmt.Println(bs.Contains(7))   // false
```

**Преимущества:**
- Очень быстро (битовые операции)
- Минимум памяти (8 байт для 0-63)

**Недостатки:**
- Только для малых чисел
- Не для произвольных типов

## Best Practices

1. ✅ Используйте `map[T]struct{}` вместо `map[T]bool`
2. ✅ Pre-allocate с make(map[T]struct{}, capacity)
3. ✅ Для чисел 0-63 рассмотрите BitSet
4. ✅ Для упорядоченности используйте OrderedSet
5. ✅ Generics для type-safe множеств
6. ❌ Не полагайтесь на порядок итерации
7. ❌ Помните о concurrent access (нужен мьютекс)

## Сравнение с другими языками

```python
# Python
s = {1, 2, 3}
s.add(4)
s.remove(2)
2 in s  # False
```

```javascript
// JavaScript
const s = new Set([1, 2, 3]);
s.add(4);
s.delete(2);
s.has(2);  // false
```

```go
// Go
s := NewSet[int]()
s.Add(1)
s.Add(2)
s.Add(3)
s.Add(4)
s.Remove(2)
s.Contains(2)  // false
```

## Связанные темы

- [[HashMap - Реализация и особенности]]
- [[Go - Карты (maps)]]
- [[Алгоритмическая сложность (Big O)]]
- [[Задача - Intersection of Two Arrays]]
- [[BFS и DFS]]
