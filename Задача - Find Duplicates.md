# Задача - Find Duplicates

Найти дубликаты в массиве.

## Условие

Дан массив целых чисел `nums` размером `n`, где числа находятся в диапазоне `[1, n]`. Каждое целое число может появляться один или два раза. Найти все числа, которые появляются дважды.

**Требования:**
- Линейное время O(n)
- Константная память O(1) (без учета выходного массива)

**Пример 1:**
```
Вход: nums = [4,3,2,7,8,2,3,1]
Выход: [2,3]
```

**Пример 2:**
```
Вход: nums = [1,1,2]
Выход: [1]
```

**Пример 3:**
```
Вход: nums = [1]
Выход: []
```

## Анализ задачи

**Ключевые наблюдения:**
- Числа в диапазоне `[1, n]` где n = длина массива
- Каждое число может быть использовано как индекс (num - 1)
- Можем пометить посещенные элементы, изменяя знак

**Подходы:**
1. **HashSet** - O(n) время, O(n) память
2. **Сортировка** - O(n log n) время, O(1) память
3. **Отрицательные метки** - O(n) время, O(1) память ✅ оптимально
4. **Floyd's Cycle Detection** - O(n) время, O(1) память (для одного дубликата)

## Решение 1: HashSet

**Идея:** Использовать Set для отслеживания встреченных элементов.

**Сложность:** O(n) по времени, O(n) по памяти

```go
func findDuplicatesHashSet(nums []int) []int {
    seen := make(map[int]bool)
    duplicates := []int{}

    for _, num := range nums {
        if seen[num] {
            duplicates = append(duplicates, num)
        } else {
            seen[num] = true
        }
    }

    return duplicates
}
```

**Недостатки:**
- ❌ Использует O(n) дополнительной памяти

## Решение 2: Сортировка

**Идея:** Отсортировать массив, дубликаты будут рядом.

**Сложность:** O(n log n) по времени, O(1) по памяти

```go
func findDuplicatesSort(nums []int) []int {
    sort.Ints(nums)
    duplicates := []int{}

    for i := 1; i < len(nums); i++ {
        if nums[i] == nums[i-1] {
            // Избегаем добавления одного дубликата дважды
            if len(duplicates) == 0 || duplicates[len(duplicates)-1] != nums[i] {
                duplicates = append(duplicates, nums[i])
            }
        }
    }

    return duplicates
}
```

**Недостатки:**
- ❌ Модифицирует входной массив
- ❌ O(n log n) медленнее, чем O(n)

## Решение 3: Отрицательные метки (Оптимальное)

**Идея:** Использовать сами числа как индексы. Когда встречаем число, помечаем соответствующий элемент отрицательным. Если элемент уже отрицательный - это дубликат.

**Алгоритм:**
1. Для каждого числа `num` вычисляем индекс: `index = abs(num) - 1`
2. Если `nums[index]` положительное → делаем отрицательным
3. Если `nums[index]` уже отрицательное → нашли дубликат

**Сложность:** O(n) по времени, O(1) по памяти ✅

```go
func findDuplicates(nums []int) []int {
    duplicates := []int{}

    for _, num := range nums {
        // Получаем абсолютное значение (число может быть помечено)
        index := abs(num) - 1

        // Если элемент уже отрицательный - дубликат
        if nums[index] < 0 {
            duplicates = append(duplicates, abs(num))
        } else {
            // Помечаем как посещенный (делаем отрицательным)
            nums[index] = -nums[index]
        }
    }

    // Опционально: восстанавливаем массив
    for i := range nums {
        nums[i] = abs(nums[i])
    }

    return duplicates
}

func abs(x int) int {
    if x < 0 {
        return -x
    }
    return x
}
```

## Пошаговое объяснение

Рассмотрим пример: `nums = [4,3,2,7,8,2,3,1]`

### Исходный массив
```
Индекс:  0  1  2  3  4  5  6  7
Значение: 4  3  2  7  8  2  3  1
```

### Шаг 1: num = 4
```
index = 4 - 1 = 3
nums[3] = 7 (положительное)
Помечаем: nums[3] = -7

Массив: [4, 3, 2, -7, 8, 2, 3, 1]
```

### Шаг 2: num = 3
```
index = 3 - 1 = 2
nums[2] = 2 (положительное)
Помечаем: nums[2] = -2

Массив: [4, 3, -2, -7, 8, 2, 3, 1]
```

### Шаг 3: num = -2 (уже помечен)
```
index = |-2| - 1 = 1
nums[1] = 3 (положительное)
Помечаем: nums[1] = -3

Массив: [4, -3, -2, -7, 8, 2, 3, 1]
```

