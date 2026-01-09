# Go - Указатели - Подводные камни

Типичные вопросы с собеседований про указатели, адреса и память в Go.

## Основы указателей

```go
var x int = 42
var p *int = &x  // p хранит адрес x

fmt.Println(x)   // 42 (значение)
fmt.Println(&x)  // 0xc0000140a0 (адрес)
fmt.Println(p)   // 0xc0000140a0 (адрес в указателе)
fmt.Println(*p)  // 42 (разыменование)

*p = 100
fmt.Println(x)   // 100 (изменили через указатель)
```

## Вопрос 1: Что выведет этот код?

```go
func main() {
    x := 10
    p := &x
    q := &x

    *p = 20

    fmt.Println(x)   // ?
    fmt.Println(*p)  // ?
    fmt.Println(*q)  // ?
}
```

<details>
<summary>Ответ</summary>

```
20
20
20
```

**Почему:** `p` и `q` указывают на один адрес (переменную `x`). Изменение через `*p` меняет `x`, что видно через оба указателя.
</details>

## Вопрос 2: Указатели в функциях

```go
func increment(n int) {
    n++
}

func incrementPtr(n *int) {
    *n++
}

func main() {
    x := 10
    increment(x)
    fmt.Println(x)  // ?

    incrementPtr(&x)
    fmt.Println(x)  // ?
}
```

<details>
<summary>Ответ</summary>

```
10
11
```

**Почему:**
- `increment(x)` получает **копию** значения → изменения не влияют на `x`
- `incrementPtr(&x)` получает **адрес** → изменяет оригинальную переменную
</details>

## Вопрос 3: Указатели на slice

```go
func appendValue(s []int, val int) {
    s = append(s, val)
}

func appendValuePtr(s *[]int, val int) {
    *s = append(*s, val)
}

func main() {
    nums := []int{1, 2, 3}

    appendValue(nums, 4)
    fmt.Println(nums)  // ?

    appendValuePtr(&nums, 4)
    fmt.Println(nums)  // ?
}
```

<details>
<summary>Ответ</summary>

```
[1 2 3]
[1 2 3 4]
```

**Почему:**
- Slice содержит указатель на underlying array, но **сам slice передаётся по значению**
- `append` может создать новый underlying array → изменение локальной копии `s`
- С указателем на slice меняем оригинальный slice
</details>

## Вопрос 4: Range и указатели

```go
type Person struct {
    Name string
}

func main() {
    people := []Person{
        {Name: "Alice"},
        {Name: "Bob"},
    }

    var pointers []*Person

    // Вариант 1
    for _, p := range people {
        pointers = append(pointers, &p)
    }

    for _, ptr := range pointers {
        fmt.Println(ptr.Name)  // ?
    }
}
```

<details>
<summary>Ответ</summary>

```
Bob
Bob
```

**Почему:** **Классическая ошибка!**
- `p` в `range` - это **одна переменная**, переиспользуемая на каждой итерации
- `&p` всегда даёт один и тот же адрес
- В конце `p` содержит последний элемент (Bob)

**Правильно:**
```go
for i := range people {
    pointers = append(pointers, &people[i])
}
```

или

```go
for _, p := range people {
    p := p  // Создать локальную копию
    pointers = append(pointers, &p)
}
```
</details>

## Вопрос 5: Nil указатель

```go
func process(p *int) {
    if p != nil {
        *p = 100
    }
}

func main() {
    var p *int
    process(p)

    fmt.Println(p == nil)  // ?

    x := 10
    p = &x
    process(p)
    fmt.Println(x)  // ?
}
```

<details>
<summary>Ответ</summary>

```
true
100
```

**Почему:**
- `var p *int` создаёт nil указатель (zero value для указателя)
- `process(p)` с nil безопасна благодаря проверке
- После `p = &x`, указатель валиден
</details>

## Вопрос 6: Методы с pointer receiver

```go
type Counter struct {
    count int
}

func (c Counter) Increment() {
    c.count++
}

func (c *Counter) IncrementPtr() {
    c.count++
}

func main() {
    c := Counter{count: 0}

    c.Increment()
    fmt.Println(c.count)  // ?

    c.IncrementPtr()
    fmt.Println(c.count)  // ?
}
```

<details>
<summary>Ответ</summary>

```
0
1
```

**Почему:**
- `Increment()` с value receiver → получает копию → изменяет копию
- `IncrementPtr()` с pointer receiver → изменяет оригинал
- Go автоматически берёт адрес: `c.IncrementPtr()` → `(&c).IncrementPtr()`
</details>

## Вопрос 7: Указатель на указатель

```go
func main() {
    x := 42
    p := &x
    pp := &p

    **pp = 100

    fmt.Println(x)   // ?
    fmt.Println(*p)  // ?
    fmt.Println(**pp) // ?
}
```

<details>
<summary>Ответ</summary>

```
100
100
100
```

**Почему:**
- `pp` → указатель на указатель
- `**pp` → дважды разыменовали → получили `x`
- Изменение через `**pp` меняет `x`
</details>

