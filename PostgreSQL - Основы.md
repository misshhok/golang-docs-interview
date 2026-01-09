# PostgreSQL - Основы

PostgreSQL - мощная open-source реляционная СУБД с поддержкой ACID, транзакций, сложных запросов и расширяемостью.

## Ключевые характеристики

- **ACID** гарантии
- **SQL** стандарт
- **Расширяемость** (custom types, functions, operators)
- **Concurrent access** с MVCC
- **Rich data types** (JSON, arrays, hstore, geometric)
- **Full-text search**
- **Репликация** и high availability

## Подключение из Go

### database/sql + pgx

```go
import (
    "database/sql"
    _ "github.com/jackc/pgx/v5/stdlib"
)

func main() {
    dsn := "postgres://user:password@localhost:5432/dbname?sslmode=disable"
    db, err := sql.Open("pgx", dsn)
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    // Проверка соединения
    if err := db.Ping(); err != nil {
        log.Fatal(err)
    }
}
```

### Connection Pool

```go
db.SetMaxOpenConns(25)               // Максимум 25 соединений
db.SetMaxIdleConns(25)               // Держать 25 idle соединений
db.SetConnMaxLifetime(5 * time.Minute) // Переоткрывать через 5 минут
```

## Базовые операции

### INSERT

```go
// Вставка одной строки
result, err := db.Exec(`
    INSERT INTO users (name, email, age)
    VALUES ($1, $2, $3)
`, "John Doe", "john@example.com", 30)

// Получить ID
id, err := result.LastInsertId() // Не поддерживается в PostgreSQL!

// ✅ В PostgreSQL используйте RETURNING
var id int
err = db.QueryRow(`
    INSERT INTO users (name, email, age)
    VALUES ($1, $2, $3)
    RETURNING id
`, "John Doe", "john@example.com", 30).Scan(&id)
```

### SELECT

```go
// Одна строка
var user User
err := db.QueryRow(`
    SELECT id, name, email, age
    FROM users
    WHERE id = $1
`, userID).Scan(&user.ID, &user.Name, &user.Email, &user.Age)

if err == sql.ErrNoRows {
    // Пользователь не найден
}

// Множество строк
rows, err := db.Query(`
    SELECT id, name, email
    FROM users
    WHERE age > $1
`, 18)
if err != nil {
    return err
}
defer rows.Close()

var users []User
for rows.Next() {
    var u User
    if err := rows.Scan(&u.ID, &u.Name, &u.Email); err != nil {
        return err
    }
    users = append(users, u)
}

// Проверка ошибок после цикла
if err := rows.Err(); err != nil {
    return err
}
```

### UPDATE

```go
result, err := db.Exec(`
    UPDATE users
    SET email = $1, updated_at = NOW()
    WHERE id = $2
`, newEmail, userID)

rowsAffected, err := result.RowsAffected()
if rowsAffected == 0 {
    // Строка не найдена
}
```

### DELETE

```go
result, err := db.Exec(`
    DELETE FROM users
    WHERE id = $1
`, userID)

rowsAffected, err := result.RowsAffected()
```

## Транзакции

```go
func transferMoney(db *sql.DB, fromID, toID int, amount float64) error {
    // Начать транзакцию
    tx, err := db.Begin()
    if err != nil {
        return err
    }

    // Rollback при panic или ошибке
    defer func() {
        if err != nil {
            tx.Rollback()
        }
    }()

    // Снять деньги
    _, err = tx.Exec(`
        UPDATE accounts
        SET balance = balance - $1
        WHERE id = $2 AND balance >= $1
    `, amount, fromID)
    if err != nil {
        return err
    }

    // Добавить деньги
    _, err = tx.Exec(`
        UPDATE accounts
        SET balance = balance + $1
        WHERE id = $2
    `, amount, toID)
    if err != nil {
        return err
    }

    // Коммит
    return tx.Commit()
}
```

### Транзакции с context

```go
func updateUserTx(ctx context.Context, db *sql.DB, user User) error {
    tx, err := db.BeginTx(ctx, nil)
    if err != nil {
        return err
    }
    defer tx.Rollback()

    // Используйте tx вместо db
    _, err = tx.ExecContext(ctx, `
        UPDATE users SET name = $1 WHERE id = $2
    `, user.Name, user.ID)
    if err != nil {
        return err
    }

    return tx.Commit()
}
```

