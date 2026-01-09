# Задача - Implement Queue using Stacks

Реализовать очередь (FIFO) используя два стека (LIFO).

## Условие

Реализуйте очередь используя только две структуры данных "стек". Реализованная очередь должна поддерживать все функции обычной очереди (`push`, `pop`, `peek`, `empty`).

**Требования:**

Реализовать класс `MyQueue`:
- `void Push(x int)` - добавить элемент x в конец очереди
- `int Pop()` - удалить элемент из начала очереди и вернуть его
- `int Peek()` - вернуть элемент из начала очереди
- `bool Empty()` - вернуть true если очередь пуста

**Ограничения:**
- Использовать только стандартные операции стека: push, pop, peek/top, size, isEmpty

**Пример:**
```go
queue := Constructor()
queue.Push(1)
queue.Push(2)
queue.Peek()    // возвращает 1
queue.Pop()     // возвращает 1
queue.Empty()   // возвращает false
```

## Анализ задачи

**Ключевые наблюдения:**
- Стек: LIFO (Last In, First Out) - последний зашел, первый вышел
- Очередь: FIFO (First In, First Out) - первый зашел, первый вышел
- Нужно "перевернуть" порядок элементов стека

**Идея:**
- Один стек для входящих элементов (input)
- Второй стек для исходящих элементов (output)
- При Pop/Peek перекладываем из input в output (переворачивая порядок)

**Визуализация:**
```
Input Stack     Output Stack
(для Push)      (для Pop/Peek)

   [3]              [1]
   [2]              [2]
   [1]              [3]
    ↑                ↓
   Push             Pop

Переворачивание: перекладываем из input в output
```

## Решение: Два стека

**Сложность:**
- Push: O(1)
- Pop: амортизированная O(1)
- Peek: амортизированная O(1)
- Empty: O(1)

```go
type MyQueue struct {
    input  []int  // Стек для входящих элементов
    output []int  // Стек для исходящих элементов
}

func Constructor() MyQueue {
    return MyQueue{
        input:  []int{},
        output: []int{},
    }
}

// Push: O(1)
func (q *MyQueue) Push(x int) {
    // Просто добавляем в input стек
    q.input = append(q.input, x)
}

// Pop: амортизированная O(1)
func (q *MyQueue) Pop() int {
    // Перемещаем элементы из input в output если нужно
    q.moveInputToOutput()

    // Pop из output стека
    val := q.output[len(q.output)-1]
    q.output = q.output[:len(q.output)-1]

    return val
}

// Peek: амортизированная O(1)
func (q *MyQueue) Peek() int {
    q.moveInputToOutput()

    return q.output[len(q.output)-1]
}

// Empty: O(1)
func (q *MyQueue) Empty() bool {
    return len(q.input) == 0 && len(q.output) == 0
}

// Вспомогательная функция: перемещение из input в output
func (q *MyQueue) moveInputToOutput() {
    // Перемещаем только если output пуст
    if len(q.output) == 0 {
        for len(q.input) > 0 {
            // Pop из input
            val := q.input[len(q.input)-1]
            q.input = q.input[:len(q.input)-1]

            // Push в output
            q.output = append(q.output, val)
        }
    }
}
```

## Пошаговое объяснение

Рассмотрим последовательность операций:

### Операция 1: Push(1)
```
Input:  [1]
Output: []
```

### Операция 2: Push(2)
```
Input:  [1, 2]
Output: []
```

### Операция 3: Push(3)
```
Input:  [1, 2, 3]
Output: []
```

### Операция 4: Pop()
```
Сначала: moveInputToOutput()
  Перемещаем все из Input в Output:
  Pop 3 from Input → Push to Output
  Pop 2 from Input → Push to Output
  Pop 1 from Input → Push to Output

После перемещения:
Input:  []
Output: [3, 2, 1]

Pop из Output:
Input:  []
Output: [3, 2]
Возвращаем: 1 ✅
```

### Операция 5: Push(4)
```
Input:  [4]
Output: [3, 2]
```

### Операция 6: Pop()
```
Output не пуст, не нужно перемещать

Pop из Output:
Input:  [4]
Output: [3]
Возвращаем: 2 ✅
```

### Операция 7: Pop()
```
Pop из Output:
Input:  [4]
Output: []
Возвращаем: 3 ✅
```

### Операция 8: Pop()
```
Output пуст, перемещаем:
Input:  [] → Output: [4]

Pop из Output:
Input:  []
Output: []
Возвращаем: 4 ✅
```

## Визуализация