## Вопрос 8: Escape analysis

```go
func createInt() *int {
    x := 42
    return &x
}

func main() {
    p := createInt()
    fmt.Println(*p)  // ?
}
```

<details>
<summary>Ответ</summary>

```
42
```

**Почему:**
- В C/C++ это undefined behavior (возврат адреса локальной переменной)
- В Go **escape analysis** определяет, что `x` используется вне функции
- `x` автоматически аллоцируется в heap, не в stack
- Код безопасен!

Проверка:
```bash
go build -gcflags="-m" main.go
# ./main.go:2:6: moved to heap: x
```
</details>

## Вопрос 9: Сравнение указателей

```go
func main() {
    x := 10
    y := 10

    px := &x
    py := &y

    fmt.Println(px == py)  // ?
    fmt.Println(*px == *py) // ?

    pz := &x
    fmt.Println(px == pz)  // ?
}
```

<details>
<summary>Ответ</summary>

```
false
true
true
```

**Почему:**
- `px == py` → сравниваем адреса → разные переменные → false
- `*px == *py` → сравниваем значения → оба 10 → true
- `pz` указывает на тот же `x` что и `px` → true
</details>

## Вопрос 10: Указатели в struct

```go
type Node struct {
    Value int
    Next  *Node
}

func main() {
    n1 := Node{Value: 1}
    n2 := Node{Value: 2}
    n3 := Node{Value: 3}

    n1.Next = &n2
    n2.Next = &n3

    fmt.Println(n1.Next.Next.Value)  // ?

    // Что если?
    var head *Node
    fmt.Println(head.Next)  // ?
}
```

<details>
<summary>Ответ</summary>

```
3
<nil>
```

**Почему:**
- `n1.Next.Next.Value` → `(&n2).Next.Value` → `(&n3).Value` → 3
- `head.Next` с nil указателем → **panic: nil pointer dereference**

**Правильно:**
```go
if head != nil {
    fmt.Println(head.Next)
}
```
</details>

## Вопрос 11: Изменение map через указатель

```go
func modifyMap(m map[string]int) {
    m["key"] = 100
}

func modifyMapPtr(m *map[string]int) {
    (*m)["key"] = 200
}

func main() {
    m := map[string]int{"key": 1}

    modifyMap(m)
    fmt.Println(m["key"])  // ?

    modifyMapPtr(&m)
    fmt.Println(m["key"])  // ?
}
```

<details>
<summary>Ответ</summary>

```
100
200
```

**Почему:**
- Map - это **reference type** (как slice)
- Уже содержит указатель на underlying data
- Изменения видны без указателя на map
- Указатель на map нужен только если хотите переназначить сам map (`*m = make(...)`)
</details>

## Вопрос 12: Копирование struct с указателем

```go
type Person struct {
    Name string
    Age  *int
}

func main() {
    age := 30
    p1 := Person{Name: "Alice", Age: &age}
    p2 := p1

    *p2.Age = 40

    fmt.Println(*p1.Age)  // ?
    fmt.Println(*p2.Age)  // ?
}
```

<details>
<summary>Ответ</summary>

```
40
40
```

**Почему:**
- `p2 := p1` → **shallow copy**
- Копируются значения полей
- `Age` - это указатель → копируется адрес (не значение!)
- `p1.Age` и `p2.Age` указывают на одну переменную `age`

**Для deep copy:**
```go
p2 := p1
newAge := *p1.Age
p2.Age = &newAge
```
</details>

## Вопрос 13: Адреса в array vs slice

```go
func main() {
    arr := [3]int{1, 2, 3}
    slice := []int{1, 2, 3}

    fmt.Printf("%p\n", &arr)
    fmt.Printf("%p\n", &arr[0])

    fmt.Printf("%p\n", &slice)
    fmt.Printf("%p\n", &slice[0])

    // Что означают эти адреса?
}
```

<details>
<summary>Ответ</summary>

```
0xc000014078  // Адрес массива
0xc000014078  // Адрес первого элемента (тот же!)

0xc00000c030  // Адрес slice header
0xc000014090  // Адрес underlying array
```

**Почему:**
- Array: `&arr` = `&arr[0]` (массив - это его первый элемент)
- Slice: `&slice` = адрес slice header (struct с ptr, len, cap)
- `&slice[0]` = адрес underlying array
</details>

## Вопрос 14: new vs make

```go
func main() {
    // new
    p1 := new(int)
    fmt.Println(p1)   // ?
    fmt.Println(*p1)  // ?

    // make
    s := make([]int, 0)
    fmt.Println(s)    // ?

    // Можно ли make(int)?
}
```

<details>
<summary>Ответ</summary>

```
0xc0000140a0  // Адрес
0             // Zero value для int

[]            // Пустой slice

Нет, make только для slice, map, chan
```

**Различия:**
- `new(T)` → возвращает `*T`, инициализирует zero value
- `make(T, ...)` → возвращает `T` (не указатель), только для slice/map/chan
</details>

## Вопрос 15: Указатель в интерфейсе

