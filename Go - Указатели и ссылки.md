# Go - Указатели и ссылки

Указатели - это переменные, которые хранят адрес памяти другой переменной. Понимание указателей критически важно для эффективной работы с Go.

## Основы указателей

### Объявление и использование

```go
var x int = 42

var p *int     // Указатель на int
p = &x         // & - оператор взятия адреса

fmt.Println(p)   // 0xc000012028 (адрес)
fmt.Println(*p)  // 42 (значение по адресу)

*p = 21          // Изменение через указатель
fmt.Println(x)   // 21 (x изменился!)
```

**Операторы:**
- `&` - взятие адреса (address-of)
- `*` - разыменование (dereference)

### Zero value

```go
var p *int
fmt.Println(p) // nil

// ❌ Разыменование nil - panic!
// fmt.Println(*p) // panic: invalid memory address

// ✅ Проверка на nil
if p != nil {
    fmt.Println(*p)
}
```

### Создание через new()

```go
p := new(int)  // Создает int и возвращает *int
*p = 42
fmt.Println(*p) // 42
```

## Value vs Pointer Receivers

### Value receiver

```go
type Counter struct {
    count int
}

// Value receiver - получает копию
func (c Counter) Increment() {
    c.count++ // Изменяется копия!
}

c := Counter{count: 0}
c.Increment()
fmt.Println(c.count) // 0 (не изменилось!)
```

### Pointer receiver

```go
// Pointer receiver - получает указатель
func (c *Counter) Increment() {
    c.count++ // Изменяется оригинал
}

c := Counter{count: 0}
c.Increment()
fmt.Println(c.count) // 1 (изменилось!)

// Go автоматически берет адрес
c.Increment() // Эквивалентно (&c).Increment()
```

**Когда использовать pointer receiver:**
- ✅ Метод изменяет receiver
- ✅ Receiver - большая структура (избегаем копирования)
- ✅ Консистентность (если один метод с pointer, все должны быть)

## Передача в функции

### By Value

```go
func increment(x int) {
    x++ // Изменяется копия
}

n := 10
increment(n)
fmt.Println(n) // 10 (не изменилось)
```

### By Pointer

```go
func increment(x *int) {
    *x++ // Изменяется оригинал
}

n := 10
increment(&n)
fmt.Println(n) // 11 (изменилось!)
```

### Слайсы, мапы, каналы - особый случай

```go
// Слайс передается по значению, но содержит указатель на массив!
func modify(s []int) {
    s[0] = 999 // Изменяет оригинальный массив
}

nums := []int{1, 2, 3}
modify(nums)
fmt.Println(nums) // [999 2 3]

// Но изменение header'а слайса не влияет на оригинал
func appendValue(s []int) {
    s = append(s, 4) // Изменяет копию header'а
}

nums := []int{1, 2, 3}
appendValue(nums)
fmt.Println(nums) // [1 2 3] (не изменилось!)

// ✅ Передаем указатель на слайс если нужно изменить header
func appendValue(s *[]int) {
    *s = append(*s, 4)
}
```

Подробнее: [[Go - Массивы и слайсы]]

## Указатели и структуры

### Автоматическое разыменование

```go
type Point struct {
    X, Y int
}

p := &Point{X: 1, Y: 2}

// Оба варианта работают одинаково:
fmt.Println(p.X)    // 1
fmt.Println((*p).X) // 1

// Go автоматически разыменовывает для доступа к полям
```

### Composite literals

```go
// Создание указателя на структуру
p1 := &Point{X: 1, Y: 2}
// Эквивалентно:
p2 := new(Point)
p2.X = 1
p2.Y = 2
```

## Практические задачи

### Задача 1: Swap

```go
// ❌ Не работает
func swap(a, b int) {
    temp := a
    a = b
    b = temp
}

x, y := 1, 2
swap(x, y)
fmt.Println(x, y) // 1 2 (не поменялись!)

// ✅ Работает
func swap(a, b *int) {
    temp := *a
    *a = *b
    *b = temp
}

x, y := 1, 2
swap(&x, &y)
fmt.Println(x, y) // 2 1 (поменялись!)

// ✅ Идиоматичный Go - множественное присваивание
func swap(a, b int) (int, int) {
    return b, a
}

x, y := 1, 2
x, y = swap(x, y)
```

