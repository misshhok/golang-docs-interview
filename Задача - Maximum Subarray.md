# Задача - Maximum Subarray

Найти подмассив с максимальной суммой элементов (алгоритм Кадане).

## Условие

Дан целочисленный массив `nums`. Найти подмассив с наибольшей суммой и вернуть эту сумму.

**Подмассив** - это непрерывная часть массива.

**Пример 1:**
```
Вход: nums = [-2,1,-3,4,-1,2,1,-5,4]
Выход: 6
Объяснение: Подмассив [4,-1,2,1] имеет наибольшую сумму 6
```

**Пример 2:**
```
Вход: nums = [1]
Выход: 1
Объяснение: Подмассив [1] имеет наибольшую сумму 1
```

**Пример 3:**
```
Вход: nums = [5,4,-1,7,8]
Выход: 23
Объяснение: Подмассив [5,4,-1,7,8] имеет наибольшую сумму 23
```

**Пример 4:**
```
Вход: nums = [-1,-2,-3,-4]
Выход: -1
Объяснение: Подмассив [-1] имеет наибольшую сумму -1
```

## Анализ задачи

**Ключевые наблюдения:**
- Если текущая сумма становится отрицательной, лучше начать новый подмассив
- Нужно отслеживать максимум на каждом шаге
- Это классическая задача на динамическое программирование

**Подходы:**
1. **Brute Force** - проверить все возможные подмассивы O(n³) или O(n²)
2. **Kadane's Algorithm** - динамическое программирование O(n) ✅
3. **Divide and Conquer** - разделяй и властвуй O(n log n)

## Решение 1: Brute Force

**Идея:** Проверить все возможные подмассивы.

**Сложность:** O(n²) по времени, O(1) по памяти

```go
func maxSubArrayBruteForce(nums []int) int {
    maxSum := nums[0]

    for i := 0; i < len(nums); i++ {
        currentSum := 0
        for j := i; j < len(nums); j++ {
            currentSum += nums[j]
            if currentSum > maxSum {
                maxSum = currentSum
            }
        }
    }

    return maxSum
}
```

**Недостатки:**
- ❌ Квадратичная сложность
- ❌ Неэффективно для больших массивов

## Решение 2: Kadane's Algorithm (Оптимальное)

**Идея:** На каждом шаге решаем: продолжить текущий подмассив или начать новый.

**Формула:**
```
currentSum = max(nums[i], currentSum + nums[i])
maxSum = max(maxSum, currentSum)
```

**Интуиция:** Если `currentSum + nums[i]` меньше `nums[i]`, то лучше начать новый подмассив с текущего элемента.

**Сложность:** O(n) по времени, O(1) по памяти ✅

```go
func maxSubArray(nums []int) int {
    if len(nums) == 0 {
        return 0
    }

    maxSum := nums[0]
    currentSum := nums[0]

    for i := 1; i < len(nums); i++ {
        // Решаем: продолжить или начать заново
        currentSum = max(nums[i], currentSum+nums[i])

        // Обновляем максимум
        maxSum = max(maxSum, currentSum)
    }

    return maxSum
}

func max(a, b int) int {
    if a > b {
        return a
    }
    return b
}
```

**Альтернативная запись:**
```go
func maxSubArray(nums []int) int {
    maxSum := nums[0]
    currentSum := nums[0]

    for i := 1; i < len(nums); i++ {
        // Если текущая сумма отрицательная, начинаем заново
        if currentSum < 0 {
            currentSum = 0
        }

        currentSum += nums[i]
        maxSum = max(maxSum, currentSum)
    }

    return maxSum
}
```

## Пошаговое объяснение

Рассмотрим пример: `nums = [-2, 1, -3, 4, -1, 2, 1, -5, 4]`

```
i | nums[i] | currentSum              | maxSum | Объяснение
--|---------|-------------------------|--------|------------
0 |   -2    | -2                      |   -2   | Инициализация
1 |    1    | max(1, -2+1)=-1 → 1     |    1   | Начали заново
2 |   -3    | max(-3, 1-3)=-2 → -2    |    1   | Продолжаем
3 |    4    | max(4, -2+4)=2 → 4      |    4   | Начали заново
4 |   -1    | max(-1, 4-1)=3 → 3      |    4   | Продолжаем
5 |    2    | max(2, 3+2)=5 → 5       |    5   | Продолжаем
6 |    1    | max(1, 5+1)=6 → 6       |    6   | Продолжаем ✅
7 |   -5    | max(-5, 6-5)=1 → 1      |    6   | Продолжаем
8 |    4    | max(4, 1+4)=5 → 5       |    6   | Продолжаем

Ответ: 6 (подмассив [4, -1, 2, 1])
```

## Визуализация

```
Массив: [-2, 1, -3, 4, -1, 2, 1, -5, 4]

Подмассив с максимальной суммой:
              [4, -1, 2, 1]
              ↓   ↓   ↓  ↓
              4 + (-1) + 2 + 1 = 6 ✅

Почему не включили -5 и 4 в конце?
[4, -1, 2, 1, -5, 4] = 5 < 6
```

