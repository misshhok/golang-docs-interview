# sqlx

sqlx — это библиотека, которая расширяет стандартный пакет `database/sql` в Go, добавляя удобные функции для работы с базами данных. В отличие от полноценных ORM, sqlx сохраняет контроль над SQL запросами, но упрощает работу с результатами.

## Основные возможности

### Установка

```bash
go get github.com/jmoiron/sqlx
```

### Поддерживаемые драйверы

```go
import (
    "github.com/jmoiron/sqlx"
    _ "github.com/lib/pq"                // PostgreSQL
    _ "github.com/go-sql-driver/mysql"   // MySQL
    _ "github.com/mattn/go-sqlite3"      // SQLite
)
```

### Подключение к базе данных

```go
// PostgreSQL
db, err := sqlx.Connect("postgres", "user=foo dbname=bar sslmode=disable")
if err != nil {
    log.Fatal(err)
}
defer db.Close()

// MySQL
// db, err := sqlx.Connect("mysql", "user:password@tcp(127.0.0.1:3306)/dbname")

// SQLite
// db, err := sqlx.Connect("sqlite3", "mydb.db")

// Или с Open (без проверки подключения)
db, err := sqlx.Open("postgres", dsn)
if err != nil {
    log.Fatal(err)
}

// Проверка подключения
if err := db.Ping(); err != nil {
    log.Fatal(err)
}
```

## Основные типы

### DB, Tx, Stmt

```go
// DB - connection pool
type DB struct {
    *sql.DB
}

// Tx - транзакция
type Tx struct {
    *sql.Tx
}

// Stmt - prepared statement
type Stmt struct {
    *sql.Stmt
}
```

## Работа со структурами

### Определение структур

```go
type User struct {
    ID        int       `db:"id"`
    Name      string    `db:"name"`
    Email     string    `db:"email"`
    Age       int       `db:"age"`
    CreatedAt time.Time `db:"created_at"`
}

// Для полей, которые могут быть NULL
type UserNullable struct {
    ID    int            `db:"id"`
    Name  string         `db:"name"`
    Email sql.NullString `db:"email"` // Может быть NULL
    Age   sql.NullInt64  `db:"age"`   // Может быть NULL
}
```

## Query методы

### Get - получение одной записи

```go
var user User

// Get сканирует одну строку в структуру
err := db.Get(&user, "SELECT * FROM users WHERE id = $1", 1)
if err != nil {
    if err == sql.ErrNoRows {
        // Запись не найдена
    }
    log.Fatal(err)
}

fmt.Printf("User: %+v\n", user)
```

### Select - получение нескольких записей

```go
var users []User

// Select сканирует несколько строк в slice
err := db.Select(&users, "SELECT * FROM users WHERE age > $1", 18)
if err != nil {
    log.Fatal(err)
}

for _, user := range users {
    fmt.Printf("User: %+v\n", user)
}
```

### Query и QueryRow

```go
// QueryRow - для одной строки
var name string
var email string
err := db.QueryRow("SELECT name, email FROM users WHERE id = $1", 1).Scan(&name, &email)

// Queryx - возвращает *sqlx.Rows с дополнительными методами
rows, err := db.Queryx("SELECT * FROM users")
if err != nil {
    log.Fatal(err)
}
defer rows.Close()

for rows.Next() {
    var user User
    err := rows.StructScan(&user)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("%+v\n", user)
}
```

## Named Queries (именованные запросы)

### Named параметры

```go
// Вместо позиционных параметров $1, $2
// используем именованные :name, :age

// С map
m := map[string]interface{}{
    "name": "Иван",
    "age":  25,
}

rows, err := db.NamedQuery("SELECT * FROM users WHERE name = :name AND age > :age", m)

// Со структурой
type SearchParams struct {
    Name string `db:"name"`
    Age  int    `db:"age"`
}

params := SearchParams{Name: "Иван", Age: 25}
rows, err := db.NamedQuery("SELECT * FROM users WHERE name = :name AND age > :age", params)
```

### NamedExec