**Объяснение**: Без указателей функция работает с копиями, изменения не влияют на оригинальные переменные.

### Задача 2: Linked List Node

```go
type Node struct {
    Value int
    Next  *Node // Указатель на следующий узел
}

// Создание списка
head := &Node{Value: 1}
head.Next = &Node{Value: 2}
head.Next.Next = &Node{Value: 3}

// Обход списка
current := head
for current != nil {
    fmt.Println(current.Value)
    current = current.Next
}
```

**Объяснение**: Указатели необходимы для рекурсивных структур данных.

### Задача 3: Изменение элемента в цикле

```go
type User struct {
    Name string
    Age  int
}

users := []User{
    {Name: "Alice", Age: 25},
    {Name: "Bob", Age: 30},
}

// ❌ Не изменяет оригинальные элементы
for _, user := range users {
    user.Age++ // Изменяется копия!
}
fmt.Println(users[0].Age) // 25 (не изменилось)

// ✅ Через индекс
for i := range users {
    users[i].Age++
}

// ✅ Через указатель (Go 1.22+)
for i, user := range users {
    users[i] = user
    users[i].Age++
}

// ✅ Слайс указателей
userPtrs := []*User{
    {Name: "Alice", Age: 25},
    {Name: "Bob", Age: 30},
}

for _, user := range userPtrs {
    user.Age++ // Изменяет оригинал
}
```

**Объяснение**: `range` копирует значения, нужно использовать индекс или указатели.

### Задача 4: Метод изменяющий структуру

```go
type BankAccount struct {
    balance float64
}

// ❌ Не работает - value receiver
func (b BankAccount) Deposit(amount float64) {
    b.balance += amount // Изменяет копию!
}

account := BankAccount{balance: 100}
account.Deposit(50)
fmt.Println(account.balance) // 100 (не изменилось!)

// ✅ Работает - pointer receiver
func (b *BankAccount) Deposit(amount float64) {
    b.balance += amount
}

account := BankAccount{balance: 100}
account.Deposit(50)
fmt.Println(account.balance) // 150
```

**Объяснение**: Для изменения receiver нужен pointer receiver.

### Задача 5: Escaping to heap

```go
// ❌ Возвращаем указатель на локальную переменную
func createUser() *User {
    user := User{Name: "John"} // Создается на стеке
    return &user // Escape to heap! Go перемещает на кучу
}

// ✅ Валидно в Go (в отличие от C/C++!)
u := createUser()
fmt.Println(u.Name) // Работает!
```

**Объяснение**: Go автоматически определяет нужно ли размещать на куче (escape analysis).

### Задача 6: Pointer to pointer

```go
func modifyPointer(p **int) {
    x := 100
    *p = &x // Меняем куда указывает p
}

a := 42
p := &a
fmt.Println(*p) // 42

modifyPointer(&p)
fmt.Println(*p) // 100
```

**Объяснение**: `**int` - указатель на указатель, позволяет изменить сам указатель.

### Задача 7: Nil pointer in struct

```go
type Config struct {
    Host *string // Nullable field
    Port int
}

func getConfig() Config {
    host := "localhost"
    return Config{
        Host: &host, // Передаем указатель
        Port: 8080,
    }
}

cfg := getConfig()
if cfg.Host != nil {
    fmt.Println(*cfg.Host)
}

// Для nil значений
var cfg2 Config
if cfg2.Host == nil {
    fmt.Println("Host not set")
}
```

**Объяснение**: Указатели позволяют различить "не установлено" (nil) и "установлено в zero value".

### Задача 8: Circular reference

```go
type Person struct {
    Name   string
    Friend *Person
}

alice := &Person{Name: "Alice"}
bob := &Person{Name: "Bob"}

alice.Friend = bob
bob.Friend = alice // Циклическая ссылка

fmt.Println(alice.Friend.Friend.Name) // Alice
```

**Объяснение**: Указатели позволяют создавать циклические структуры.