### Пример с отрицательными числами

```
Массив: [-1, -2, -3, -4]

i | nums[i] | currentSum           | maxSum
--|---------|----------------------|-------
0 |   -1    | -1                   |   -1
1 |   -2    | max(-2, -1-2) = -2   |   -1
2 |   -3    | max(-3, -2-3) = -3   |   -1
3 |   -4    | max(-4, -3-4) = -4   |   -1

Ответ: -1 (наименьшее отрицательное)
```

## Решение 3: Divide and Conquer

**Идея:** Разделить массив пополам, найти максимум в левой, правой и пересекающей середину частях.

**Сложность:** O(n log n) по времени, O(log n) по памяти (стек рекурсии)

```go
func maxSubArrayDivideConquer(nums []int) int {
    return maxSubArrayHelper(nums, 0, len(nums)-1)
}

func maxSubArrayHelper(nums []int, left, right int) int {
    if left == right {
        return nums[left]
    }

    mid := (left + right) / 2

    // Максимум в левой половине
    leftMax := maxSubArrayHelper(nums, left, mid)

    // Максимум в правой половине
    rightMax := maxSubArrayHelper(nums, mid+1, right)

    // Максимум пересекающий середину
    crossMax := maxCrossingSum(nums, left, mid, right)

    return max(max(leftMax, rightMax), crossMax)
}

func maxCrossingSum(nums []int, left, mid, right int) int {
    // Максимум слева от середины
    leftSum := nums[mid]
    sum := 0
    for i := mid; i >= left; i-- {
        sum += nums[i]
        if sum > leftSum {
            leftSum = sum
        }
    }

    // Максимум справа от середины
    rightSum := nums[mid+1]
    sum = 0
    for i := mid + 1; i <= right; i++ {
        sum += nums[i]
        if sum > rightSum {
            rightSum = sum
        }
    }

    return leftSum + rightSum
}
```

## Вернуть сам подмассив (не только сумму)

```go
func maxSubArrayWithIndices(nums []int) (int, int, int) {
    if len(nums) == 0 {
        return 0, 0, 0
    }

    maxSum := nums[0]
    currentSum := nums[0]
    start, end := 0, 0
    tempStart := 0

    for i := 1; i < len(nums); i++ {
        if currentSum < 0 {
            currentSum = 0
            tempStart = i
        }

        currentSum += nums[i]

        if currentSum > maxSum {
            maxSum = currentSum
            start = tempStart
            end = i
        }
    }

    return maxSum, start, end
}

// Использование
func main() {
    nums := []int{-2, 1, -3, 4, -1, 2, 1, -5, 4}
    sum, start, end := maxSubArrayWithIndices(nums)

    fmt.Printf("Максимальная сумма: %d\n", sum)
    fmt.Printf("Подмассив: %v\n", nums[start:end+1])
}
```

## Тестовые случаи

```go
func TestMaxSubArray(t *testing.T) {
    tests := []struct {
        nums     []int
        expected int
    }{
        {
            nums:     []int{-2, 1, -3, 4, -1, 2, 1, -5, 4},
            expected: 6,
        },
        {
            nums:     []int{1},
            expected: 1,
        },
        {
            nums:     []int{5, 4, -1, 7, 8},
            expected: 23,
        },
        {
            nums:     []int{-1, -2, -3, -4},
            expected: -1,
        },
        {
            nums:     []int{-2, -1},
            expected: -1,
        },
        {
            nums:     []int{1, 2, 3, 4, 5},
            expected: 15,
        },
        {
            nums:     []int{-1, 0, -2},
            expected: 0,
        },
    }

    for _, tt := range tests {
        result := maxSubArray(tt.nums)
        if result != tt.expected {
            t.Errorf("maxSubArray(%v) = %d, expected %d",
                tt.nums, result, tt.expected)
        }
    }
}
```

## Полное решение

```go
package main

import "fmt"

func maxSubArray(nums []int) int {
    if len(nums) == 0 {
        return 0
    }

    maxSum := nums[0]
    currentSum := nums[0]

    for i := 1; i < len(nums); i++ {
        currentSum = max(nums[i], currentSum+nums[i])
        maxSum = max(maxSum, currentSum)
    }

    return maxSum
}

func max(a, b int) int {
    if a > b {
        return a
    }
    return b
}

func main() {
    testCases := [][]int{
        {-2, 1, -3, 4, -1, 2, 1, -5, 4},
        {1},
        {5, 4, -1, 7, 8},
        {-1, -2, -3, -4},
    }

    for i, nums := range testCases {
        result := maxSubArray(nums)
        fmt.Printf("Тест %d: %v → Максимальная сумма: %d\n", i+1, nums, result)
    }
}
```

## Вариации задачи

