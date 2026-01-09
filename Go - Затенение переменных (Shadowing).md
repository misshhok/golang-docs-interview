# Go - Затенение переменных (Shadowing)

Shadowing (затенение) возникает, когда переменная во внутреннем scope объявлена с тем же именем, что и переменная во внешнем scope, "перекрывая" её. Это одна из самых частых ошибок в Go, особенно с оператором `:=`.

## Основная концепция

```go
x := 1
fmt.Println(x) // 1

if true {
    x := 2  // Shadowing! Новая переменная x
    fmt.Println(x) // 2
}

fmt.Println(x) // 1 (оригинальная переменная не изменилась)
```

**Что произошло:**
- `:=` создаёт **новую** переменную `x` внутри блока `if`
- Внутренняя `x` "затеняет" внешнюю
- После выхода из блока внешняя `x` остаётся без изменений

## Вопрос 1: Что выведет этот код?

```go
func main() {
    count := 0

    for i := 0; i < 5; i++ {
        count := count + 1
        fmt.Print(count, " ")
    }

    fmt.Println("\nФинальное значение:", count)
}
```

<details>
<summary>Ответ</summary>

```
1 1 1 1 1
Финальное значение: 0
```

**Почему:**
- `count := count + 1` создаёт **новую** переменную `count` в scope цикла
- Правая часть читает внешний `count` (всегда 0)
- Левая часть создаёт локальный `count` = 1
- Внешний `count` никогда не изменяется

**Правильно:**
```go
count = count + 1  // Использовать =, не :=
```
</details>

## Вопрос 2: Классическая ловушка с err

```go
func processData() error {
    data, err := fetchData()
    if err != nil {
        return err
    }

    if needsValidation(data) {
        data, err := validateData(data)  // Что здесь не так?
        if err != nil {
            return err
        }
        // Работа с data
    }

    return saveData(data)  // Какой data мы сохраним?
}
```

<details>
<summary>Ответ</summary>

**Проблема:** `data, err := validateData(data)` создаёт **новые** локальные переменные `data` и `err` внутри блока `if`.

**Что происходит:**
- Валидированный `data` существует только внутри блока `if`
- `saveData(data)` получает **оригинальный** невалидированный `data`
- Это баг!

**Правильные варианты:**

```go
// Вариант 1: Использовать присваивание вместо объявления
if needsValidation(data) {
    data, err = validateData(data)  // = вместо :=
    if err != nil {
        return err
    }
}

// Вариант 2: Разные имена переменных
if needsValidation(data) {
    validatedData, err := validateData(data)
    if err != nil {
        return err
    }
    data = validatedData
}
```

**Почему `:=` создаёт новую переменную?**
- `:=` создаёт новую переменную, если хотя бы одна из переменных слева новая
- `data, err := ...` - обе переменные существуют во внешнем scope, но `:=` создаёт новые в текущем scope
</details>

## Вопрос 3: Shadowing параметров функции

```go
func updateUser(user *User) error {
    if user == nil {
        user := &User{}  // Что не так?
        user.Name = "Default"
    }

    return db.Save(user)
}
```

<details>
<summary>Ответ</summary>

**Проблема:** `user := &User{}` создаёт **новую** локальную переменную `user`, которая затеняет параметр функции.

**Что происходит:**
- Локальный `user` инициализируется и изменяется
- После выхода из блока `if` используется оригинальный `user` (nil)
- `db.Save(user)` получает nil → ошибка

**Правильно:**
```go
func updateUser(user *User) error {
    if user == nil {
        user = &User{}  // = вместо :=
        user.Name = "Default"
    }

    return db.Save(user)
}
```
</details>

## Вопрос 4: Shadowing в defer

```go
func process() (err error) {
    err = doSomething()

    defer func() {
        if err != nil {
            err := fmt.Errorf("wrapped: %w", err)
            log.Println(err)
        }
    }()

    return err  // Какой err вернётся?
}
```

<details>
<summary>Ответ</summary>

**Проблема:** `err := fmt.Errorf(...)` создаёт **новую** переменную `err` внутри defer, затеняя named return value.

**Что происходит:**
- Оборачивание ошибки происходит в локальной переменной
- Named return value `err` остаётся без изменений
- Функция возвращает необёрнутую ошибку

**Правильно:**
```go
defer func() {
    if err != nil {
        err = fmt.Errorf("wrapped: %w", err)  // = вместо :=
        log.Println(err)
    }
}()
```

**Named return values** в defer позволяют модифицировать возвращаемое значение, но нужно использовать `=`, не `:=`.
</details>

## Shadowing с одинаковыми именами в разных блоках

```go
func example() {
    x := "outer"

    if true {
        x := "if block"
        fmt.Println(x)
    }

    for i := 0; i < 1; i++ {
        x := "for block"
        fmt.Println(x)
    }

    switch {
    case true:
        x := "case block"
        fmt.Println(x)
    }

    fmt.Println(x)
}
```

<details>
<summary>Ответ</summary>

```
if block
for block
case block
outer
```

Каждый блок создаёт свою собственную переменную `x`, затеняющую внешнюю. После выхода из блока видна оригинальная переменная.
</details>

## Обнаружение shadowing с go vet

Go предоставляет инструменты для обнаружения shadowing:

```bash
# Установить shadow checker
go install golang.org/x/tools/go/analysis/passes/shadow/cmd/shadow@latest

# Проверить код
go vet -vettool=$(which shadow) ./...
```

