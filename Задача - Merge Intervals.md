# Задача - Merge Intervals

Объединить пересекающиеся интервалы в один.

## Условие

Дан массив интервалов `intervals`, где `intervals[i] = [start_i, end_i]`. Объедините все пересекающиеся интервалы и верните массив непересекающихся интервалов, которые покрывают все интервалы из входного массива.

**Пример 1:**
```
Вход: intervals = [[1,3],[2,6],[8,10],[15,18]]
Выход: [[1,6],[8,10],[15,18]]
Объяснение: Интервалы [1,3] и [2,6] пересекаются, объединяем в [1,6]
```

**Пример 2:**
```
Вход: intervals = [[1,4],[4,5]]
Выход: [[1,5]]
Объяснение: Интервалы [1,4] и [4,5] считаются пересекающимися
```

**Пример 3:**
```
Вход: intervals = [[1,4],[0,4]]
Выход: [[0,4]]
```

**Пример 4:**
```
Вход: intervals = [[1,4],[2,3]]
Выход: [[1,4]]
Объяснение: [2,3] полностью внутри [1,4]
```

## Анализ задачи

**Ключевые наблюдения:**
- Если интервалы отсортированы по начальной позиции, проще находить пересечения
- Два интервала `[a, b]` и `[c, d]` пересекаются, если `c <= b` (при условии, что `a <= c`)
- Объединенный интервал: `[min(a, c), max(b, d)]`

**Условие пересечения:**
```
Интервал 1: [a, b]
Интервал 2: [c, d]

Если c <= b (при a <= c), то пересекаются
Результат: [a, max(b, d)]
```

**Edge cases:**
- Пустой массив → []
- Один интервал → возвращаем как есть
- Все интервалы пересекаются → один большой интервал
- Нет пересечений → все интервалы остаются

## Решение: Сортировка + проход

**Идея:** Отсортировать интервалы по начальной позиции, затем пройтись и объединять пересекающиеся.

**Алгоритм:**
1. Отсортировать интервалы по началу
2. Инициализировать результат первым интервалом
3. Для каждого следующего интервала:
   - Если пересекается с последним в результате → объединить
   - Иначе → добавить как отдельный интервал

**Сложность:** O(n log n) по времени (из-за сортировки), O(n) по памяти

```go
func merge(intervals [][]int) [][]int {
    if len(intervals) == 0 {
        return [][]int{}
    }

    // Сортируем по началу интервала
    sort.Slice(intervals, func(i, j int) bool {
        return intervals[i][0] < intervals[j][0]
    })

    // Результат начинаем с первого интервала
    result := [][]int{intervals[0]}

    for i := 1; i < len(intervals); i++ {
        current := intervals[i]
        lastMerged := result[len(result)-1]

        // Проверяем пересечение
        if current[0] <= lastMerged[1] {
            // Пересекаются - объединяем
            // Берем максимальный конец
            if current[1] > lastMerged[1] {
                lastMerged[1] = current[1]
            }
        } else {
            // Не пересекаются - добавляем как новый
            result = append(result, current)
        }
    }

    return result
}
```

## Пошаговое объяснение

Рассмотрим пример: `intervals = [[1,3],[2,6],[8,10],[15,18]]`

### Шаг 1: Сортировка
```
Уже отсортированы: [[1,3],[2,6],[8,10],[15,18]]
```

### Шаг 2: Инициализация
```
result = [[1,3]]
```

### Шаг 3: Обработка [2,6]
```
current = [2,6]
lastMerged = [1,3]

2 <= 3? Да! Пересекаются
Объединяем: [1, max(3, 6)] = [1,6]

result = [[1,6]]
```

### Шаг 4: Обработка [8,10]
```
current = [8,10]
lastMerged = [1,6]

8 <= 6? Нет! Не пересекаются
Добавляем отдельно

result = [[1,6], [8,10]]
```

### Шаг 5: Обработка [15,18]
```
current = [15,18]
lastMerged = [8,10]

15 <= 10? Нет! Не пересекаются
Добавляем отдельно

result = [[1,6], [8,10], [15,18]] ✅
```

## Визуализация

```
Исходные интервалы:
0    5    10   15   20
|-------|        [1,3]
  |----------|   [2,6]
         |---|   [8,10]
               |----| [15,18]

После объединения:
|----------|     [1,6]
         |---|   [8,10]
               |----| [15,18]
```

### Пример с полным перекрытием