Подробнее: [[Go - Context]]

## Prepared Statements

```go
// Подготовить statement
stmt, err := db.Prepare(`
    SELECT id, name, email
    FROM users
    WHERE age > $1
`)
if err != nil {
    return err
}
defer stmt.Close()

// Использовать многократно
rows, err := stmt.Query(18)
// ...

rows, err = stmt.Query(21)
// ...
```

**Когда использовать:**
- ✅ Один запрос выполняется много раз
- ❌ Запрос выполняется один раз (лишний overhead)

## NULL значения

```go
type User struct {
    ID    int
    Name  string
    Email sql.NullString // Может быть NULL
    Age   sql.NullInt64  // Может быть NULL
}

// Чтение
var user User
err := db.QueryRow("SELECT id, name, email, age FROM users WHERE id = $1", id).
    Scan(&user.ID, &user.Name, &user.Email, &user.Age)

// Проверка NULL
if user.Email.Valid {
    fmt.Println("Email:", user.Email.String)
} else {
    fmt.Println("Email is NULL")
}

// Запись NULL
db.Exec("INSERT INTO users (name, email) VALUES ($1, $2)",
    "John", sql.NullString{String: "", Valid: false}) // NULL
```

## Индексы

Индексы ускоряют поиск данных.

```sql
-- Создание индекса
CREATE INDEX idx_users_email ON users(email);

-- Уникальный индекс
CREATE UNIQUE INDEX idx_users_email_unique ON users(email);

-- Композитный индекс
CREATE INDEX idx_users_name_age ON users(name, age);

-- Partial индекс
CREATE INDEX idx_active_users ON users(email) WHERE active = true;

-- B-tree (по умолчанию)
CREATE INDEX idx_users_created ON users(created_at);

-- Hash индекс (только для =)
CREATE INDEX idx_users_id_hash ON users USING HASH (id);

-- GIN для JSON
CREATE INDEX idx_users_metadata ON users USING GIN (metadata);
```

Подробнее: [[PostgreSQL - Типы индексов]]

## EXPLAIN

Анализ плана выполнения запроса:

```sql
EXPLAIN ANALYZE
SELECT * FROM users WHERE email = 'john@example.com';

-- Output:
-- Seq Scan on users  (cost=0.00..25.00 rows=1 width=40)
--   Filter: (email = 'john@example.com'::text)
--   Planning Time: 0.123 ms
--   Execution Time: 0.456 ms

-- После создания индекса:
-- Index Scan using idx_users_email on users (cost=0.28..8.29 rows=1 width=40)
--   Index Cond: (email = 'john@example.com'::text)
```

**Типы сканирования:**
- `Seq Scan` - последовательное сканирование (медленно)
- `Index Scan` - использует индекс (быстро)
- `Index Only Scan` - только индекс, без таблицы (очень быстро)
- `Bitmap Index Scan` - для больших результатов

Подробнее: [[PostgreSQL - Оптимизация запросов]]

## JOIN'ы

```sql
-- INNER JOIN
SELECT u.name, o.total
FROM users u
INNER JOIN orders o ON u.id = o.user_id;

-- LEFT JOIN
SELECT u.name, COALESCE(SUM(o.total), 0) as total_spent
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.name;

-- RIGHT JOIN
SELECT u.name, o.order_number
FROM users u
RIGHT JOIN orders o ON u.id = o.user_id;

-- FULL OUTER JOIN
SELECT u.name, o.order_number
FROM users u
FULL OUTER JOIN orders o ON u.id = o.user_id;
```

Подробнее: [[SQL - JOIN операции]]

## Агрегация и группировка

```sql
-- COUNT, SUM, AVG, MIN, MAX
SELECT
    category,
    COUNT(*) as total_products,
    AVG(price) as avg_price,
    MAX(price) as max_price
FROM products
GROUP BY category
HAVING AVG(price) > 100;
```

Подробнее: [[SQL - Агрегация и группировка]]

## Типичные паттерны в Go

### Repository Pattern

