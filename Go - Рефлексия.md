# Go - Рефлексия

Рефлексия (reflection) - способность программы исследовать свою структуру во время выполнения.

## Пакет reflect

```go
import "reflect"
```

### TypeOf - получить тип

```go
var x int = 42
t := reflect.TypeOf(x)
fmt.Println(t)        // int
fmt.Println(t.Kind()) // int
fmt.Println(t.Name()) // int

// Для структур
type Person struct {
    Name string
    Age  int
}

p := Person{"Alice", 25}
t = reflect.TypeOf(p)
fmt.Println(t)        // main.Person
fmt.Println(t.Kind()) // struct
fmt.Println(t.Name()) // Person
```

### ValueOf - получить значение

```go
var x int = 42
v := reflect.ValueOf(x)
fmt.Println(v)          // 42
fmt.Println(v.Type())   // int
fmt.Println(v.Kind())   // int
fmt.Println(v.Int())    // 42
```

## Kind vs Type

```go
type MyInt int

var x MyInt = 42

t := reflect.TypeOf(x)
fmt.Println(t.Kind()) // int (базовый тип)
fmt.Println(t.Name()) // MyInt (имя типа)
```

**Kind** - базовая категория (int, string, struct, slice, etc.)
**Type** - конкретный тип (MyInt, Person, etc.)

## Работа со структурами

### Получить поля

```go
type Person struct {
    Name string `json:"name"`
    Age  int    `json:"age"`
}

p := Person{"Alice", 25}
t := reflect.TypeOf(p)

// Количество полей
numFields := t.NumField()

// Итерация по полям
for i := 0; i < numFields; i++ {
    field := t.Field(i)
    fmt.Printf("Field %d: %s %s\n", i, field.Name, field.Type)
    fmt.Printf("Tag: %s\n", field.Tag.Get("json"))
}
```

### Изменить значения

```go
p := Person{"Alice", 25}
v := reflect.ValueOf(&p).Elem() // Нужен указатель!

// Изменить поле Name
nameField := v.FieldByName("Name")
if nameField.CanSet() {
    nameField.SetString("Bob")
}

fmt.Println(p.Name) // Bob
```

**Важно:** Для изменения нужен указатель и вызов Elem()!

## Работа со слайсами

```go
slice := []int{1, 2, 3, 4, 5}
v := reflect.ValueOf(slice)

fmt.Println(v.Kind())   // slice
fmt.Println(v.Len())    // 5
fmt.Println(v.Index(0)) // 1

// Итерация
for i := 0; i < v.Len(); i++ {
    fmt.Println(v.Index(i).Int())
}

// Создать новый слайс
sliceType := reflect.TypeOf(slice)
newSlice := reflect.MakeSlice(sliceType, 0, 10)
```

## Работа с map

```go
m := map[string]int{"a": 1, "b": 2, "c": 3}
v := reflect.ValueOf(m)

// Итерация
iter := v.MapRange()
for iter.Next() {
    key := iter.Key()
    value := iter.Value()
    fmt.Printf("%s: %d\n", key, value)
}

// Получить значение
key := reflect.ValueOf("a")
value := v.MapIndex(key)
fmt.Println(value.Int()) // 1
```

## Вызов функций

```go
func add(a, b int) int {
    return a + b
}

// Получить функцию
fn := reflect.ValueOf(add)

// Подготовить аргументы
args := []reflect.Value{
    reflect.ValueOf(10),
    reflect.ValueOf(20),
}

// Вызвать
results := fn.Call(args)
fmt.Println(results[0].Int()) // 30
```

## Создание значений

```go
// Создать int
v := reflect.New(reflect.TypeOf(42))
v.Elem().SetInt(100)
fmt.Println(v.Elem().Int()) // 100

// Создать struct
t := reflect.TypeOf(Person{})
v = reflect.New(t)
person := v.Interface().(*Person)
person.Name = "Alice"
```

## Interface{}

```go
var x interface{} = 42

// Type assertion с рефлексией
v := reflect.ValueOf(x)

switch v.Kind() {
case reflect.Int:
    fmt.Println("Int:", v.Int())
case reflect.String:
    fmt.Println("String:", v.String())
case reflect.Struct:
    fmt.Println("Struct")
}
```