```go
// INSERT с именованными параметрами
user := User{
    Name:  "Иван",
    Email: "ivan@example.com",
    Age:   25,
}

result, err := db.NamedExec(`
    INSERT INTO users (name, email, age)
    VALUES (:name, :email, :age)
`, user)

if err != nil {
    log.Fatal(err)
}

id, _ := result.LastInsertId()
rows, _ := result.RowsAffected()
```

## IN запросы

### Обработка IN условий

```go
// Проблема: SELECT * FROM users WHERE id IN (?)
// Нужно: SELECT * FROM users WHERE id IN (?, ?, ?)

ids := []int{1, 2, 3, 4, 5}

// Использование In для расширения параметров
query, args, err := sqlx.In("SELECT * FROM users WHERE id IN (?)", ids)
if err != nil {
    log.Fatal(err)
}

// Rebind для корректной замены placeholders (?, $1, и т.д.)
query = db.Rebind(query)

var users []User
err = db.Select(&users, query, args...)
```

### Named IN запросы

```go
type Params struct {
    IDs []int `db:"ids"`
}

params := Params{IDs: []int{1, 2, 3}}

query, args, err := sqlx.Named("SELECT * FROM users WHERE id IN (:ids)", params)
query, args, err = sqlx.In(query, args...)
query = db.Rebind(query)

var users []User
err = db.Select(&users, query, args...)
```

## Транзакции

### Основные операции

```go
// Начать транзакцию
tx, err := db.Beginx()
if err != nil {
    log.Fatal(err)
}

// Операции в транзакции
_, err = tx.Exec("INSERT INTO users (name, email) VALUES ($1, $2)", "Иван", "ivan@example.com")
if err != nil {
    tx.Rollback()
    log.Fatal(err)
}

_, err = tx.Exec("UPDATE accounts SET balance = balance - 100 WHERE user_id = $1", 1)
if err != nil {
    tx.Rollback()
    log.Fatal(err)
}

// Commit транзакции
err = tx.Commit()
if err != nil {
    log.Fatal(err)
}
```

### Named операции в транзакциях

```go
tx, err := db.Beginx()
if err != nil {
    log.Fatal(err)
}
defer func() {
    if err != nil {
        tx.Rollback()
        return
    }
    err = tx.Commit()
}()

user := User{Name: "Иван", Email: "ivan@example.com", Age: 25}

_, err = tx.NamedExec(`
    INSERT INTO users (name, email, age)
    VALUES (:name, :email, :age)
`, user)
```

## Prepared Statements

```go
// Prepared statement для многократного использования
stmt, err := db.Preparex("SELECT * FROM users WHERE age > $1")
if err != nil {
    log.Fatal(err)
}
defer stmt.Close()

// Использование prepared statement
var users1 []User
err = stmt.Select(&users1, 18)

var users2 []User
err = stmt.Select(&users2, 25)
```

## Сканирование результатов

### StructScan

```go
rows, err := db.Queryx("SELECT * FROM users")
defer rows.Close()

for rows.Next() {
    var user User
    err := rows.StructScan(&user)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("%+v\n", user)
}
```

### MapScan

```go
rows, err := db.Queryx("SELECT * FROM users")
defer rows.Close()

for rows.Next() {
    result := make(map[string]interface{})
    err := rows.MapScan(result)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("%v\n", result)
}
```

### SliceScan

```go
rows, err := db.Queryx("SELECT name, email FROM users")
defer rows.Close()

for rows.Next() {
    var result []interface{}
    err := rows.SliceScan(&result)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Name: %s, Email: %s\n", result[0], result[1])
}
```

## Полный пример CRUD