### Шаг 4: num = -7
```
index = |-7| - 1 = 6
nums[6] = 3 (положительное)
Помечаем: nums[6] = -3

Массив: [4, -3, -2, -7, 8, 2, -3, 1]
```

### Шаг 5: num = 8
```
index = 8 - 1 = 7
nums[7] = 1 (положительное)
Помечаем: nums[7] = -1

Массив: [4, -3, -2, -7, 8, 2, -3, -1]
```

### Шаг 6: num = 2
```
index = 2 - 1 = 1
nums[1] = -3 (ОТРИЦАТЕЛЬНОЕ!) ✅
Дубликат найден: 2

Массив: [4, -3, -2, -7, 8, 2, -3, -1]
duplicates = [2]
```

### Шаг 7: num = -3
```
index = |-3| - 1 = 2
nums[2] = -2 (ОТРИЦАТЕЛЬНОЕ!) ✅
Дубликат найден: 3

Массив: [4, -3, -2, -7, 8, 2, -3, -1]
duplicates = [2, 3]
```

### Шаг 8: num = -1
```
index = |-1| - 1 = 0
nums[0] = 4 (положительное)
Помечаем: nums[0] = -4

Массив: [-4, -3, -2, -7, 8, 2, -3, -1]
```

### Результат
```
duplicates = [2, 3] ✅
```

## Визуализация

```
nums = [4, 3, 2, 7, 8, 2, 3, 1]
        ↓  ↓  ↓  ↓  ↓  ↓  ↓  ↓
Используем как индексы (num-1):

4 → index 3 → пометить nums[3]
3 → index 2 → пометить nums[2]
2 → index 1 → пометить nums[1]
...
2 → index 1 → УЖЕ ПОМЕЧЕН! → Дубликат ✅
3 → index 2 → УЖЕ ПОМЕЧЕН! → Дубликат ✅
```

## Решение 4: Floyd's Cycle Detection (для одного дубликата)

**Применимо когда:** Все числа в `[1, n-1]`, один дубликат.

**Идея:** Рассматриваем массив как связный список, где `nums[i]` указывает на следующий узел.

**Сложность:** O(n) по времени, O(1) по памяти

```go
func findDuplicateFloyd(nums []int) int {
    // Фаза 1: Найти пересечение в цикле
    slow, fast := nums[0], nums[0]

    for {
        slow = nums[slow]
        fast = nums[nums[fast]]
        if slow == fast {
            break
        }
    }

    // Фаза 2: Найти вход в цикл (дубликат)
    slow = nums[0]
    for slow != fast {
        slow = nums[slow]
        fast = nums[fast]
    }

    return slow
}
```

**Применение:** Специфичная вариация задачи LeetCode #287.

## Тестовые случаи

```go
func TestFindDuplicates(t *testing.T) {
    tests := []struct {
        nums     []int
        expected []int
    }{
        {
            nums:     []int{4, 3, 2, 7, 8, 2, 3, 1},
            expected: []int{2, 3},
        },
        {
            nums:     []int{1, 1, 2},
            expected: []int{1},
        },
        {
            nums:     []int{1},
            expected: []int{},
        },
        {
            nums:     []int{1, 2, 3, 4, 5},
            expected: []int{},
        },
        {
            nums:     []int{5, 4, 3, 2, 1, 1, 2, 3, 4, 5},
            expected: []int{1, 2, 3, 4, 5},
        },
    }

    for _, tt := range tests {
        // Копируем массив т.к. функция модифицирует
        numsCopy := make([]int, len(tt.nums))
        copy(numsCopy, tt.nums)

        result := findDuplicates(numsCopy)
        sort.Ints(result)
        sort.Ints(tt.expected)

        if !reflect.DeepEqual(result, tt.expected) {
            t.Errorf("findDuplicates(%v) = %v, expected %v",
                tt.nums, result, tt.expected)
        }
    }
}
```

## Полное решение

```go
package main

import (
    "fmt"
    "sort"
)

// Оптимальное решение: O(n) время, O(1) память
func findDuplicates(nums []int) []int {
    duplicates := []int{}

    for i := 0; i < len(nums); i++ {
        index := abs(nums[i]) - 1

        if nums[index] < 0 {
            duplicates = append(duplicates, abs(nums[i]))
        } else {
            nums[index] = -nums[index]
        }
    }

    // Восстанавливаем массив
    for i := range nums {
        if nums[i] < 0 {
            nums[i] = -nums[i]
        }
    }

    return duplicates
}

func abs(x int) int {
    if x < 0 {
        return -x
    }
    return x
}

func main() {
    nums := []int{4, 3, 2, 7, 8, 2, 3, 1}

    fmt.Println("Исходный массив:", nums)

    duplicates := findDuplicates(nums)
    sort.Ints(duplicates)

    fmt.Println("Дубликаты:", duplicates)
    fmt.Println("Массив после восстановления:", nums)
}
```

