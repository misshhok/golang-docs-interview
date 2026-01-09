# Задача - Container With Most Water

Найти два столбца, которые вместе с осью x образуют контейнер с максимальным количеством воды.

## Условие

Дан массив целых чисел `height` длиной `n`. Есть `n` вертикальных линий, где конечные точки i-й линии находятся в `(i, 0)` и `(i, height[i])`.

Найти две линии, которые вместе с осью x образуют контейнер, содержащий наибольшее количество воды.

**Важно:** Контейнер нельзя наклонять.

**Пример 1:**
```
Вход: height = [1,8,6,2,5,4,8,3,7]
Выход: 49
Объяснение:
Линии на индексах 1 (высота 8) и 8 (высота 7)
Площадь = min(8, 7) * (8 - 1) = 7 * 7 = 49
```

**Пример 2:**
```
Вход: height = [1,1]
Выход: 1
```

**Пример 3:**
```
Вход: height = [4,3,2,1,4]
Выход: 16
Объяснение:
Линии на индексах 0 и 4
Площадь = min(4, 4) * (4 - 0) = 4 * 4 = 16
```

## Анализ задачи

**Формула площади:**
```
area = min(height[i], height[j]) * (j - i)

где:
- min(height[i], height[j]) - высота воды (ограничена меньшей линией)
- (j - i) - ширина контейнера
```

**Ключевые наблюдения:**
- Высота воды ограничена меньшей из двух линий
- Чем больше расстояние между линиями, тем больше ширина
- Нужен баланс между высотой и шириной

**Подходы:**
1. **Brute Force** - проверить все пары O(n²)
2. **Два указателя** - жадный алгоритм O(n) ✅

## Решение 1: Brute Force

**Идея:** Проверить все возможные пары линий.

**Сложность:** O(n²) по времени, O(1) по памяти

```go
func maxAreaBruteForce(height []int) int {
    maxArea := 0

    for i := 0; i < len(height); i++ {
        for j := i + 1; j < len(height); j++ {
            // Вычисляем площадь
            h := min(height[i], height[j])
            w := j - i
            area := h * w

            if area > maxArea {
                maxArea = area
            }
        }
    }

    return maxArea
}

func min(a, b int) int {
    if a < b {
        return a
    }
    return b
}
```

**Недостатки:**
- ❌ Квадратичная сложность
- ❌ Неэффективно для больших массивов

## Решение 2: Два указателя (Оптимальное)

**Идея:** Начинаем с максимальной ширины (начало и конец), двигаем меньший указатель внутрь.

**Почему это работает?**
- Начинаем с максимальной ширины
- Площадь ограничена меньшей высотой
- Двигая меньший указатель, есть шанс найти большую высоту
- Двигая больший указатель, только уменьшаем ширину без гарантии увеличения высоты

**Сложность:** O(n) по времени, O(1) по памяти ✅

```go
func maxArea(height []int) int {
    maxArea := 0
    left, right := 0, len(height)-1

    for left < right {
        // Вычисляем площадь для текущей пары
        h := min(height[left], height[right])
        w := right - left
        area := h * w

        // Обновляем максимум
        if area > maxArea {
            maxArea = area
        }

        // Двигаем указатель с меньшей высотой
        if height[left] < height[right] {
            left++
        } else {
            right--
        }
    }

    return maxArea
}

func min(a, b int) int {
    if a < b {
        return a
    }
    return b
}
```

**Преимущества:**
- ✅ Линейная сложность
- ✅ Один проход по массиву
- ✅ Константная память

## Пошаговое объяснение

Рассмотрим пример: `height = [1,8,6,2,5,4,8,3,7]`

```
Индексы: 0  1  2  3  4  5  6  7  8
Высоты:  1  8  6  2  5  4  8  3  7
```

### Шаг 1: left=0, right=8
```
height[0]=1, height[8]=7
area = min(1, 7) * (8-0) = 1 * 8 = 8
maxArea = 8

height[0] < height[8], двигаем left
left = 1
```

### Шаг 2: left=1, right=8
```
height[1]=8, height[8]=7
area = min(8, 7) * (8-1) = 7 * 7 = 49 ✅
maxArea = 49

height[1] > height[8], двигаем right
right = 7
```

### Шаг 3: left=1, right=7
```
height[1]=8, height[7]=3
area = min(8, 3) * (7-1) = 3 * 6 = 18
maxArea = 49

height[1] > height[7], двигаем right
right = 6
```