```
Вход: [[1,4],[2,3]]

До:
0  1  2  3  4  5
   |--------|    [1,4]
      |--|       [2,3]

После:
   |--------|    [1,4]
```

### Пример с касанием границ

```
Вход: [[1,4],[4,5]]

До:
0  1  2  3  4  5
   |--------|    [1,4]
            |-|  [4,5]

После:
   |--------|-|  [1,5]
```

## Альтернативная реализация (in-place)

```go
func mergeInPlace(intervals [][]int) [][]int {
    if len(intervals) <= 1 {
        return intervals
    }

    sort.Slice(intervals, func(i, j int) bool {
        return intervals[i][0] < intervals[j][0]
    })

    writeIdx := 0

    for i := 1; i < len(intervals); i++ {
        if intervals[writeIdx][1] >= intervals[i][0] {
            // Объединяем с текущим
            intervals[writeIdx][1] = max(intervals[writeIdx][1], intervals[i][1])
        } else {
            // Переходим к следующему интервалу
            writeIdx++
            intervals[writeIdx] = intervals[i]
        }
    }

    return intervals[:writeIdx+1]
}

func max(a, b int) int {
    if a > b {
        return a
    }
    return b
}
```

**Преимущество:** Меньше аллокаций памяти.

## Тестовые случаи

```go
func TestMerge(t *testing.T) {
    tests := []struct {
        intervals [][]int
        expected  [][]int
    }{
        {
            intervals: [][]int{{1, 3}, {2, 6}, {8, 10}, {15, 18}},
            expected:  [][]int{{1, 6}, {8, 10}, {15, 18}},
        },
        {
            intervals: [][]int{{1, 4}, {4, 5}},
            expected:  [][]int{{1, 5}},
        },
        {
            intervals: [][]int{{1, 4}, {0, 4}},
            expected:  [][]int{{0, 4}},
        },
        {
            intervals: [][]int{{1, 4}, {2, 3}},
            expected:  [][]int{{1, 4}},
        },
        {
            intervals: [][]int{},
            expected:  [][]int{},
        },
        {
            intervals: [][]int{{1, 10}},
            expected:  [][]int{{1, 10}},
        },
        {
            intervals: [][]int{{1, 2}, {3, 4}, {5, 6}},
            expected:  [][]int{{1, 2}, {3, 4}, {5, 6}},
        },
        {
            intervals: [][]int{{1, 10}, {2, 3}, {4, 5}, {6, 7}},
            expected:  [][]int{{1, 10}},
        },
    }

    for _, tt := range tests {
        result := merge(tt.intervals)

        if !reflect.DeepEqual(result, tt.expected) {
            t.Errorf("merge(%v) = %v, expected %v",
                tt.intervals, result, tt.expected)
        }
    }
}
```

## Полное решение с обработкой ошибок

```go
package main

import (
    "fmt"
    "sort"
)

type Interval struct {
    Start int
    End   int
}

func mergeIntervals(intervals []Interval) []Interval {
    if len(intervals) == 0 {
        return []Interval{}
    }

    // Сортировка по началу
    sort.Slice(intervals, func(i, j int) bool {
        return intervals[i].Start < intervals[j].Start
    })

    result := []Interval{intervals[0]}

    for i := 1; i < len(intervals); i++ {
        last := &result[len(result)-1]
        current := intervals[i]

        if current.Start <= last.End {
            // Объединяем
            if current.End > last.End {
                last.End = current.End
            }
        } else {
            // Добавляем новый
            result = append(result, current)
        }
    }

    return result
}

func main() {
    intervals := []Interval{
        {1, 3},
        {2, 6},
        {8, 10},
        {15, 18},
    }

    merged := mergeIntervals(intervals)

    fmt.Println("Исходные интервалы:")
    for _, iv := range intervals {
        fmt.Printf("[%d, %d] ", iv.Start, iv.End)
    }

    fmt.Println("\n\nОбъединенные интервалы:")
    for _, iv := range merged {
        fmt.Printf("[%d, %d] ", iv.Start, iv.End)
    }
    fmt.Println()
}
```

## Вариации задачи

### 1. Insert Interval

Вставить новый интервал и объединить пересекающиеся:

```go
func insert(intervals [][]int, newInterval []int) [][]int {
    result := [][]int{}
    i := 0
    n := len(intervals)

    // Добавить все интервалы до нового
    for i < n && intervals[i][1] < newInterval[0] {
        result = append(result, intervals[i])
        i++
    }

    // Объединить пересекающиеся
    for i < n && intervals[i][0] <= newInterval[1] {
        newInterval[0] = min(newInterval[0], intervals[i][0])
        newInterval[1] = max(newInterval[1], intervals[i][1])
        i++
    }
    result = append(result, newInterval)

    // Добавить оставшиеся
    for i < n {
        result = append(result, intervals[i])
        i++
    }

    return result
}
```