```go
package main

import (
    "fmt"
    "log"
    "github.com/jmoiron/sqlx"
    _ "github.com/lib/pq"
)

type User struct {
    ID    int    `db:"id"`
    Name  string `db:"name"`
    Email string `db:"email"`
    Age   int    `db:"age"`
}

func main() {
    // Подключение
    db, err := sqlx.Connect("postgres", "user=postgres dbname=testdb sslmode=disable")
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    // CREATE
    user := User{
        Name:  "Иван",
        Email: "ivan@example.com",
        Age:   25,
    }

    result, err := db.NamedExec(`
        INSERT INTO users (name, email, age)
        VALUES (:name, :email, :age)
    `, user)
    if err != nil {
        log.Fatal(err)
    }

    id, _ := result.LastInsertId()
    fmt.Printf("Created user with ID: %d\n", id)

    // READ - одна запись
    var fetchedUser User
    err = db.Get(&fetchedUser, "SELECT * FROM users WHERE id = $1", id)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Fetched user: %+v\n", fetchedUser)

    // READ - несколько записей
    var users []User
    err = db.Select(&users, "SELECT * FROM users WHERE age > $1", 18)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Found %d users\n", len(users))

    // UPDATE
    _, err = db.Exec("UPDATE users SET age = $1 WHERE id = $2", 26, id)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println("User updated")

    // DELETE
    _, err = db.Exec("DELETE FROM users WHERE id = $1", id)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println("User deleted")
}
```

## Когда использовать sqlx вместо GORM

### Используйте sqlx когда:

✅ **Преимущества sqlx:**
- Нужен полный контроль над SQL запросами
- Работа со сложными запросами и оптимизациями
- Важна производительность (меньше overhead)
- Простая схема БД без сложных связей
- Нужна прозрачность (видно какие запросы выполняются)

### Используйте GORM когда:

✅ **Преимущества GORM:**
- Много связей между таблицами (has many, belongs to)
- Нужны автоматические миграции
- Hooks для бизнес-логики
- Прототипирование или быстрая разработка
- Команда больше знакома с ORM подходом

## Best Practices

### ✅ Правильно

```go
// Использовать Rebind для портируемости
query := db.Rebind("SELECT * FROM users WHERE id = ?")

// Проверка на sql.ErrNoRows
err := db.Get(&user, "SELECT * FROM users WHERE id = $1", id)
if err == sql.ErrNoRows {
    // Запись не найдена
}

// Использование транзакций для связанных операций
tx, _ := db.Beginx()
defer func() {
    if err != nil {
        tx.Rollback()
    }
}()

// Использование prepared statements для повторяющихся запросов
stmt, _ := db.Preparex("SELECT * FROM users WHERE age > $1")
defer stmt.Close()
```

### ❌ Неправильно

```go
// SQL injection
query := fmt.Sprintf("SELECT * FROM users WHERE name = '%s'", userInput) // Опасно!

// Игнорирование ошибок
db.Exec("UPDATE users SET name = $1", "John")

// Не закрывать rows
rows, _ := db.Queryx("SELECT * FROM users")
// Забыли rows.Close()

// Использование Get для нескольких записей
var users []User
db.Get(&users, "SELECT * FROM users") // Неправильно! Нужен Select
```

## Вопросы с собеседований

**Вопрос:** В чем отличие sqlx от стандартного database/sql?

**Ответ:** sqlx расширяет `database/sql`, добавляя:
- Автоматическое сканирование в структуры (StructScan)
- Именованные запросы (Named queries)
- Поддержка IN запросов (sqlx.In)
- Get/Select для упрощенной работы
При этом сохраняется полная совместимость с `database/sql`.

**Вопрос:** Когда использовать Get, а когда Select?

**Ответ:**
- `Get` — для получения одной записи, сканирует в структуру
- `Select` — для получения нескольких записей, сканирует в slice структур

**Вопрос:** Как работать с IN запросами в sqlx?

**Ответ:** Использовать `sqlx.In()` для расширения параметров и `Rebind()` для корректной замены placeholders:
```go
query, args, _ := sqlx.In("SELECT * FROM users WHERE id IN (?)", []int{1,2,3})
query = db.Rebind(query)
db.Select(&users, query, args...)
```

## Связанные темы

- [[GORM - ORM для Go]]
- [[PostgreSQL - Основы]]
- [[Go - Структуры (struct)]]
- [[SQL - JOIN операции]]
- [[PostgreSQL - Оптимизация запросов]]