### Шаг 4: left=1, right=6
```
height[1]=8, height[6]=8
area = min(8, 8) * (6-1) = 8 * 5 = 40
maxArea = 49

height[1] == height[6], двигаем любой (right)
right = 5
```

### Продолжаем...

```
Итоговый максимум: 49 ✅
(между индексами 1 и 8)
```

## Визуализация

```
height = [1, 8, 6, 2, 5, 4, 8, 3, 7]

Графически:
  8 |   █       █
  7 |   █       █   █
  6 |   █ █     █   █
  5 |   █ █   █ █   █
  4 |   █ █   █ █ █ █
  3 |   █ █   █ █ █ █ █
  2 |   █ █ █ █ █ █ █ █
  1 | █ █ █ █ █ █ █ █ █
    +-------------------
      0 1 2 3 4 5 6 7 8

Максимальный контейнер:
Между позициями 1 и 8
Высота: min(8, 7) = 7
Ширина: 8 - 1 = 7
Площадь: 7 * 7 = 49

    |   █       █   |
    |   █       █   |
    |   █       █   |
    |   █       █   |
    |   █       █   |
    |   █       █   |
    |   █       █   |
    +-------------------
        1           8
```

## Почему жадный алгоритм работает?

**Доказательство корректности:**

1. Начинаем с максимальной ширины `[0, n-1]`

2. Площадь = `min(h[left], h[right]) * width`

3. Если `h[left] < h[right]`:
   - Текущая высота ограничена `h[left]`
   - Двигая `right` влево, уменьшаем ширину
   - Высота остается `≤ h[left]` (ограничена меньшим)
   - Площадь только уменьшится
   - **Вывод:** Бессмысленно двигать `right`, двигаем `left`

4. Аналогично для `h[left] ≥ h[right]`

5. Таким образом, мы не пропускаем оптимальное решение

**Пример:**
```
[4, 3, 2, 1, 4]
 ↑           ↑

Area = min(4, 4) * 4 = 16

Двигая любой указатель, width уменьшается до 3
Максимальная высота остается 4
Max area = 4 * 3 = 12 < 16

Оптимум найден!
```

## Тестовые случаи

```go
func TestMaxArea(t *testing.T) {
    tests := []struct {
        height   []int
        expected int
    }{
        {
            height:   []int{1, 8, 6, 2, 5, 4, 8, 3, 7},
            expected: 49,
        },
        {
            height:   []int{1, 1},
            expected: 1,
        },
        {
            height:   []int{4, 3, 2, 1, 4},
            expected: 16,
        },
        {
            height:   []int{1, 2, 1},
            expected: 2,
        },
        {
            height:   []int{2, 3, 4, 5, 18, 17, 6},
            expected: 17,
        },
    }

    for _, tt := range tests {
        result := maxArea(tt.height)
        if result != tt.expected {
            t.Errorf("maxArea(%v) = %d, expected %d",
                tt.height, result, tt.expected)
        }
    }
}
```

## Полное решение

```go
package main

import "fmt"

func maxArea(height []int) int {
    maxArea := 0
    left, right := 0, len(height)-1

    for left < right {
        // Вычисляем площадь
        h := min(height[left], height[right])
        w := right - left
        area := h * w

        // Обновляем максимум
        maxArea = max(maxArea, area)

        // Двигаем указатель с меньшей высотой
        if height[left] < height[right] {
            left++
        } else {
            right--
        }
    }

    return maxArea
}

func min(a, b int) int {
    if a < b {
        return a
    }
    return b
}

func max(a, b int) int {
    if a > b {
        return a
    }
    return b
}

func main() {
    testCases := [][]int{
        {1, 8, 6, 2, 5, 4, 8, 3, 7},
        {1, 1},
        {4, 3, 2, 1, 4},
    }

    for i, height := range testCases {
        result := maxArea(height)
        fmt.Printf("Тест %d: %v → Максимальная площадь: %d\n",
            i+1, height, result)
    }
}
```

## Альтернативная реализация с деталями

```go
func maxAreaDetailed(height []int) (int, int, int) {
    maxArea := 0
    bestLeft, bestRight := 0, 0
    left, right := 0, len(height)-1

    for left < right {
        h := min(height[left], height[right])
        w := right - left
        area := h * w

        if area > maxArea {
            maxArea = area
            bestLeft = left
            bestRight = right
        }

        if height[left] < height[right] {
            left++
        } else {
            right--
        }
    }

    return maxArea, bestLeft, bestRight
}

// Использование
func main() {
    height := []int{1, 8, 6, 2, 5, 4, 8, 3, 7}
    area, left, right := maxAreaDetailed(height)

    fmt.Printf("Максимальная площадь: %d\n", area)
    fmt.Printf("Между индексами: %d и %d\n", left, right)
    fmt.Printf("Высоты: %d и %d\n", height[left], height[right])
}
```

