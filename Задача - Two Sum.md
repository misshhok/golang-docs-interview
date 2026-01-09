# Задача - Two Sum

Найти два числа в массиве, сумма которых равна target.

## Условие

Дан массив целых чисел `nums` и целое число `target`. Нужно найти индексы двух чисел, сумма которых равна `target`.

**Требования:**
- Каждый элемент можно использовать только один раз
- Гарантируется, что существует ровно одно решение
- Вернуть индексы в любом порядке

**Пример:**
```
Вход: nums = [2, 7, 11, 15], target = 9
Выход: [0, 1]
Объяснение: nums[0] + nums[1] = 2 + 7 = 9
```

**Пример 2:**
```
Вход: nums = [3, 2, 4], target = 6
Выход: [1, 2]
```

**Пример 3:**
```
Вход: nums = [3, 3], target = 6
Выход: [0, 1]
```

## Анализ задачи

**Ключевые наблюдения:**
- Для каждого числа `x` ищем дополнение `target - x`
- Нужно помнить индексы уже просмотренных элементов
- Можно использовать HashMap для быстрого поиска

**Edge cases:**
- Массив из двух элементов
- Одинаковые числа (например, [3, 3])
- Отрицательные числа

## Решение 1: Brute Force

**Идея:** Проверить все возможные пары чисел.

**Алгоритм:**
1. Два вложенных цикла
2. Для каждой пары проверяем сумму
3. Если сумма равна target, возвращаем индексы

**Сложность:** O(n²) по времени, O(1) по памяти

```go
func twoSumBruteForce(nums []int, target int) []int {
    // Перебираем все возможные пары
    for i := 0; i < len(nums); i++ {
        for j := i + 1; j < len(nums); j++ {
            if nums[i] + nums[j] == target {
                return []int{i, j}
            }
        }
    }

    return nil  // Решение не найдено
}
```

**Недостатки:**
- ❌ Неэффективно для больших массивов
- ❌ Квадратичная сложность

## Решение 2: HashMap (Оптимальное)

**Идея:** Использовать HashMap для хранения уже просмотренных элементов и их индексов.

**Алгоритм:**
1. Создаем map для хранения: число → индекс
2. Для каждого элемента вычисляем дополнение: `complement = target - nums[i]`
3. Проверяем, есть ли дополнение в map
4. Если есть - нашли пару, возвращаем индексы
5. Если нет - добавляем текущий элемент в map

**Сложность:** O(n) по времени, O(n) по памяти

```go
func twoSum(nums []int, target int) []int {
    // Карта: число → индекс
    seen := make(map[int]int)

    for i, num := range nums {
        // Вычисляем дополнение
        complement := target - num

        // Проверяем, видели ли мы дополнение раньше
        if j, ok := seen[complement]; ok {
            return []int{j, i}
        }

        // Запоминаем текущее число и его индекс
        seen[num] = i
    }

    return nil  // Решение не найдено
}
```

**Преимущества:**
- ✅ Один проход по массиву
- ✅ Линейная сложность
- ✅ Простая и понятная логика

## Пошаговое объяснение

Рассмотрим пример: `nums = [2, 7, 11, 15]`, `target = 9`

### Шаг 1: i = 0, num = 2
```
complement = 9 - 2 = 7
seen = {}
7 не найден в seen
seen = {2: 0}
```

### Шаг 2: i = 1, num = 7
```
complement = 9 - 7 = 2
seen = {2: 0}
2 найден в seen на индексе 0! ✅
Возвращаем [0, 1]
```

## Визуализация

```
nums:   [2,   7,  11,  15]
индексы: 0    1   2    3
target = 9

Итерация 1:
  num = 2, complement = 7
  seen: {2: 0}

Итерация 2:
  num = 7, complement = 2
  2 найден в seen[0]!
  Ответ: [0, 1] ✅
```

## Тестовые случаи

```go
func TestTwoSum(t *testing.T) {
    tests := []struct {
        nums     []int
        target   int
        expected []int
    }{
        {
            nums:     []int{2, 7, 11, 15},
            target:   9,
            expected: []int{0, 1},
        },
        {
            nums:     []int{3, 2, 4},
            target:   6,
            expected: []int{1, 2},
        },
        {
            nums:     []int{3, 3},
            target:   6,
            expected: []int{0, 1},
        },
        {
            nums:     []int{-1, -2, -3, -4, -5},
            target:   -8,
            expected: []int{2, 4},
        },
    }

    for _, tt := range tests {
        result := twoSum(tt.nums, tt.target)

        // Проверяем, что сумма правильная
        if tt.nums[result[0]] + tt.nums[result[1]] != tt.target {
            t.Errorf("twoSum(%v, %d) = %v, сумма не равна target",
                tt.nums, tt.target, result)
        }
    }
}
```