### 2. Meeting Rooms

Можно ли посетить все встречи (нет перекрытий)?

```go
func canAttendMeetings(intervals [][]int) bool {
    sort.Slice(intervals, func(i, j int) bool {
        return intervals[i][0] < intervals[j][0]
    })

    for i := 1; i < len(intervals); i++ {
        if intervals[i][0] < intervals[i-1][1] {
            return false  // Перекрытие
        }
    }

    return true
}
```

### 3. Meeting Rooms II

Минимальное количество комнат для всех встреч:

```go
func minMeetingRooms(intervals [][]int) int {
    if len(intervals) == 0 {
        return 0
    }

    starts := make([]int, len(intervals))
    ends := make([]int, len(intervals))

    for i, interval := range intervals {
        starts[i] = interval[0]
        ends[i] = interval[1]
    }

    sort.Ints(starts)
    sort.Ints(ends)

    rooms := 0
    maxRooms := 0
    startPtr := 0
    endPtr := 0

    for startPtr < len(starts) {
        if starts[startPtr] < ends[endPtr] {
            rooms++
            startPtr++
        } else {
            rooms--
            endPtr++
        }
        maxRooms = max(maxRooms, rooms)
    }

    return maxRooms
}
```

## Где спрашивают

- **LeetCode:** #56 - Merge Intervals
- **Яндекс:** Популярная задача на алгоритмическом раунде
- **Авито:** Часто спрашивают вариацию с расписанием встреч
- **Т-Банк:** Могут спросить оптимизацию для больших данных
- **Озон:** Классическая задача с интервалами

## Вопросы с собеседований

**Вопрос 1:** Почему сначала нужно сортировать?

**Ответ:** Без сортировки пришлось бы сравнивать каждый интервал со всеми остальными O(n²). Сортировка позволяет за один проход O(n) после сортировки O(n log n) найти все пересечения.

---

**Вопрос 2:** Что делать, если интервалы уже отсортированы?

**Ответ:** Можно пропустить шаг сортировки, тогда сложность будет O(n) вместо O(n log n).

---

**Вопрос 3:** Как обрабатывать случай, когда один интервал полностью внутри другого?

**Ответ:** Наш алгоритм это учитывает - при объединении мы берем `max(b, d)` для конца интервала, что покрывает случай вложенности.

---

**Вопрос 4:** Можно ли решить без дополнительной памяти?

**Ответ:** Да, можно модифицировать входной массив in-place после сортировки, используя two-pointer технику для записи результата.

## Best Practices

1. ✅ Всегда сортируйте интервалы по началу
2. ✅ Проверяйте edge cases (пустой массив, один элемент)
3. ✅ Используйте `max()` для определения конца объединенного интервала
4. ✅ Рассматривайте касание границ как пересечение
5. ❌ Не забывайте обновлять конец при объединении
6. ❌ Помните, что интервал может быть полностью внутри другого

## Применение в реальности

### Объединение временных слотов

```go
type TimeSlot struct {
    Start time.Time
    End   time.Time
}

func mergeTimeSlots(slots []TimeSlot) []TimeSlot {
    if len(slots) == 0 {
        return []TimeSlot{}
    }

    sort.Slice(slots, func(i, j int) bool {
        return slots[i].Start.Before(slots[j].Start)
    })

    result := []TimeSlot{slots[0]}

    for i := 1; i < len(slots); i++ {
        last := &result[len(result)-1]
        current := slots[i]

        if current.Start.Before(last.End) || current.Start.Equal(last.End) {
            if current.End.After(last.End) {
                last.End = current.End
            }
        } else {
            result = append(result, current)
        }
    }

    return result
}
```

### Оптимизация диапазонов IP адресов

```go
type IPRange struct {
    Start uint32
    End   uint32
}

func mergeIPRanges(ranges []IPRange) []IPRange {
    // Аналогично merge intervals
    // Используется для оптимизации firewall rules
    // ...
}
```

## Связанные темы

- [[Сортировки - Быстрая, слиянием, пузырьком]]
- [[Алгоритмическая сложность (Big O)]]
- [[Go - Массивы и слайсы]]
- [[Два указателя (Two Pointers)]]