## Вариации задачи

### 1. Найти все отсутствующие числа

```go
func findDisappearedNumbers(nums []int) []int {
    // Помечаем присутствующие
    for i := 0; i < len(nums); i++ {
        index := abs(nums[i]) - 1
        if nums[index] > 0 {
            nums[index] = -nums[index]
        }
    }

    // Собираем непомеченные (отсутствующие)
    missing := []int{}
    for i := 0; i < len(nums); i++ {
        if nums[i] > 0 {
            missing = append(missing, i+1)
        }
    }

    return missing
}
```

### 2. Найти первый дубликат с минимальным расстоянием

```go
func findFirstDuplicate(nums []int) int {
    minDistance := len(nums)
    duplicate := -1

    seen := make(map[int]int)

    for i, num := range nums {
        if lastIndex, exists := seen[num]; exists {
            distance := i - lastIndex
            if distance < minDistance {
                minDistance = distance
                duplicate = num
            }
        }
        seen[num] = i
    }

    return duplicate
}
```

### 3. Подсчитать количество дубликатов

```go
func countDuplicates(nums []int) int {
    count := 0

    for i := 0; i < len(nums); i++ {
        index := abs(nums[i]) - 1

        if nums[index] < 0 {
            count++
        } else {
            nums[index] = -nums[index]
        }
    }

    // Восстанавливаем
    for i := range nums {
        nums[i] = abs(nums[i])
    }

    return count
}
```

## Где спрашивают

- **LeetCode:** #442 - Find All Duplicates in an Array
- **LeetCode:** #287 - Find the Duplicate Number (Floyd's)
- **LeetCode:** #448 - Find All Numbers Disappeared in an Array
- **Яндекс:** Часто на алгоритмическом раунде
- **Авито:** Могут спросить оптимизацию памяти
- **Т-Банк:** Просят объяснить трюк с отрицательными числами
- **Озон:** Вариации с подсчетом

## Вопросы с собеседований

**Вопрос 1:** Почему работает трюк с отрицательными числами?

**Ответ:** Числа в диапазоне [1, n] могут использоваться как индексы [0, n-1]. Знак числа не влияет на его значение как индекса, поэтому можем использовать знак как флаг "посещено".

---

**Вопрос 2:** Что если числа могут быть отрицательными или нулевыми?

**Ответ:** Этот метод не сработает. Нужно использовать HashSet O(n) памяти или другой подход.

---

**Вопрос 3:** Можно ли найти дубликаты без модификации массива с O(1) памятью?

**Ответ:** Для общего случая - нет. Floyd's cycle detection работает для специального случая (числа [1, n] с одним дубликатом), но это исключение.

---

**Вопрос 4:** Как восстановить массив после поиска?

**Ответ:** Пройтись по массиву и взять abs() от каждого элемента, возвращая все в положительное состояние.

## Best Practices

1. ✅ Для диапазона [1, n] используйте отрицательные метки
2. ✅ Всегда берите abs() перед использованием как индекса
3. ✅ Восстанавливайте массив после поиска если нужно
4. ✅ Для произвольных чисел используйте HashSet
5. ❌ Не забывайте про edge cases (пустой массив, один элемент)
6. ❌ Помните об ограничениях метода (только положительные числа)

## Применение в реальности

### Проверка дублирующихся ID

```go
func findDuplicateIDs(ids []int) []int {
    if len(ids) == 0 {
        return []int{}
    }

    // Нормализуем ID к диапазону [1, n]
    minID := ids[0]
    for _, id := range ids {
        if id < minID {
            minID = id
        }
    }

    // Создаем рабочий массив
    normalized := make([]int, len(ids))
    for i, id := range ids {
        normalized[i] = id - minID + 1
    }

    // Ищем дубликаты
    duplicates := findDuplicates(normalized)

    // Восстанавливаем оригинальные ID
    for i := range duplicates {
        duplicates[i] = duplicates[i] + minID - 1
    }

    return duplicates
}
```

### Валидация уникальности в БД

```go
type Record struct {
    ID   int
    Data string
}

func validateUniqueRecords(records []Record) ([]int, error) {
    ids := make([]int, len(records))
    for i, r := range records {
        ids[i] = r.ID
    }

    duplicates := findDuplicatesHashSet(ids)

    if len(duplicates) > 0 {
        return duplicates, fmt.Errorf("found %d duplicate IDs", len(duplicates))
    }

    return nil, nil
}
```

## Связанные темы

- [[HashSet]]
- [[Go - Карты (maps)]]
- [[Алгоритмическая сложность (Big O)]]
- [[Go - Массивы и слайсы]]
- [[Сортировки - Быстрая, слиянием, пузырьком]]
