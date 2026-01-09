# Go - Пакет fmt

Пакет для форматированного ввода-вывода.

## Print функции

```go
// Print - без перевода строки
fmt.Print("Hello")
fmt.Print("World")  // HelloWorld

// Println - с переводом строки
fmt.Println("Hello")
fmt.Println("World")
// Hello
// World

// Printf - с форматированием
name := "Alice"
age := 25
fmt.Printf("Name: %s, Age: %d\n", name, age)
// Name: Alice, Age: 25
```

## Sprint функции (возвращают string)

```go
// Sprint
s := fmt.Sprint("Hello", "World")  // "HelloWorld"

// Sprintln
s := fmt.Sprintln("Hello", "World")  // "Hello World\n"

// Sprintf
s := fmt.Sprintf("Name: %s, Age: %d", "Alice", 25)
// "Name: Alice, Age: 25"
```

## Fprint функции (в io.Writer)

```go
// В файл
file, _ := os.Create("output.txt")
fmt.Fprintf(file, "Hello, %s!", "World")

// В буфер
var buf bytes.Buffer
fmt.Fprintf(&buf, "Value: %d", 42)
```

## Основные форматы

```go
// Целые числа
fmt.Printf("%d", 42)      // 42 (десятичное)
fmt.Printf("%b", 42)      // 101010 (двоичное)
fmt.Printf("%o", 42)      // 52 (восьмеричное)
fmt.Printf("%x", 42)      // 2a (шестнадцатеричное)
fmt.Printf("%X", 42)      // 2A (заглавное hex)

// Вещественные числа
fmt.Printf("%f", 3.14159)    // 3.141590 (по умолчанию 6 знаков)
fmt.Printf("%.2f", 3.14159)  // 3.14
fmt.Printf("%e", 1234.5)     // 1.234500e+03 (научная нотация)
fmt.Printf("%g", 1234.5)     // 1234.5 (компактная форма)

// Строки
fmt.Printf("%s", "text")     // text
fmt.Printf("%q", "text")     // "text" (в кавычках)
fmt.Printf("%10s", "text")   // "      text" (ширина 10)
fmt.Printf("%-10s", "text")  // "text      " (выравнивание влево)

// Булевы
fmt.Printf("%t", true)       // true

// Указатели
p := &x
fmt.Printf("%p", p)          // 0xc000010230

// Типы
fmt.Printf("%T", 42)         // int
fmt.Printf("%T", "text")     // string

// Универсальный формат
fmt.Printf("%v", anything)   // Значение в формате по умолчанию
fmt.Printf("%+v", struct)    // С именами полей структуры
fmt.Printf("%#v", anything)  // Go-syntax представление
```

## Примеры форматирования

```go
type Person struct {
    Name string
    Age  int
}

p := Person{Name: "Alice", Age: 25}

fmt.Printf("%v\n", p)   // {Alice 25}
fmt.Printf("%+v\n", p)  // {Name:Alice Age:25}
fmt.Printf("%#v\n", p)  // main.Person{Name:"Alice", Age:25}
```

## Scan функции (ввод)

```go
// Scan - разделитель пробел/перевод строки
var name string
var age int
fmt.Scan(&name, &age)

// Scanln - до перевода строки
fmt.Scanln(&name, &age)

// Scanf - с форматом
fmt.Scanf("%s %d", &name, &age)
```

## Errorf - создание ошибки

```go
err := fmt.Errorf("user not found: %d", userID)

// С wrapping (Go 1.13+)
err := fmt.Errorf("failed to get user: %w", originalErr)
```

## Stringer интерфейс

```go
type Stringer interface {
    String() string
}

// Пример
type Point struct {
    X, Y int
}

func (p Point) String() string {
    return fmt.Sprintf("(%d, %d)", p.X, p.Y)
}

p := Point{X: 1, Y: 2}
fmt.Println(p)  // (1, 2)
```

## Ширина и точность

```go
// Ширина
fmt.Printf("%5d", 42)      // "   42"
fmt.Printf("%-5d", 42)     // "42   "
fmt.Printf("%05d", 42)     // "00042"

// Точность
fmt.Printf("%.2f", 3.14159)   // "3.14"
fmt.Printf("%5.2f", 3.14159)  // " 3.14"
```

## Связанные темы

- [[Go - Основы синтаксиса]]
- [[Go - Строки и UTF-8]]
- [[Go - Обработка ошибок]]
