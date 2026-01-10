# Heap (куча)

Heap - это complete binary tree (завершенное бинарное дерево) со свойством упорядоченности.

## Типы кучи

### Max Heap (максимальная куча)

Родитель ≥ дети

```
      50
     /  \
   30    40
   / \   /
  10 20 15

Корень - максимальный элемент
```

### Min Heap (минимальная куча)

Родитель ≤ дети

```
      10
     /  \
   20    15
   / \   /
  30 40 50

Корень - минимальный элемент
```

## Представление в массиве

Heap хранится в массиве без указателей!

```
       10
      /  \
    20    30
   / \    /
  40 50  60

Массив: [10, 20, 30, 40, 50, 60]
Индексы: 0   1   2   3   4   5
```

### Формулы для навигации

Для узла с индексом `i`:

```go
parent := (i - 1) / 2
leftChild := 2*i + 1
rightChild := 2*i + 2
```

**Пример:**
- Элемент `30` (индекс 2)
  - Parent: (2-1)/2 = 0 → `10`
  - Left: 2*2+1 = 5 → `60`
  - Right: 2*2+2 = 6 → не существует

## Реализация Min Heap

```go
type MinHeap struct {
    items []int
}

func NewMinHeap() *MinHeap {
    return &MinHeap{items: []int{}}
}

func (h *MinHeap) parent(i int) int {
    return (i - 1) / 2
}

func (h *MinHeap) leftChild(i int) int {
    return 2*i + 1
}

func (h *MinHeap) rightChild(i int) int {
    return 2*i + 2
}

func (h *MinHeap) swap(i, j int) {
    h.items[i], h.items[j] = h.items[j], h.items[i]
}

func (h *MinHeap) Size() int {
    return len(h.items)
}

func (h *MinHeap) IsEmpty() bool {
    return len(h.items) == 0
}

func (h *MinHeap) Peek() (int, error) {
    if h.IsEmpty() {
        return 0, errors.New("heap is empty")
    }
    return h.items[0], nil
}
```

### Вставка (Push)

1. Добавить элемент в конец
2. Bubble up (просеивание вверх)

```go
func (h *MinHeap) Push(value int) {
    h.items = append(h.items, value)
    h.bubbleUp(len(h.items) - 1)
}

func (h *MinHeap) bubbleUp(i int) {
    for i > 0 {
        parent := h.parent(i)

        // Min heap: если родитель больше, меняем
        if h.items[parent] > h.items[i] {
            h.swap(parent, i)
            i = parent
        } else {
            break
        }
    }
}
```

**Пример:**

```
Вставить 5:

    10          10          5
   /  \        /  \        / \
  20  30  →  20   5   →  20  10
 /          /  \            /  \
40         40   30         40   30

1. Добавить в конец
2. Bubble up: 5 < 30, swap
3. Bubble up: 5 < 10, swap
```

**Сложность:** O(log n)

### Удаление минимума (Pop)

1. Взять корень (минимум)
2. Заменить корень последним элементом
3. Bubble down (просеивание вниз)

```go
func (h *MinHeap) Pop() (int, error) {
    if h.IsEmpty() {
        return 0, errors.New("heap is empty")
    }

    min := h.items[0]

    // Заменить корень последним элементом
    lastIdx := len(h.items) - 1
    h.items[0] = h.items[lastIdx]
    h.items = h.items[:lastIdx]

    // Bubble down
    if !h.IsEmpty() {
        h.bubbleDown(0)
    }

    return min, nil
}

func (h *MinHeap) bubbleDown(i int) {
    for {
        left := h.leftChild(i)
        right := h.rightChild(i)
        smallest := i

        // Найти наименьшего из трёх
        if left < len(h.items) && h.items[left] < h.items[smallest] {
            smallest = left
        }

        if right < len(h.items) && h.items[right] < h.items[smallest] {
            smallest = right
        }

        // Если i уже наименьший, готово
        if smallest == i {
            break
        }

        h.swap(i, smallest)
        i = smallest
    }
}
```

**Пример:**

```
Pop из:

     10           60           20
    /  \         /  \         /  \
   20  30  →   20   30  →   60  30
  /  \         /           /
 40  60       40          40

1. Взять 10 (min)
2. Заменить на 60 (последний)
3. Bubble down: 60 > 20, swap
4. Bubble down: 60 > 40, swap
```

**Сложность:** O(log n)

## Heapify - создание кучи из массива

```go
func NewMinHeapFromArray(arr []int) *MinHeap {
    h := &MinHeap{items: make([]int, len(arr))}
    copy(h.items, arr)

    // Heapify снизу вверх
    // Начинаем с последнего родителя
    for i := len(h.items)/2 - 1; i >= 0; i-- {
        h.bubbleDown(i)
    }

    return h
}
```