## Полное решение с обработкой ошибок

```go
package main

import (
    "errors"
    "fmt"
)

func twoSum(nums []int, target int) ([]int, error) {
    if len(nums) < 2 {
        return nil, errors.New("массив должен содержать минимум 2 элемента")
    }

    seen := make(map[int]int)

    for i, num := range nums {
        complement := target - num

        if j, ok := seen[complement]; ok {
            return []int{j, i}, nil
        }

        seen[num] = i
    }

    return nil, errors.New("решение не найдено")
}

func main() {
    nums := []int{2, 7, 11, 15}
    target := 9

    result, err := twoSum(nums, target)
    if err != nil {
        fmt.Println("Ошибка:", err)
        return
    }

    fmt.Printf("Индексы: %v\n", result)
    fmt.Printf("Числа: %d + %d = %d\n",
        nums[result[0]], nums[result[1]], target)
}
```

## Вариации задачи

### Two Sum II (отсортированный массив)

Если массив отсортирован, можно использовать два указателя:

```go
func twoSumSorted(nums []int, target int) []int {
    left, right := 0, len(nums)-1

    for left < right {
        sum := nums[left] + nums[right]

        if sum == target {
            return []int{left, right}
        } else if sum < target {
            left++  // Нужна большая сумма
        } else {
            right--  // Нужна меньшая сумма
        }
    }

    return nil
}
```

**Сложность:** O(n) по времени, O(1) по памяти

### Three Sum

Найти все уникальные тройки, дающие сумму 0:

```go
func threeSum(nums []int) [][]int {
    sort.Ints(nums)
    result := [][]int{}

    for i := 0; i < len(nums)-2; i++ {
        // Пропускаем дубликаты
        if i > 0 && nums[i] == nums[i-1] {
            continue
        }

        // Two Sum для оставшейся части
        target := -nums[i]
        left, right := i+1, len(nums)-1

        for left < right {
            sum := nums[left] + nums[right]

            if sum == target {
                result = append(result, []int{nums[i], nums[left], nums[right]})

                // Пропускаем дубликаты
                for left < right && nums[left] == nums[left+1] {
                    left++
                }
                for left < right && nums[right] == nums[right-1] {
                    right--
                }

                left++
                right--
            } else if sum < target {
                left++
            } else {
                right--
            }
        }
    }

    return result
}
```

### Four Sum

Расширение на четыре числа - используем вложенные циклы + Two Sum.

## Где спрашивают

- **LeetCode:** #1 - Two Sum
- **Яндекс:** Часто на первом этапе
- **Авито:** В качестве разминки
- **Т-Банк:** Могут попросить объяснить сложность
- **Озон:** Могут спросить про вариации (Three Sum)

## Вопросы с собеседований

**Вопрос 1:** Почему HashMap лучше, чем brute force?

**Ответ:** HashMap дает O(1) lookup в среднем случае, что позволяет решить задачу за один проход O(n). Brute force требует O(n²) из-за вложенных циклов.

---

**Вопрос 2:** Что если в массиве есть дубликаты?

**Ответ:** Наше решение корректно работает с дубликатами. Например, для `[3, 3]` и `target = 6`, мы добавим первую тройку в map, а когда встретим вторую - найдем первую.

---

**Вопрос 3:** Можно ли решить с O(1) памятью?

**Ответ:** Если массив несортированный - нет, нужен HashMap. Если отсортированный - да, используя два указателя.

---

**Вопрос 4:** Что если нужно найти все пары, а не одну?

**Ответ:** Продолжаем итерацию после нахождения пары, не возвращаемся сразу. Нужно учесть, что одна пара может встретиться дважды (с разными индексами).

## Best Practices

1. ✅ Используйте HashMap для O(n) решения
2. ✅ Проверяйте edge cases (пустой массив, один элемент)
3. ✅ Для отсортированного массива используйте два указателя
4. ✅ Помните о возможности отрицательных чисел
5. ❌ Не забывайте, что один элемент нельзя использовать дважды

## Связанные темы

- [[Go - Карты (maps)]]
- [[HashMap - Реализация и особенности]]
- [[Алгоритмическая сложность (Big O)]]
- [[Два указателя (Two Pointers)]]
- [[Сортировки - Быстрая, слиянием, пузырьком]]