```go
type Printer interface {
    Print()
}

type Document struct {
    content string
}

func (d *Document) Print() {
    fmt.Println(d.content)
}

func main() {
    var p Printer

    // Вариант 1
    d := Document{content: "Hello"}
    p = d  // Компилируется?

    // Вариант 2
    p = &d  // Компилируется?
}
```

<details>
<summary>Ответ</summary>

**Вариант 1:** ❌ Ошибка компиляции
```
cannot use d (type Document) as type Printer:
Document does not implement Printer (Print method has pointer receiver)
```

**Вариант 2:** ✅ Компилируется

**Почему:**
- Метод с pointer receiver: только `*Document` реализует интерфейс
- `Document` (value) **не реализует** интерфейс
- Нужен `p = &d`

**Правило:**
- Pointer receiver → только pointer реализует интерфейс
- Value receiver → и value, и pointer реализуют интерфейс
</details>

## Практические задачи

### Задача 1: Reverse Linked List

```go
type ListNode struct {
    Val  int
    Next *ListNode
}

// Развернуть список
func reverseList(head *ListNode) *ListNode {
    // Ваш код
}

// Тест
// Input:  1 → 2 → 3 → 4 → nil
// Output: 4 → 3 → 2 → 1 → nil
```

<details>
<summary>Решение</summary>

```go
func reverseList(head *ListNode) *ListNode {
    var prev *ListNode
    current := head

    for current != nil {
        next := current.Next    // Сохранить next
        current.Next = prev     // Развернуть указатель
        prev = current          // Сдвинуть prev
        current = next          // Сдвинуть current
    }

    return prev
}
```
</details>

### Задача 2: Swap Nodes

```go
// Поменять местами значения двух узлов
func swapNodes(a, b *ListNode) {
    // Ваш код
}

// Тест
n1 := &ListNode{Val: 1}
n2 := &ListNode{Val: 2}
swapNodes(n1, n2)
// n1.Val = 2, n2.Val = 1
```

<details>
<summary>Решение</summary>

```go
func swapNodes(a, b *ListNode) {
    a.Val, b.Val = b.Val, a.Val
}

// Или если нужно поменять сами узлы (не значения)
func swapNodePointers(prev1, prev2 *ListNode) {
    if prev1 == nil || prev2 == nil {
        return
    }

    node1 := prev1.Next
    node2 := prev2.Next

    if node1 == nil || node2 == nil {
        return
    }

    prev1.Next = node2
    prev2.Next = node1

    node1.Next, node2.Next = node2.Next, node1.Next
}
```
</details>

### Задача 3: Deep Copy

```go
type Node struct {
    Val    int
    Next   *Node
    Random *Node
}

// Глубокое копирование списка с random указателями
func copyRandomList(head *Node) *Node {
    // Ваш код
}
```

<details>
<summary>Решение</summary>

```go
func copyRandomList(head *Node) *Node {
    if head == nil {
        return nil
    }

    // Этап 1: Создать копии и вставить после оригиналов
    current := head
    for current != nil {
        copy := &Node{Val: current.Val}
        copy.Next = current.Next
        current.Next = copy
        current = copy.Next
    }

    // Этап 2: Установить random указатели
    current = head
    for current != nil {
        if current.Random != nil {
            current.Next.Random = current.Random.Next
        }
        current = current.Next.Next
    }

    // Этап 3: Разделить списки
    current = head
    newHead := head.Next
    for current != nil {
        copy := current.Next
        current.Next = copy.Next
        if copy.Next != nil {
            copy.Next = copy.Next.Next
        }
        current = current.Next
    }

    return newHead
}
```
</details>

## Memory Leaks с указателями

```go
// ❌ Memory leak
type Cache struct {
    data map[string]*LargeObject
}

func (c *Cache) Set(key string, obj *LargeObject) {
    c.data[key] = obj  // Объект не освободится пока в map
}

// ✅ Правильно
func (c *Cache) Delete(key string) {
    delete(c.data, key)  // Явно удалить
}

// Или использовать weak references (но в Go нет встроенных)
```

## Best Practices

1. ✅ Используйте pointer receiver если:
   - Метод модифицирует receiver
   - Struct большая (избегаем копирования)
   - Нужна консистентность (все методы с pointer receiver)

2. ✅ Проверяйте nil перед разыменованием
```go
if ptr != nil {
    value := *ptr
}
```

3. ✅ Range: берите индекс, не элемент
```go
for i := range items {
    ptr := &items[i]  // ✅
}

for _, item := range items {
    ptr := &item  // ❌ Всегда один адрес
}
```

4. ✅ Избегайте указателей на slice/map/chan (уже reference types)

5. ✅ Используйте `new()` для аллокации zero value с указателем

6. ❌ Не возвращайте адреса локальных переменных в C стиле (Go безопасен благодаря escape analysis)

7. ❌ Не храните указатели на loop variables

## Связанные темы

- [[Go - Указатели и ссылки]]
- [[Go - Управление памятью]]
- [[Go - Garbage Collector]]
- [[Связные списки]]
- [[Go - Интерфейсы]]
