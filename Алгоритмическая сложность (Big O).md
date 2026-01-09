# Алгоритмическая сложность (Big O)

Big O нотация описывает как растет время выполнения или использование памяти алгоритма относительно размера входных данных.

## Что такое Big O

Big O показывает **верхнюю границу** (worst case) роста функции. Мы интересуемся поведением при **больших n**.

> "Как изменится время работы если увеличить данные в 10 раз?"

## Основные классы сложности

От лучшего к худшему:

| Обозначение | Название | Пример | n=10 | n=100 | n=1000 |
|-------------|----------|--------|------|-------|--------|
| O(1) | Константная | Доступ к элементу массива по индексу | 1 | 1 | 1 |
| O(log n) | Логарифмическая | Бинарный поиск | 3 | 7 | 10 |
| O(n) | Линейная | Поиск в несортированном массиве | 10 | 100 | 1000 |
| O(n log n) | Линейно-логарифмическая | Эффективная сортировка (merge, quick) | 30 | 700 | 10000 |
| O(n²) | Квадратичная | Вложенные циклы, bubble sort | 100 | 10000 | 1000000 |
| O(n³) | Кубическая | Три вложенных цикла | 1000 | 1000000 | 1B |
| O(2ⁿ) | Экспоненциальная | Рекурсивный Fibonacci | 1024 | 1.27×10³⁰ | - |
| O(n!) | Факториальная | Генерация всех перестановок | 3.6M | - | - |

## Правила вычисления Big O

### 1. Отбрасываем константы

```go
// Два прохода по массиву
func twoPass(arr []int) {
    // O(n)
    for _, v := range arr {
        fmt.Println(v)
    }

    // O(n)
    for _, v := range arr {
        fmt.Println(v * 2)
    }
}

// Сложность: O(2n) = O(n), константа 2 отбрасывается
```

### 2. Отбрасываем младшие члены

```go
func example(n int) {
    // O(n²)
    for i := 0; i < n; i++ {
        for j := 0; j < n; j++ {
            // ...
        }
    }

    // O(n)
    for i := 0; i < n; i++ {
        // ...
    }
}

// Сложность: O(n² + n) = O(n²), член n отбрасывается
```

### 3. Разные переменные - разные обозначения

```go
func twoArrays(arr1 []int, arr2 []int) {
    // O(a)
    for _, v := range arr1 {
        fmt.Println(v)
    }

    // O(b)
    for _, v := range arr2 {
        fmt.Println(v)
    }
}

// Сложность: O(a + b), НЕ O(n)!
```

## Примеры на Go

### O(1) - Константная

```go
// Доступ по индексу
func getElement(arr []int, index int) int {
    return arr[index] // O(1)
}

// Доступ к map
func getValue(m map[string]int, key string) int {
    return m[key] // O(1) в среднем
}

// Арифметические операции
func add(a, b int) int {
    return a + b // O(1)
}
```

**Характеристика**: Время не зависит от размера данных.

### O(log n) - Логарифмическая

```go
// Бинарный поиск
func binarySearch(arr []int, target int) int {
    left, right := 0, len(arr)-1

    for left <= right {
        mid := (left + right) / 2
        if arr[mid] == target {
            return mid
        } else if arr[mid] < target {
            left = mid + 1
        } else {
            right = mid - 1
        }
    }

    return -1 // O(log n)
}
```

**Характеристика**: Каждый шаг уменьшает проблему вдвое.

Подробнее: [[Бинарный поиск]]

### O(n) - Линейная

```go
// Поиск в массиве
func linearSearch(arr []int, target int) int {
    for i, v := range arr {
        if v == target {
            return i
        }
    }
    return -1 // O(n)
}

// Сумма элементов
func sum(arr []int) int {
    total := 0
    for _, v := range arr {
        total += v
    }
    return total // O(n)
}

// Поиск максимума
func findMax(arr []int) int {
    max := arr[0]
    for _, v := range arr {
        if v > max {
            max = v
        }
    }
    return max // O(n)
}
```

**Характеристика**: Просматриваем каждый элемент один раз.

### O(n log n) - Линейно-логарифмическая

```go
// Merge Sort
func mergeSort(arr []int) []int {
    if len(arr) <= 1 {
        return arr
    }

    mid := len(arr) / 2
    left := mergeSort(arr[:mid])   // T(n/2)
    right := mergeSort(arr[mid:])  // T(n/2)

    return merge(left, right)      // O(n)
}

// T(n) = 2T(n/2) + O(n) = O(n log n)
```