Пример вывода:
```
./main.go:15:3: declaration of "err" shadows declaration at line 12
```

### Популярные линтеры с проверкой shadowing

```bash
# golangci-lint с включенным shadow
golangci-lint run --enable=govet --enable=shadow
```

## Вопрос 5: Частичное shadowing

```go
func multiReturn() (int, error) {
    x, err := firstCall()
    if err != nil {
        return 0, err
    }

    // Вариант 1
    x, err := secondCall()  // Ошибка компиляции?

    // Вариант 2
    y, err := secondCall()  // Ошибка компиляции?

    return x, nil
}
```

<details>
<summary>Ответ</summary>

**Вариант 1:** ❌ Ошибка компиляции
```
no new variables on left side of :=
```
Обе переменные `x` и `err` уже объявлены в текущем scope. Нельзя использовать `:=`.

**Вариант 2:** ✅ Компилируется
```go
y, err := secondCall()  // y - новая, err - переиспользуется
```

**Правило `:=`:**
- Можно использовать, если **хотя бы одна** переменная слева новая
- Новые переменные создаются
- Существующие переменные переиспользуются (если в том же scope)

**Но осторожно с scope:**
```go
if true {
    y, err := secondCall()  // Создаёт НОВЫЙ err внутри if (shadowing!)
}
```
</details>

## Shadowing встроенных типов и функций

```go
// ❌ Плохая практика
func process() {
    true := false  // Shadowing встроенной константы
    len := 10      // Shadowing встроенной функции
    error := "oops" // Shadowing встроенного типа

    arr := []int{1, 2, 3}
    fmt.Println(len(arr))  // Ошибка: len это int, не функция
}
```

Go позволяет затенять встроенные идентификаторы, но это **очень плохая практика**:
- `true`, `false`, `nil`
- `int`, `string`, `error`, `bool`
- `len`, `cap`, `make`, `append`
- `print`, `println`

## Практический пример: HTTP Handler

```go
// ❌ С ошибкой shadowing
func handleRequest(w http.ResponseWriter, r *http.Request) {
    user, err := getUser(r)
    if err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    // Нужна дополнительная валидация
    if user.Premium {
        user, err := getPremiumData(user.ID)  // Shadowing!
        if err != nil {
            http.Error(w, err.Error(), http.StatusInternalServerError)
            return
        }
        // user здесь - с премиум данными
    }

    // user здесь - БЕЗ премиум данных (оригинальный)
    json.NewEncoder(w).Encode(user)
}

// ✅ Правильно
func handleRequest(w http.ResponseWriter, r *http.Request) {
    user, err := getUser(r)
    if err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    if user.Premium {
        user, err = getPremiumData(user.ID)  // = вместо :=
        if err != nil {
            http.Error(w, err.Error(), http.StatusInternalServerError)
            return
        }
    }

    json.NewEncoder(w).Encode(user)
}
```

## Best Practices

1. ✅ **Используйте `=` вместо `:=` если переменная уже существует**
   ```go
   data, err := fetchData()
   data, err = processData(data)  // ✅
   ```

2. ✅ **Давайте разные имена при необходимости**
   ```go
   user, err := getUser(id)
   admin, err := getAdmin(id)  // Разные имена
   ```

3. ✅ **Включите shadow checker в CI/CD**
   ```yaml
   # .golangci.yml
   linters:
     enable:
       - govet
       - shadow
   ```

4. ✅ **Проверяйте named return values в defer**
   ```go
   func process() (err error) {
       defer func() {
           if err != nil {
               err = fmt.Errorf("wrapped: %w", err)  // = не :=
           }
       }()
   }
   ```

5. ❌ **Не затеняйте встроенные идентификаторы**
   ```go
   len := 10      // ❌ Ужасная практика
   length := 10   // ✅ Используйте другое имя
   ```

6. ✅ **Минимизируйте scope переменных**
   ```go
   // ❌ Широкий scope
   var result int
   if condition {
       result = calculate()
   }

   // ✅ Узкий scope
   if condition {
       result := calculate()
       // Используем result только здесь
   }
   ```

7. ✅ **Используйте IDE с поддержкой shadowing warnings**
   - GoLand/IntelliJ IDEA
   - VS Code с gopls
   - Vim/Neovim с LSP

## Вопросы с собеседований

**Вопрос 1:** В чём разница между `:=` и `=`?

**Ответ:**
- `:=` - краткое объявление (short variable declaration), создаёт новую переменную
- `=` - присваивание существующей переменной
- `:=` можно использовать, если хотя бы одна переменная слева новая в текущем scope

**Вопрос 2:** Почему этот код не работает?
```go
var err error
data, err := fetchData()  // Ошибка: err already declared
```

**Ответ:** `var err error` объявляет `err` в текущем scope. Затем `:=` пытается создать новую переменную `err`, но получает конфликт. Нужно использовать `=`:
```go
var err error
data, err = fetchData()
```

Или просто:
```go
data, err := fetchData()  // err создаётся здесь
```

**Вопрос 3:** Как обнаружить shadowing в существующем коде?

**Ответ:**
- `go vet` с shadow checker
- `golangci-lint` с включенным shadow linter
- Встроенные проверки IDE
- Code review практики

## Связанные темы

- [[Go - Переменные и константы]]
- [[Go - Указатели - Подводные камни]]
- [[Go - Область видимости (scope)]]
- [[Go - Defer, Panic, Recover]]