## Частые ошибки

### 1. Разыменование nil

```go
// ❌ Panic!
var p *int
fmt.Println(*p) // panic: invalid memory address

// ✅ Проверка
if p != nil {
    fmt.Println(*p)
}
```

### 2. Изменение копии

```go
// ❌ Изменяется копия
func modify(u User) {
    u.Name = "Changed"
}

user := User{Name: "Original"}
modify(user)
fmt.Println(user.Name) // Original

// ✅ Через указатель
func modify(u *User) {
    u.Name = "Changed"
}

modify(&user)
fmt.Println(user.Name) // Changed
```

### 3. Сравнение указателей на структуры

```go
p1 := &User{Name: "John"}
p2 := &User{Name: "John"}

// Сравниваются адреса, не содержимое!
fmt.Println(p1 == p2) // false (разные адреса)

// Для сравнения содержимого
fmt.Println(*p1 == *p2) // true
```

### 4. Указатель в range loop

```go
users := []User{{Name: "A"}, {Name: "B"}}

// ❌ Все указатели на одну переменную!
var ptrs []*User
for _, u := range users {
    ptrs = append(ptrs, &u) // u - одна переменная цикла!
}

for _, p := range ptrs {
    fmt.Println(p.Name) // B, B (оба указывают на последнее значение)
}

// ✅ Через индекс
var ptrs []*User
for i := range users {
    ptrs = append(ptrs, &users[i])
}

// ✅ Копия в цикле
for _, u := range users {
    u := u // Создаем копию
    ptrs = append(ptrs, &u)
}
```

## Производительность

### Stack vs Heap

```go
// Размещается на стеке (быстро)
func onStack() {
    x := 42
    _ = x
}

// Размещается на куче (медленнее, нагрузка на GC)
func onHeap() *int {
    x := 42
    return &x // Escape to heap
}
```

**Benchmark:**
```
Stack allocation: ~0.3 ns
Heap allocation:  ~10-20 ns
```

**Совет**: Go сам определяет где размещать (escape analysis). Не нужно оптимизировать преждевременно.

### Копирование больших структур

```go
type BigStruct struct {
    data [1000]int
}

// ❌ Копирует 8KB на каждый вызов
func processByValue(b BigStruct) {
    // ...
}

// ✅ Передает 8 bytes (указатель)
func processByPointer(b *BigStruct) {
    // ...
}
```

**Правило**: Если структура > 3-4 полей или содержит большие массивы, используйте указатель.

## Best Practices

### 1. Используйте value receiver по умолчанию

```go
// ✅ Small types - value receiver
func (p Point) Distance() float64 {
    return math.Sqrt(float64(p.X*p.X + p.Y*p.Y))
}

// ✅ Large types или mutation - pointer receiver
func (u *User) UpdateEmail(email string) {
    u.Email = email
}
```

### 2. Будьте консистентны

```go
// ❌ Смешанные receivers
func (u User) GetName() string    { return u.Name }
func (u *User) SetName(n string)  { u.Name = n }

// ✅ Все pointer receivers для консистентности
func (u *User) GetName() string   { return u.Name }
func (u *User) SetName(n string)  { u.Name = n }
```

### 3. Nil-safe методы

```go
// ✅ Методы работают с nil receiver
type Tree struct {
    Value int
    Left  *Tree
    Right *Tree
}

func (t *Tree) Sum() int {
    if t == nil {
        return 0
    }
    return t.Value + t.Left.Sum() + t.Right.Sum()
}

var tree *Tree
fmt.Println(tree.Sum()) // 0 (не panic!)
```

### 4. Документируйте ownership

```go
// Cache stores pointers to values.
// The caller must not modify the values after storing.
type Cache struct {
    items map[string]*Item
}

func (c *Cache) Store(key string, item *Item) {
    c.items[key] = item // Кто owner?
}
```

## Связанные темы

- [[Go - Структуры (struct)]]
- [[Go - Функции и методы]]
- [[Go - Массивы и слайсы]]
- [[Go - Управление памятью]]
- [[Go - Garbage Collector]]
- [[Связные списки]]