**Сложность:** O(n) (не O(n log n)!)

**Почему O(n)?**
- Листья (половина узлов) не двигаются: 0 операций
- Предпоследний уровень: 1 операция каждый
- И так далее...
- Суммарно: O(n)

## Max Heap

```go
type MaxHeap struct {
    items []int
}

func (h *MaxHeap) bubbleUp(i int) {
    for i > 0 {
        parent := (i - 1) / 2

        // Max heap: если родитель меньше, меняем
        if h.items[parent] < h.items[i] {
            h.items[parent], h.items[i] = h.items[i], h.items[parent]
            i = parent
        } else {
            break
        }
    }
}

func (h *MaxHeap) bubbleDown(i int) {
    for {
        left := 2*i + 1
        right := 2*i + 2
        largest := i

        if left < len(h.items) && h.items[left] > h.items[largest] {
            largest = left
        }

        if right < len(h.items) && h.items[right] > h.items[largest] {
            largest = right
        }

        if largest == i {
            break
        }

        h.items[i], h.items[largest] = h.items[largest], h.items[i]
        i = largest
    }
}
```

## Сложность операций

| Операция | Сложность |
|----------|-----------|
| Peek | O(1) |
| Push | O(log n) |
| Pop | O(log n) |
| Heapify | O(n) |
| Поиск | O(n) |

## Go container/heap

Go предоставляет пакет `container/heap` с интерфейсом.

```go
import "container/heap"

type IntHeap []int

func (h IntHeap) Len() int           { return len(h) }
func (h IntHeap) Less(i, j int) bool { return h[i] < h[j] }  // Min heap
func (h IntHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }

func (h *IntHeap) Push(x interface{}) {
    *h = append(*h, x.(int))
}

func (h *IntHeap) Pop() interface{} {
    old := *h
    n := len(old)
    x := old[n-1]
    *h = old[0 : n-1]
    return x
}

// Использование
h := &IntHeap{}
heap.Init(h)

heap.Push(h, 3)
heap.Push(h, 1)
heap.Push(h, 5)

fmt.Println(heap.Pop(h))  // 1 (минимум)
```

### Max Heap через container/heap

Поменять `Less`:

```go
func (h IntHeap) Less(i, j int) bool {
    return h[i] > h[j]  // > вместо <
}
```

### Priority Queue

```go
type Item struct {
    value    string
    priority int
    index    int
}

type PriorityQueue []*Item

func (pq PriorityQueue) Len() int { return len(pq) }

func (pq PriorityQueue) Less(i, j int) bool {
    return pq[i].priority < pq[j].priority  // Меньше = выше приоритет
}

func (pq PriorityQueue) Swap(i, j int) {
    pq[i], pq[j] = pq[j], pq[i]
    pq[i].index = i
    pq[j].index = j
}

func (pq *PriorityQueue) Push(x interface{}) {
    n := len(*pq)
    item := x.(*Item)
    item.index = n
    *pq = append(*pq, item)
}

func (pq *PriorityQueue) Pop() interface{} {
    old := *pq
    n := len(old)
    item := old[n-1]
    item.index = -1
    *pq = old[0 : n-1]
    return item
}

// Использование
pq := &PriorityQueue{}
heap.Init(pq)

heap.Push(pq, &Item{value: "task1", priority: 5})
heap.Push(pq, &Item{value: "task2", priority: 1})
heap.Push(pq, &Item{value: "task3", priority: 3})

item := heap.Pop(pq).(*Item)
fmt.Println(item.value)  // "task2" (priority 1)
```

## Применение

### 1. Kth Largest Element

```go
func findKthLargest(nums []int, k int) int {
    h := &IntHeap{}  // Min heap
    heap.Init(h)

    for _, num := range nums {
        heap.Push(h, num)

        // Поддерживаем размер k
        if h.Len() > k {
            heap.Pop(h)
        }
    }

    return (*h)[0]  // Минимум в куче = k-й по величине
}

// findKthLargest([3,2,1,5,6,4], 2) → 5
```

### 2. Merge K Sorted Lists

```go
type ListNode struct {
    Val  int
    Next *ListNode
}

type NodeHeap []*ListNode

func (h NodeHeap) Len() int           { return len(h) }
func (h NodeHeap) Less(i, j int) bool { return h[i].Val < h[j].Val }
func (h NodeHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }

func (h *NodeHeap) Push(x interface{}) {
    *h = append(*h, x.(*ListNode))
}

func (h *NodeHeap) Pop() interface{} {
    old := *h
    n := len(old)
    x := old[n-1]
    *h = old[0 : n-1]
    return x
}

func mergeKLists(lists []*ListNode) *ListNode {
    h := &NodeHeap{}
    heap.Init(h)

    // Добавить первый элемент каждого списка
    for _, list := range lists {
        if list != nil {
            heap.Push(h, list)
        }
    }

    dummy := &ListNode{}
    current := dummy

    for h.Len() > 0 {
        node := heap.Pop(h).(*ListNode)
        current.Next = node
        current = current.Next

        if node.Next != nil {
            heap.Push(h, node.Next)
        }
    }

    return dummy.Next
}
```