```go
type UserRepository struct {
    db *sql.DB
}

func NewUserRepository(db *sql.DB) *UserRepository {
    return &UserRepository{db: db}
}

func (r *UserRepository) GetByID(ctx context.Context, id int) (*User, error) {
    var user User
    err := r.db.QueryRowContext(ctx, `
        SELECT id, name, email, age
        FROM users
        WHERE id = $1
    `, id).Scan(&user.ID, &user.Name, &user.Email, &user.Age)

    if err == sql.ErrNoRows {
        return nil, ErrNotFound
    }
    if err != nil {
        return nil, fmt.Errorf("get user: %w", err)
    }

    return &user, nil
}

func (r *UserRepository) Create(ctx context.Context, user *User) error {
    return r.db.QueryRowContext(ctx, `
        INSERT INTO users (name, email, age)
        VALUES ($1, $2, $3)
        RETURNING id
    `, user.Name, user.Email, user.Age).Scan(&user.ID)
}
```

Подробнее: [[Repository паттерн]]

### Pagination

```go
func (r *UserRepository) List(ctx context.Context, offset, limit int) ([]User, error) {
    rows, err := r.db.QueryContext(ctx, `
        SELECT id, name, email
        FROM users
        ORDER BY id
        LIMIT $1 OFFSET $2
    `, limit, offset)
    if err != nil {
        return nil, err
    }
    defer rows.Close()

    var users []User
    for rows.Next() {
        var u User
        if err := rows.Scan(&u.ID, &u.Name, &u.Email); err != nil {
            return nil, err
        }
        users = append(users, u)
    }

    return users, rows.Err()
}

// Использование
users, err := repo.List(ctx, 0, 10) // Первые 10
users, err = repo.List(ctx, 10, 10) // Следующие 10
```

### Bulk Insert

```go
func (r *UserRepository) BulkCreate(ctx context.Context, users []User) error {
    tx, err := r.db.BeginTx(ctx, nil)
    if err != nil {
        return err
    }
    defer tx.Rollback()

    stmt, err := tx.PrepareContext(ctx, `
        INSERT INTO users (name, email, age)
        VALUES ($1, $2, $3)
    `)
    if err != nil {
        return err
    }
    defer stmt.Close()

    for _, user := range users {
        if _, err := stmt.ExecContext(ctx, user.Name, user.Email, user.Age); err != nil {
            return err
        }
    }

    return tx.Commit()
}
```

## Частые ошибки

### 1. SQL Injection

```go
// ❌ SQL Injection уязвимость!
query := fmt.Sprintf("SELECT * FROM users WHERE name = '%s'", userName)
db.Query(query)

// ✅ Используйте параметризованные запросы
db.Query("SELECT * FROM users WHERE name = $1", userName)
```

### 2. Не закрывать rows

```go
// ❌ Утечка ресурсов
rows, err := db.Query("SELECT * FROM users")
for rows.Next() {
    // ...
}
// Забыли rows.Close()!

// ✅ Всегда defer Close()
rows, err := db.Query("SELECT * FROM users")
if err != nil {
    return err
}
defer rows.Close()
```

### 3. Игнорировать rows.Err()

```go
// ❌ Пропускаем ошибки
for rows.Next() {
    rows.Scan(&user)
}

// ✅ Проверяем ошибки после цикла
for rows.Next() {
    if err := rows.Scan(&user); err != nil {
        return err
    }
}
if err := rows.Err(); err != nil {
    return err
}
```

### 4. Неправильный Rollback

```go
// ❌ Rollback только в одном месте
tx, _ := db.Begin()
if err := doSomething(tx); err != nil {
    tx.Rollback()
    return err
}
tx.Commit()

// ✅ defer Rollback (безопасно вызывать после Commit)
tx, _ := db.Begin()
defer tx.Rollback()

if err := doSomething(tx); err != nil {
    return err
}

return tx.Commit()
```

## Best Practices

1. ✅ Используйте параметризованные запросы ($1, $2)
2. ✅ Всегда defer Close() для rows и statements
3. ✅ Используйте контекст для timeout'ов
4. ✅ Настройте connection pool
5. ✅ Используйте транзакции для связанных операций
6. ✅ Создавайте индексы для часто запрашиваемых полей
7. ✅ Используйте EXPLAIN для анализа запросов

## Связанные темы

- [[PostgreSQL - Типы индексов]]
- [[PostgreSQL - ACID]]
- [[PostgreSQL - Блокировки и изоляция транзакций]]
- [[SQL - JOIN операции]]
- [[SQL - Агрегация и группировка]]
- [[PostgreSQL - Оптимизация запросов]]
- [[Repository паттерн]]
- [[Go - Context]]