```
Последовательность: Push(1), Push(2), Push(3), Pop(), Pop()

Push(1):
Input:  [1]        Output: []

Push(2):
Input:  [1, 2]     Output: []

Push(3):
Input:  [1, 2, 3]  Output: []

Pop() - нужно переместить:
                   Перемещение:
Input:  [1, 2, 3]  →  Input: []
                      Output: [3, 2, 1]

                   Pop:
Input:  []            Output: [3, 2] → возвращаем 1

Pop() - output не пуст:
Input:  []            Output: [3] → возвращаем 2
```

## Амортизированная сложность

**Почему Pop() - амортизированная O(1)?**

Каждый элемент:
1. Push в input: 1 операция
2. Pop из input (при перемещении): 1 операция
3. Push в output (при перемещении): 1 операция
4. Pop из output: 1 операция

**Итого:** 4 операции на элемент → амортизированная O(1)

**Пример:**
```
n операций Push + n операций Pop
= n операций + 2n операций (перемещение) + n операций
= 4n операций
= O(n) на n элементов
= O(1) амортизированная на операцию
```

## Полное решение

```go
package main

import "fmt"

type MyQueue struct {
    input  []int
    output []int
}

func Constructor() MyQueue {
    return MyQueue{
        input:  make([]int, 0),
        output: make([]int, 0),
    }
}

func (q *MyQueue) Push(x int) {
    q.input = append(q.input, x)
}

func (q *MyQueue) Pop() int {
    q.moveInputToOutput()

    if len(q.output) == 0 {
        panic("queue is empty")
    }

    val := q.output[len(q.output)-1]
    q.output = q.output[:len(q.output)-1]
    return val
}

func (q *MyQueue) Peek() int {
    q.moveInputToOutput()

    if len(q.output) == 0 {
        panic("queue is empty")
    }

    return q.output[len(q.output)-1]
}

func (q *MyQueue) Empty() bool {
    return len(q.input) == 0 && len(q.output) == 0
}

func (q *MyQueue) moveInputToOutput() {
    if len(q.output) == 0 {
        for len(q.input) > 0 {
            val := q.input[len(q.input)-1]
            q.input = q.input[:len(q.input)-1]
            q.output = append(q.output, val)
        }
    }
}

func main() {
    queue := Constructor()

    queue.Push(1)
    queue.Push(2)
    queue.Push(3)

    fmt.Println("Peek:", queue.Peek())    // 1
    fmt.Println("Pop:", queue.Pop())       // 1
    fmt.Println("Pop:", queue.Pop())       // 2
    fmt.Println("Empty:", queue.Empty())   // false
    fmt.Println("Pop:", queue.Pop())       // 3
    fmt.Println("Empty:", queue.Empty())   // true
}
```

## Тестовые случаи

```go
func TestMyQueue(t *testing.T) {
    queue := Constructor()

    // Тест 1: Push и Pop
    queue.Push(1)
    queue.Push(2)

    if queue.Pop() != 1 {
        t.Error("Expected 1")
    }

    queue.Push(3)

    if queue.Pop() != 2 {
        t.Error("Expected 2")
    }

    if queue.Pop() != 3 {
        t.Error("Expected 3")
    }

    // Тест 2: Empty
    if !queue.Empty() {
        t.Error("Queue should be empty")
    }

    // Тест 3: Peek
    queue.Push(10)
    queue.Push(20)

    if queue.Peek() != 10 {
        t.Error("Expected 10")
    }

    if queue.Peek() != 10 {  // Peek не удаляет
        t.Error("Peek should not remove element")
    }

    // Тест 4: Чередование Push и Pop
    queue = Constructor()
    for i := 1; i <= 100; i++ {
        queue.Push(i)
        if i%2 == 0 {
            queue.Pop()
        }
    }
}
```

## Альтернативная реализация с проверками

```go
type MyQueue struct {
    input  []int
    output []int
}

func Constructor() MyQueue {
    return MyQueue{}
}

func (q *MyQueue) Push(x int) {
    q.input = append(q.input, x)
}

func (q *MyQueue) Pop() (int, bool) {
    if q.Empty() {
        return 0, false
    }

    q.moveInputToOutput()
    val := q.output[len(q.output)-1]
    q.output = q.output[:len(q.output)-1]

    return val, true
}

func (q *MyQueue) Peek() (int, bool) {
    if q.Empty() {
        return 0, false
    }

    q.moveInputToOutput()
    return q.output[len(q.output)-1], true
}

func (q *MyQueue) Empty() bool {
    return len(q.input) == 0 && len(q.output) == 0
}

func (q *MyQueue) Size() int {
    return len(q.input) + len(q.output)
}

func (q *MyQueue) moveInputToOutput() {
    if len(q.output) == 0 {
        for len(q.input) > 0 {
            val := q.input[len(q.input)-1]
            q.input = q.input[:len(q.input)-1]
            q.output = append(q.output, val)
        }
    }
}
```

## Вариации задачи

### 1. Implement Stack using Queues