### 3. Top K Frequent Elements

```go
func topKFrequent(nums []int, k int) []int {
    // Подсчитать частоты
    freq := make(map[int]int)
    for _, num := range nums {
        freq[num]++
    }

    // Min heap по частоте
    h := &FreqHeap{}
    heap.Init(h)

    for num, count := range freq {
        heap.Push(h, &FreqItem{num: num, freq: count})

        if h.Len() > k {
            heap.Pop(h)
        }
    }

    // Извлечь результаты
    result := make([]int, k)
    for i := k - 1; i >= 0; i-- {
        result[i] = heap.Pop(h).(*FreqItem).num
    }

    return result
}

type FreqItem struct {
    num  int
    freq int
}

type FreqHeap []*FreqItem

func (h FreqHeap) Len() int           { return len(h) }
func (h FreqHeap) Less(i, j int) bool { return h[i].freq < h[j].freq }
func (h FreqHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }

func (h *FreqHeap) Push(x interface{}) {
    *h = append(*h, x.(*FreqItem))
}

func (h *FreqHeap) Pop() interface{} {
    old := *h
    n := len(old)
    item := old[n-1]
    *h = old[0 : n-1]
    return item
}
```

### 4. Median from Data Stream

```go
type MedianFinder struct {
    maxHeap *MaxHeap  // Левая половина (меньшие числа)
    minHeap *MinHeap  // Правая половина (большие числа)
}

func (mf *MedianFinder) AddNum(num int) {
    // Добавить в maxHeap (левая половина)
    heap.Push(mf.maxHeap, num)

    // Балансировка: убедиться что max(left) <= min(right)
    if mf.maxHeap.Len() > 0 && mf.minHeap.Len() > 0 {
        if mf.maxHeap.Peek() > mf.minHeap.Peek() {
            val := heap.Pop(mf.maxHeap).(int)
            heap.Push(mf.minHeap, val)
        }
    }

    // Балансировка размеров
    if mf.maxHeap.Len() > mf.minHeap.Len()+1 {
        val := heap.Pop(mf.maxHeap).(int)
        heap.Push(mf.minHeap, val)
    }

    if mf.minHeap.Len() > mf.maxHeap.Len() {
        val := heap.Pop(mf.minHeap).(int)
        heap.Push(mf.maxHeap, val)
    }
}

func (mf *MedianFinder) FindMedian() float64 {
    if mf.maxHeap.Len() > mf.minHeap.Len() {
        return float64(mf.maxHeap.Peek())
    }

    return float64(mf.maxHeap.Peek()+mf.minHeap.Peek()) / 2.0
}
```

### 5. Heap Sort

```go
func heapSort(arr []int) []int {
    h := &IntHeap{}
    heap.Init(h)

    // Push all
    for _, num := range arr {
        heap.Push(h, num)
    }

    // Pop all (sorted)
    result := make([]int, len(arr))
    for i := 0; i < len(arr); i++ {
        result[i] = heap.Pop(h).(int)
    }

    return result
}

// Сложность: O(n log n)
// Память: O(n)
```

## Heap vs BST

| Свойство | Heap | BST |
|----------|------|-----|
| Поиск min/max | O(1) | O(log n) |
| Вставка | O(log n) | O(log n) |
| Удаление min/max | O(log n) | O(log n) |
| Поиск элемента | O(n) | O(log n) |
| Inorder traversal | ❌ | ✅ отсортирован |
| Память | Массив | Узлы с указателями |

**Используйте Heap когда:**
- Нужен быстрый доступ к min/max
- Priority queue
- Heap sort
- Top K элементов

**Используйте BST когда:**
- Нужна полная упорядоченность
- Range queries
- Поиск произвольного элемента

## Best Practices

1. ✅ Используйте container/heap в Go
2. ✅ Min heap для k largest, Max heap для k smallest
3. ✅ Heapify за O(n) вместо n insertions за O(n log n)
4. ✅ Heap отлично для streaming данных
5. ❌ Не используйте heap для поиска произвольного элемента
6. ❌ Heap не поддерживает эффективное обновление элементов

## Связанные темы

- [[Стек и очередь]]
- [[Бинарные деревья поиска]]
- [[Деревья - Основы]]
- [[Сортировки - Быстрая, слиянием, пузырьком]]
- [[Алгоритмическая сложность (Big O)]]