**Характеристика**: Лучшая возможная сложность для сортировки на основе сравнений.

Подробнее: [[Сортировки - Быстрая, слиянием, пузырьком]]

### O(n²) - Квадратичная

```go
// Bubble Sort
func bubbleSort(arr []int) {
    n := len(arr)
    for i := 0; i < n; i++ {           // O(n)
        for j := 0; j < n-i-1; j++ {   // O(n)
            if arr[j] > arr[j+1] {
                arr[j], arr[j+1] = arr[j+1], arr[j]
            }
        }
    }
} // O(n²)

// Поиск дубликатов (наивный)
func hasDuplicates(arr []int) bool {
    for i := 0; i < len(arr); i++ {
        for j := i + 1; j < len(arr); j++ {
            if arr[i] == arr[j] {
                return true
            }
        }
    }
    return false // O(n²)
}
```

**Характеристика**: Вложенные циклы по одним и тем же данным.

### O(2ⁿ) - Экспоненциальная

```go
// Наивный Fibonacci
func fibonacci(n int) int {
    if n <= 1 {
        return n
    }
    return fibonacci(n-1) + fibonacci(n-2) // O(2ⁿ)
}

// Дерево вызовов для fib(5):
//           fib(5)
//        /         \
//     fib(4)      fib(3)
//    /    \       /    \
// fib(3) fib(2) fib(2) fib(1)
// ...
```

**Характеристика**: Удваивается на каждом шаге. Очень медленно для больших n.

### O(n!) - Факториальная

```go
// Все перестановки массива
func permutations(arr []int) [][]int {
    if len(arr) == 0 {
        return [][]int{{}}
    }

    var result [][]int
    for i, v := range arr {
        rest := append([]int{}, arr[:i]...)
        rest = append(rest, arr[i+1:]...)

        for _, perm := range permutations(rest) {
            result = append(result, append([]int{v}, perm...))
        }
    }

    return result // O(n!)
}

// n=3: 6 перестановок
// n=10: 3,628,800 перестановок!
```

**Характеристика**: Комбинаторный взрыв. Непрактично для n > 10-12.

## Анализ сложных алгоритмов

### Пример 1: Два последовательных цикла

```go
func example1(arr []int) {
    // Цикл 1: O(n)
    for _, v := range arr {
        fmt.Println(v)
    }

    // Цикл 2: O(n)
    for _, v := range arr {
        fmt.Println(v * 2)
    }
}

// Итого: O(n) + O(n) = O(2n) = O(n)
```

### Пример 2: Вложенные циклы

```go
func example2(arr []int) {
    n := len(arr)

    for i := 0; i < n; i++ {           // O(n)
        for j := 0; j < n; j++ {       // O(n)
            fmt.Printf("%d, %d\n", i, j)
        }
    }
}

// Итого: O(n) * O(n) = O(n²)
```

### Пример 3: Неполные вложенные циклы

```go
func example3(arr []int) {
    n := len(arr)

    for i := 0; i < n; i++ {
        for j := i + 1; j < n; j++ {   // Меньше итераций!
            fmt.Printf("%d, %d\n", i, j)
        }
    }
}

// Итерации: n + (n-1) + (n-2) + ... + 1 = n(n+1)/2
// Итого: O(n²/2) = O(n²)
```

### Пример 4: Логарифмический цикл

```go
func example4(n int) {
    for i := 1; i < n; i *= 2 {  // i удваивается: 1, 2, 4, 8, 16...
        fmt.Println(i)
    }
}

// Итерации: log₂(n)
// Итого: O(log n)
```

### Пример 5: Рекурсия с половинным делением

```go
func example5(n int) {
    if n <= 0 {
        return
    }

    example5(n / 2)  // Делим проблему пополам
    example5(n / 2)

    // O(n) работы на каждом уровне
    for i := 0; i < n; i++ {
        fmt.Println(i)
    }
}

// T(n) = 2T(n/2) + O(n)
// Итого: O(n log n) по Master Theorem
```

## Space Complexity

Сложность по памяти - сколько дополнительной памяти использует алгоритм.

### O(1) - Константная память

```go
func swap(a, b int) (int, int) {
    return b, a // Только два значения
}

func findMax(arr []int) int {
    max := arr[0] // Одна переменная
    for _, v := range arr {
        if v > max {
            max = v
        }
    }
    return max
}
```

### O(n) - Линейная память