### 1. Максимальный подмассив с длиной не больше k

```go
func maxSubArrayLenK(nums []int, k int) int {
    maxSum := nums[0]

    for i := 0; i < len(nums); i++ {
        currentSum := 0
        for j := i; j < len(nums) && j-i < k; j++ {
            currentSum += nums[j]
            maxSum = max(maxSum, currentSum)
        }
    }

    return maxSum
}
```

### 2. Максимальная сумма циклического подмассива

```go
func maxSubarraySumCircular(nums []int) int {
    // Случай 1: максимальный подмассив не пересекает границу
    maxKadane := maxSubArray(nums)

    // Случай 2: максимальный подмассив пересекает границу
    // Это эквивалентно: totalSum - минимальный подмассив
    totalSum := 0
    for i := range nums {
        totalSum += nums[i]
        nums[i] = -nums[i]  // Инвертируем для поиска минимума
    }

    maxWrap := totalSum + maxSubArray(nums)  // + потому что инвертировали

    // Если все числа отрицательные
    if maxWrap == 0 {
        return maxKadane
    }

    return max(maxKadane, maxWrap)
}
```

### 3. Максимальное произведение подмассива

```go
func maxProduct(nums []int) int {
    if len(nums) == 0 {
        return 0
    }

    maxProd := nums[0]
    minProd := nums[0]
    result := nums[0]

    for i := 1; i < len(nums); i++ {
        if nums[i] < 0 {
            // Меняем местами при отрицательном числе
            maxProd, minProd = minProd, maxProd
        }

        maxProd = max(nums[i], maxProd*nums[i])
        minProd = min(nums[i], minProd*nums[i])

        result = max(result, maxProd)
    }

    return result
}
```

## Где спрашивают

- **LeetCode:** #53 - Maximum Subarray
- **Яндекс:** Очень популярная задача
- **Авито:** Часто на алгоритмическом раунде
- **Т-Банк:** Могут спросить вариацию с произведением
- **Озон:** Классическая задача на DP
- **Google, Facebook, Amazon:** В топе популярных задач

## Вопросы с собеседований

**Вопрос 1:** Почему алгоритм Кадане работает?

**Ответ:** На каждом шаге мы делаем жадный выбор: если добавление текущего элемента к текущей сумме дает меньше, чем сам элемент, значит предыдущий подмассив "тянет вниз" и лучше начать заново. Это оптимально, так как мы всегда держим максимально возможную сумму на текущий момент.

---

**Вопрос 2:** Как адаптировать для кругового массива?

**Ответ:** Два случая: 1) Максимальный подмассив внутри (обычный Kadane), 2) Максимальный подмассив пересекает границу (totalSum - минимальный подмассив).

---

**Вопрос 3:** Можно ли использовать для 2D массива?

**Ответ:** Да, можно расширить. Для каждой пары строк суммируем столбцы и применяем Kadane к полученному 1D массиву. Сложность O(n³) для матрицы n×n.

---

**Вопрос 4:** Как изменится решение, если нужен подмассив длиной минимум k?

**Ответ:** Сначала находим сумму первых k элементов. Затем используем модифицированный Kadane, поддерживая окно минимум k элементов.

## Best Practices

1. ✅ Используйте алгоритм Кадане для O(n) решения
2. ✅ Помните про случай всех отрицательных чисел
3. ✅ Инициализируйте maxSum и currentSum первым элементом
4. ✅ Для поиска индексов отслеживайте tempStart
5. ❌ Не забывайте обновлять maxSum на каждой итерации
6. ❌ Не путайте max(a, b) с Math.max (в Go нет встроенной)

## Применение в реальности

### Анализ финансовых данных

```go
// Найти период с максимальной прибылью
func maxProfitPeriod(dailyChanges []float64) (float64, int, int) {
    maxProfit := dailyChanges[0]
    currentProfit := dailyChanges[0]
    start, end := 0, 0
    tempStart := 0

    for i := 1; i < len(dailyChanges); i++ {
        if currentProfit < 0 {
            currentProfit = 0
            tempStart = i
        }

        currentProfit += dailyChanges[i]

        if currentProfit > maxProfit {
            maxProfit = currentProfit
            start = tempStart
            end = i
        }
    }

    return maxProfit, start, end
}
```

### Мониторинг производительности

```go
// Найти период с лучшей производительностью сервера
func findBestPerformancePeriod(metrics []int) Period {
    sum, start, end := maxSubArrayWithIndices(metrics)

    return Period{
        Score:     sum,
        StartTime: time.Now().Add(-time.Duration(len(metrics)-start) * time.Hour),
        EndTime:   time.Now().Add(-time.Duration(len(metrics)-end) * time.Hour),
    }
}
```

## Связанные темы

- [[Динамическое программирование - Основы]]
- [[Алгоритмическая сложность (Big O)]]
- [[Go - Массивы и слайсы]]
- [[Жадные алгоритмы]]