## Практические примеры

### Generic Print функция

```go
func Print(v interface{}) {
    val := reflect.ValueOf(v)
    typ := reflect.TypeOf(v)

    switch val.Kind() {
    case reflect.Struct:
        fmt.Printf("%s {\n", typ.Name())
        for i := 0; i < val.NumField(); i++ {
            field := typ.Field(i)
            value := val.Field(i)
            fmt.Printf("  %s: %v\n", field.Name, value)
        }
        fmt.Println("}")

    case reflect.Slice:
        fmt.Print("[")
        for i := 0; i < val.Len(); i++ {
            if i > 0 {
                fmt.Print(", ")
            }
            fmt.Print(val.Index(i))
        }
        fmt.Println("]")

    default:
        fmt.Println(val)
    }
}
```

### Struct to Map

```go
func StructToMap(obj interface{}) map[string]interface{} {
    result := make(map[string]interface{})

    v := reflect.ValueOf(obj)
    t := reflect.TypeOf(obj)

    for i := 0; i < v.NumField(); i++ {
        field := t.Field(i)
        value := v.Field(i)

        // Используем json tag если есть
        name := field.Tag.Get("json")
        if name == "" {
            name = field.Name
        }

        result[name] = value.Interface()
    }

    return result
}
```

### Deep Equal

```go
func DeepEqual(a, b interface{}) bool {
    return reflect.DeepEqual(a, b)
}

// Сравнивает любые типы глубоко
slice1 := []int{1, 2, 3}
slice2 := []int{1, 2, 3}
fmt.Println(DeepEqual(slice1, slice2)) // true
```

### Copy struct

```go
func CopyStruct(src, dst interface{}) error {
    srcVal := reflect.ValueOf(src).Elem()
    dstVal := reflect.ValueOf(dst).Elem()

    if srcVal.Type() != dstVal.Type() {
        return errors.New("types don't match")
    }

    for i := 0; i < srcVal.NumField(); i++ {
        dstVal.Field(i).Set(srcVal.Field(i))
    }

    return nil
}
```

## Когда использовать рефлексию

✅ **Используйте:**
- JSON/XML marshal/unmarshal
- ORM (database/sql)
- Валидация
- Dependency Injection
- Testing frameworks
- Сериализация

❌ **Избегайте:**
- Обычный application код
- Когда есть альтернатива через интерфейсы
- Performance-critical секции

## Производительность

Рефлексия **медленнее** обычного кода:

```go
// ❌ Медленно с рефлексией
v := reflect.ValueOf(x)
result := v.Int() + 10

// ✅ Быстро без рефлексии
result := x + 10
```

**Используйте рефлексию только когда действительно нужно!**

## Ограничения

```go
// ❌ Нельзя изменить не экспортированные поля
type Person struct {
    name string // Строчная буква
}

p := Person{"Alice"}
v := reflect.ValueOf(&p).Elem()
nameField := v.FieldByName("name")
// nameField.SetString("Bob") // Panic!

// ❌ Нельзя получить Value без указателя для изменения
v := reflect.ValueOf(p) // Копия!
// v.Field(0).SetString("Bob") // Panic!

// ✅ Нужен указатель
v := reflect.ValueOf(&p).Elem()
```

## Type assertions vs Reflection

```go
var x interface{} = 42

// Type assertion - быстро
if val, ok := x.(int); ok {
    fmt.Println(val + 10)
}

// Рефлексия - медленно
v := reflect.ValueOf(x)
if v.Kind() == reflect.Int {
    fmt.Println(v.Int() + 10)
}
```

**Предпочитайте type assertions когда возможно!**

## Best Practices

1. ✅ Используйте рефлексию только когда необходимо
2. ✅ Проверяйте CanSet() перед изменением
3. ✅ Обрабатывайте ошибки (многие операции могут паниковать)
4. ✅ Кэшируйте результаты TypeOf/ValueOf если используете многократно
5. ✅ Используйте интерфейсы вместо рефлексии где возможно
6. ❌ Не используйте рефлексию в hot paths
7. ❌ Не используйте рефлексию для обычного application кода

## Связанные темы

- [[Go - Интерфейсы]]
- [[Go - Теги структур]]
- [[Go - Структуры (struct)]]