Обратная задача - реализовать стек используя очереди:

```go
type MyStack struct {
    q1 []int
    q2 []int
}

func (s *MyStack) Push(x int) {
    // Push в q1
    s.q1 = append(s.q1, x)
}

func (s *MyStack) Pop() int {
    // Перемещаем все кроме последнего в q2
    for len(s.q1) > 1 {
        val := s.q1[0]
        s.q1 = s.q1[1:]
        s.q2 = append(s.q2, val)
    }

    // Pop последний элемент
    val := s.q1[0]
    s.q1 = s.q1[1:]

    // Swap q1 и q2
    s.q1, s.q2 = s.q2, s.q1

    return val
}
```

### 2. Min Queue

Очередь с операцией GetMin() за O(1):

```go
type MinQueue struct {
    queue MyQueue
    mins  []int  // Монотонная очередь для минимумов
}

func (mq *MinQueue) Push(x int) {
    mq.queue.Push(x)

    // Удаляем элементы больше x из конца
    for len(mq.mins) > 0 && mq.mins[len(mq.mins)-1] > x {
        mq.mins = mq.mins[:len(mq.mins)-1]
    }

    mq.mins = append(mq.mins, x)
}

func (mq *MinQueue) Pop() int {
    val := mq.queue.Pop()

    if len(mq.mins) > 0 && mq.mins[0] == val {
        mq.mins = mq.mins[1:]
    }

    return val
}

func (mq *MinQueue) GetMin() int {
    return mq.mins[0]
}
```

## Где спрашивают

- **LeetCode:** #232 - Implement Queue using Stacks
- **LeetCode:** #225 - Implement Stack using Queues
- **Яндекс:** Популярная задача на понимание структур данных
- **Авито:** Часто на первом техническом раунде
- **Т-Банк:** Просят объяснить амортизированную сложность
- **Озон:** Классическая задача на стеки

## Вопросы с собеседований

**Вопрос 1:** Почему сложность Pop() амортизированная O(1)?

**Ответ:** Хотя однократное перемещение стоит O(n), каждый элемент перемещается максимум один раз. На n операций получается O(n) работы, то есть O(1) в среднем на операцию.

---

**Вопрос 2:** Можно ли реализовать с одним стеком?

**Ответ:** Нет, нужно минимум два стека чтобы "переворачивать" порядок элементов без рекурсии.

---

**Вопрос 3:** Что если нужна thread-safe очередь?

**Ответ:** Добавить `sync.Mutex` для защиты доступа к стекам:

```go
type MyQueue struct {
    mu     sync.Mutex
    input  []int
    output []int
}

func (q *MyQueue) Push(x int) {
    q.mu.Lock()
    defer q.mu.Unlock()
    q.input = append(q.input, x)
}
```

---

**Вопрос 4:** В чем разница между этим подходом и обычной реализацией очереди?

**Ответ:** Обычная очередь (с кольцевым буфером или linked list) имеет O(1) worst-case для всех операций. Здесь Pop() имеет O(n) worst-case, но амортизированная O(1).

## Best Practices

1. ✅ Перемещайте элементы только когда output пуст
2. ✅ Проверяйте пустоту перед Pop/Peek
3. ✅ Используйте амортизированный анализ для оценки сложности
4. ✅ Рассмотрите использование встроенных container/list для production кода
5. ❌ Не перемещайте элементы при каждом Pop (неэффективно)
6. ❌ Не забывайте про граничные случаи (пустая очередь)

## Применение в реальности

### Обработка задач

```go
type TaskQueue struct {
    queue MyQueue
}

type Task struct {
    ID   int
    Data string
}

func (tq *TaskQueue) AddTask(task Task) {
    tq.queue.Push(task.ID)
}

func (tq *TaskQueue) ProcessNext() (Task, bool) {
    if tq.queue.Empty() {
        return Task{}, false
    }

    taskID := tq.queue.Pop()
    // Загружаем задачу из БД
    task := loadTaskFromDB(taskID)
    return task, true
}
```

### Буферизация сообщений

```go
type MessageBuffer struct {
    buffer MyQueue
    maxSize int
}

func (mb *MessageBuffer) Add(msg string) error {
    if mb.buffer.Size() >= mb.maxSize {
        return errors.New("buffer full")
    }

    mb.buffer.Push(hashMessage(msg))
    return nil
}

func (mb *MessageBuffer) GetNext() (string, bool) {
    if mb.buffer.Empty() {
        return "", false
    }

    msgHash := mb.buffer.Pop()
    msg := reconstructMessage(msgHash)
    return msg, true
}
```

## Связанные темы

- [[Стек и очередь]]
- [[Амортизированная сложность]]
- [[Go - Массивы и слайсы]]
- [[Алгоритмическая сложность (Big O)]]