## Вариации задачи

### 1. Trapping Rain Water

Сколько воды может быть захвачено между столбцами:

```go
func trap(height []int) int {
    if len(height) == 0 {
        return 0
    }

    left, right := 0, len(height)-1
    leftMax, rightMax := 0, 0
    water := 0

    for left < right {
        if height[left] < height[right] {
            if height[left] >= leftMax {
                leftMax = height[left]
            } else {
                water += leftMax - height[left]
            }
            left++
        } else {
            if height[right] >= rightMax {
                rightMax = height[right]
            } else {
                water += rightMax - height[right]
            }
            right--
        }
    }

    return water
}
```

### 2. Максимальный прямоугольник в гистограмме

```go
func largestRectangleArea(heights []int) int {
    stack := []int{}
    maxArea := 0
    heights = append(heights, 0)  // Sentinel

    for i := 0; i < len(heights); i++ {
        for len(stack) > 0 && heights[i] < heights[stack[len(stack)-1]] {
            h := heights[stack[len(stack)-1]]
            stack = stack[:len(stack)-1]

            var w int
            if len(stack) == 0 {
                w = i
            } else {
                w = i - stack[len(stack)-1] - 1
            }

            maxArea = max(maxArea, h*w)
        }

        stack = append(stack, i)
    }

    return maxArea
}
```

## Где спрашивают

- **LeetCode:** #11 - Container With Most Water
- **LeetCode:** #42 - Trapping Rain Water (более сложная версия)
- **Яндекс:** Популярная задача на собеседованиях
- **Авито:** Часто на алгоритмическом раунде
- **Т-Банк:** Просят объяснить почему жадный алгоритм работает
- **Озон:** Классическая задача на два указателя
- **Google, Facebook:** В топе популярных задач

## Вопросы с собеседований

**Вопрос 1:** Почему мы двигаем указатель с меньшей высотой?

**Ответ:** Площадь ограничена меньшей высотой. Если мы двигаем указатель с большей высотой, мы уменьшаем ширину, но высота остается ограниченной меньшим значением. Поэтому площадь может только уменьшиться. Двигая меньший указатель, мы можем найти большую высоту.

---

**Вопрос 2:** Можно ли пропустить оптимальное решение с этим подходом?

**Ответ:** Нет. Жадный алгоритм гарантирует, что мы не пропустим оптимум, так как мы всегда делаем локально оптимальный выбор: двигаем указатель, который не может улучшить текущую конфигурацию.

---

**Вопрос 3:** Какая разница между этой задачей и Trapping Rain Water?

**Ответ:**
- Container: ищем два столбца, игнорируя все между ними
- Trapping Rain: суммируем воду над каждой позицией между столбцами

---

**Вопрос 4:** Можно ли улучшить сложность?

**Ответ:** Нет, O(n) уже оптимально - нужно просмотреть каждый элемент минимум один раз.

## Best Practices

1. ✅ Используйте два указателя для O(n) решения
2. ✅ Двигайте указатель с меньшей высотой
3. ✅ Отслеживайте максимум на каждой итерации
4. ✅ Помните: площадь = высота × ширина
5. ❌ Не пытайтесь двигать оба указателя одновременно
6. ❌ Не забывайте, что высота ограничена минимумом

## Применение в реальности

### Оптимизация хранилища

```go
// Максимальная емкость между двумя контейнерами
type Container struct {
    Position int
    Height   int
}

func findOptimalContainerPair(containers []Container) (int, int, int) {
    heights := make([]int, len(containers))
    for i, c := range containers {
        heights[i] = c.Height
    }

    area, left, right := maxAreaDetailed(heights)

    return area, containers[left].Position, containers[right].Position
}
```

### Планирование ресурсов

```go
// Найти оптимальный период для максимальной производительности
func findOptimalPeriod(capacity []int) (startDay, endDay, maxThroughput int) {
    maxThroughput, startDay, endDay = maxAreaDetailed(capacity)
    return
}
```

## Связанные темы

- [[Два указателя (Two Pointers)]]
- [[Жадные алгоритмы]]
- [[Алгоритмическая сложность (Big O)]]
- [[Go - Массивы и слайсы]]