```go
func reverse(arr []int) []int {
    result := make([]int, len(arr)) // O(n) памяти
    for i, v := range arr {
        result[len(arr)-1-i] = v
    }
    return result
}

// Рекурсия тоже использует память (call stack)!
func factorial(n int) int {
    if n <= 1 {
        return 1
    }
    return n * factorial(n - 1) // O(n) на стеке вызовов
}
```

### O(n²) - Квадратичная память

```go
// Матрица смежности для графа
func createAdjMatrix(n int) [][]bool {
    matrix := make([][]bool, n)
    for i := range matrix {
        matrix[i] = make([]bool, n) // n * n = O(n²)
    }
    return matrix
}
```

## Амортизированная сложность

Средняя сложность операции на длинной последовательности.

```go
// append к слайсу
slice := make([]int, 0)

for i := 0; i < 1000; i++ {
    slice = append(slice, i)
}

// Отдельный append может быть O(n) при reallocation
// Но амортизированная сложность O(1)!
```

Подробнее: [[Амортизированная сложность]]

## Best, Average, Worst Case

| Алгоритм | Best | Average | Worst |
|----------|------|---------|-------|
| Linear Search | O(1) | O(n) | O(n) |
| Binary Search | O(1) | O(log n) | O(log n) |
| Quick Sort | O(n log n) | O(n log n) | O(n²) |
| Merge Sort | O(n log n) | O(n log n) | O(n log n) |
| Hash Table Get | O(1) | O(1) | O(n) |

## Практические советы

### 1. Оптимизация алгоритма важнее константной оптимизации

```go
// ❌ O(n²) - медленно даже с оптимизацией
for i := 0; i < n; i++ {
    for j := 0; j < n; j++ {
        // Быстрая операция
    }
}

// ✅ O(n) - быстро даже с медленной операцией
m := make(map[int]bool)
for _, v := range arr {
    m[v] = true // Чуть медленнее, но O(1)
}
```

### 2. Используйте правильные структуры данных

```go
// ❌ O(n) для поиска
var list []int
for _, v := range list {
    if v == target {
        return true
    }
}

// ✅ O(1) для поиска
set := make(map[int]bool)
if set[target] {
    return true
}
```

Подробнее: [[HashMap - Реализация и особенности]]

### 3. Избегайте ненужных вложенных циклов

```go
// ❌ O(n²) - поиск дубликатов
func hasDuplicates(arr []int) bool {
    for i := 0; i < len(arr); i++ {
        for j := i + 1; j < len(arr); j++ {
            if arr[i] == arr[j] {
                return true
            }
        }
    }
    return false
}

// ✅ O(n) - через map
func hasDuplicates(arr []int) bool {
    seen := make(map[int]bool)
    for _, v := range arr {
        if seen[v] {
            return true
        }
        seen[v] = true
    }
    return false
}
```

### 4. Master Theorem для рекурсии

Для рекурсии вида `T(n) = aT(n/b) + f(n)`:

```
Если f(n) = O(n^c):
- c < log_b(a): T(n) = O(n^log_b(a))
- c = log_b(a): T(n) = O(n^c log n)
- c > log_b(a): T(n) = O(f(n))
```

**Примеры:**
- Merge Sort: `T(n) = 2T(n/2) + O(n)` → `O(n log n)`
- Binary Search: `T(n) = T(n/2) + O(1)` → `O(log n)`

## Частые ошибки

### 1. Путать O(log n) и O(n)

```go
// ❌ O(n) - проходим все элементы
for _, v := range arr {
    // ...
}

// ✅ O(log n) - делим пополам
mid := len(arr) / 2
search(arr[:mid])
```

### 2. Забывать про сложность встроенных функций

```go
// ❌ Выглядит как O(n), но это O(n²)!
for _, v := range arr {
    result = append(result, v) // append может быть O(n)!
}

// ✅ Pre-allocate для O(n)
result := make([]int, 0, len(arr))
for _, v := range arr {
    result = append(result, v)
}
```

### 3. Игнорировать пространственную сложность

```go
// ❌ O(1) время, но O(n) память!
cache := make(map[int]Result)
for _, v := range arr {
    cache[v] = compute(v) // Храним все результаты
}
```

## Связанные темы

- [[Временная и пространственная сложность]]
- [[Амортизированная сложность]]
- [[Бинарный поиск]]
- [[Сортировки - Быстрая, слиянием, пузырьком]]
- [[HashMap - Реализация и особенности]]
- [[Динамическое программирование - Основы]]
