# GORM - ORM для Go

GORM — это полнофункциональная ORM (Object-Relational Mapping) библиотека для Go, предоставляющая удобный способ работы с реляционными базами данных через объектно-ориентированный подход.

## Основные возможности

### Установка

```bash
go get -u gorm.io/gorm
go get -u gorm.io/driver/postgres  # для PostgreSQL
go get -u gorm.io/driver/mysql     # для MySQL
go get -u gorm.io/driver/sqlite    # для SQLite
```

### Подключение к базе данных

```go
import (
    "gorm.io/driver/postgres"
    "gorm.io/gorm"
)

func main() {
    // PostgreSQL
    dsn := "host=localhost user=postgres password=secret dbname=testdb port=5432 sslmode=disable"
    db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{})
    if err != nil {
        panic("failed to connect database")
    }

    // MySQL
    // dsn := "user:password@tcp(127.0.0.1:3306)/dbname?charset=utf8mb4&parseTime=True&loc=Local"
    // db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{})

    // SQLite
    // db, err := gorm.Open(sqlite.Open("test.db"), &gorm.Config{})
}
```

## Определение моделей

### Базовая модель

```go
type User struct {
    ID        uint           `gorm:"primaryKey"`
    Name      string         `gorm:"size:100;not null"`
    Email     string         `gorm:"uniqueIndex;not null"`
    Age       int            `gorm:"default:18"`
    CreatedAt time.Time
    UpdatedAt time.Time
    DeletedAt gorm.DeletedAt `gorm:"index"` // Для soft delete
}
```

### Встроенная модель gorm.Model

```go
// gorm.Model уже содержит ID, CreatedAt, UpdatedAt, DeletedAt
type User struct {
    gorm.Model
    Name  string
    Email string
}

// Эквивалентно:
// type User struct {
//     ID        uint `gorm:"primaryKey"`
//     CreatedAt time.Time
//     UpdatedAt time.Time
//     DeletedAt gorm.DeletedAt `gorm:"index"`
//     Name      string
//     Email     string
// }
```

### Теги структур

```go
type Product struct {
    ID          uint    `gorm:"primaryKey"`
    Code        string  `gorm:"unique;not null;size:50"`
    Price       float64 `gorm:"type:decimal(10,2);default:0"`
    Description string  `gorm:"type:text"`
    Stock       int     `gorm:"check:stock >= 0"`
    IsActive    bool    `gorm:"default:true;index"`
}
```

## CRUD операции

### Create (Создание)

```go
// Создание одной записи
user := User{Name: "Иван", Email: "ivan@example.com", Age: 25}
result := db.Create(&user)

if result.Error != nil {
    // Обработка ошибки
}
// user.ID теперь содержит ID созданной записи

// Создание нескольких записей
users := []User{
    {Name: "Иван", Email: "ivan@example.com"},
    {Name: "Мария", Email: "maria@example.com"},
}
db.Create(&users)

// Создание с выбранными полями
db.Select("Name", "Email").Create(&user)

// Пропуск определенных полей
db.Omit("Age").Create(&user)
```

### Read (Чтение)

```go
var user User

// Получить первую запись
db.First(&user)
// SELECT * FROM users ORDER BY id LIMIT 1;

// Получить запись по primary key
db.First(&user, 10)
// SELECT * FROM users WHERE id = 10;

// Получить последнюю запись
db.Last(&user)
// SELECT * FROM users ORDER BY id DESC LIMIT 1;

// Получить все записи
var users []User
db.Find(&users)
// SELECT * FROM users;

// Условия WHERE
db.Where("name = ?", "Иван").First(&user)
db.Where("age > ?", 20).Find(&users)
db.Where("name IN ?", []string{"Иван", "Мария"}).Find(&users)

// Множественные условия
db.Where("name = ? AND age >= ?", "Иван", 18).Find(&users)

// Struct условия
db.Where(&User{Name: "Иван", Age: 20}).First(&user)

// Map условия
db.Where(map[string]interface{}{"name": "Иван", "age": 20}).Find(&users)
```

### Update (Обновление)

```go
var user User
db.First(&user, 1)

// Обновить одно поле
db.Model(&user).Update("Name", "Петр")

// Обновить несколько полей
db.Model(&user).Updates(User{Name: "Петр", Age: 30})

// Обновить с помощью map
db.Model(&user).Updates(map[string]interface{}{"Name": "Петр", "Age": 30})

// Обновить все записи
db.Model(&User{}).Where("age < ?", 18).Update("age", 18)

// Обновить с выбранными полями
db.Model(&user).Select("Name").Updates(User{Name: "Петр", Age: 30})
// Обновится только Name

// Обновить все поля (включая нулевые)
db.Model(&user).Select("*").Updates(User{Name: "", Age: 0})
```

### Delete (Удаление)

```go
var user User
db.First(&user, 1)

// Soft Delete (если модель имеет DeletedAt)
db.Delete(&user)
// UPDATE users SET deleted_at = NOW() WHERE id = 1;

// Удалить по primary key
db.Delete(&User{}, 10)
db.Delete(&User{}, []int{1, 2, 3})

// Удалить с условием
db.Where("age < ?", 18).Delete(&User{})

// Hard Delete (физическое удаление)
db.Unscoped().Delete(&user)
// DELETE FROM users WHERE id = 1;

// Получить soft deleted записи
db.Unscoped().Where("age > ?", 20).Find(&users)
```

## Продвинутые запросы

### Сортировка

```go
// Сортировка по одному полю
db.Order("age desc").Find(&users)

// Множественная сортировка
db.Order("age desc, name").Find(&users)

// Сортировка по нескольким полям
db.Order("age desc").Order("name").Find(&users)
```

### Лимит и Offset

```go
// LIMIT
db.Limit(10).Find(&users)

// OFFSET
db.Offset(5).Limit(10).Find(&users)

// Пагинация
func Paginate(page, pageSize int) func(db *gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        offset := (page - 1) * pageSize
        return db.Offset(offset).Limit(pageSize)
    }
}

db.Scopes(Paginate(1, 20)).Find(&users)
```

### Group By и Having

```go
type Result struct {
    Age   int
    Count int
}

var results []Result
db.Model(&User{}).Select("age, count(*) as count").
    Group("age").
    Having("count > ?", 1).
    Find(&results)
```

### Joins

```go
type User struct {
    ID   uint
    Name string
}

type Order struct {
    ID     uint
    UserID uint
    Amount float64
}

// Inner Join
db.Model(&User{}).
    Select("users.name, orders.amount").
    Joins("JOIN orders ON orders.user_id = users.id").
    Scan(&results)

// Left Join
db.Model(&User{}).
    Select("users.name, orders.amount").
    Joins("LEFT JOIN orders ON orders.user_id = users.id").
    Scan(&results)
```

### Подзапросы

```go
// Подзапрос в WHERE
db.Where("amount > (?)", db.Table("orders").Select("AVG(amount)")).Find(&orders)

// Подзапрос в FROM
subQuery := db.Select("AVG(age)").Where("name LIKE ?", "name%").Table("users")
db.Select("AVG(age) as avgage").Group("name").Having("AVG(age) > (?)", subQuery).Find(&results)
```

## Связи (Associations)

### Belongs To (один к одному)

```go
type User struct {
    gorm.Model
    Name      string
    CompanyID uint
    Company   Company // Связь belongs to
}

type Company struct {
    gorm.Model
    Name string
}

// Загрузка с связью
var user User
db.Preload("Company").First(&user, 1)
```

### Has One (один к одному)

```go
type User struct {
    gorm.Model
    Name    string
    Profile Profile // Has One
}

type Profile struct {
    gorm.Model
    UserID uint
    Bio    string
}

// Загрузка с связью
db.Preload("Profile").Find(&users)
```

### Has Many (один ко многим)

```go
type User struct {
    gorm.Model
    Name   string
    Orders []Order // Has Many
}

type Order struct {
    gorm.Model
    UserID uint
    Amount float64
}

// Загрузка с связью
db.Preload("Orders").Find(&users)

// Загрузка с условием
db.Preload("Orders", "amount > ?", 100).Find(&users)
```

### Many To Many (многие ко многим)

```go
type User struct {
    gorm.Model
    Name      string
    Languages []Language `gorm:"many2many:user_languages;"`
}

type Language struct {
    gorm.Model
    Name string
}

// Загрузка с связью
db.Preload("Languages").Find(&users)

// Добавление связи
user := User{Name: "Иван"}
language := Language{Name: "Go"}

db.Model(&user).Association("Languages").Append(&language)

// Удаление связи
db.Model(&user).Association("Languages").Delete(&language)
```

## Транзакции

### Ручные транзакции

```go
// Начать транзакцию
tx := db.Begin()

// Выполнить операции
if err := tx.Create(&user).Error; err != nil {
    tx.Rollback()
    return err
}

if err := tx.Create(&order).Error; err != nil {
    tx.Rollback()
    return err
}

// Зафиксировать транзакцию
tx.Commit()
```

### Транзакции с помощью функции

```go
err := db.Transaction(func(tx *gorm.DB) error {
    // Операции внутри транзакции
    if err := tx.Create(&user).Error; err != nil {
        return err // Автоматический rollback
    }

    if err := tx.Create(&order).Error; err != nil {
        return err
    }

    // Вернуть nil для commit
    return nil
})
```

## Hooks (Хуки)

```go
type User struct {
    gorm.Model
    Name     string
    Password string
}

// Before Create
func (u *User) BeforeCreate(tx *gorm.DB) error {
    // Хеширование пароля перед созданием
    hashedPassword, err := bcrypt.GenerateFromPassword([]byte(u.Password), bcrypt.DefaultCost)
    if err != nil {
        return err
    }
    u.Password = string(hashedPassword)
    return nil
}

// After Create
func (u *User) AfterCreate(tx *gorm.DB) error {
    // Логирование после создания
    fmt.Printf("User created: %s\n", u.Name)
    return nil
}

// Before Update
func (u *User) BeforeUpdate(tx *gorm.DB) error {
    // Валидация перед обновлением
    if u.Name == "" {
        return errors.New("name cannot be empty")
    }
    return nil
}

// Before Delete
func (u *User) BeforeDelete(tx *gorm.DB) error {
    // Проверка перед удалением
    return nil
}
```

## Миграции

```go
// AutoMigrate создает/обновляет таблицы
db.AutoMigrate(&User{}, &Order{}, &Product{})

// Создание таблицы если не существует
db.Migrator().CreateTable(&User{})

// Удаление таблицы
db.Migrator().DropTable(&User{})

// Проверка существования таблицы
hasTable := db.Migrator().HasTable(&User{})

// Добавление колонки
db.Migrator().AddColumn(&User{}, "Age")

// Удаление колонки
db.Migrator().DropColumn(&User{}, "Age")

// Создание индекса
db.Migrator().CreateIndex(&User{}, "Email")
```

## Scopes (области видимости)

```go
// Определение scope
func ActiveUsers(db *gorm.DB) *gorm.DB {
    return db.Where("is_active = ?", true)
}

func AgeGreaterThan(age int) func(db *gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        return db.Where("age > ?", age)
    }
}

// Использование scopes
db.Scopes(ActiveUsers).Find(&users)
db.Scopes(ActiveUsers, AgeGreaterThan(18)).Find(&users)
```

## Best Practices

### ✅ Правильно

```go
// Проверка ошибок
if err := db.First(&user, id).Error; err != nil {
    if errors.Is(err, gorm.ErrRecordNotFound) {
        // Запись не найдена
    } else {
        // Другая ошибка
    }
}

// Использование транзакций для связанных операций
db.Transaction(func(tx *gorm.DB) error {
    if err := tx.Create(&user).Error; err != nil {
        return err
    }
    return tx.Create(&order).Error
})

// Preload для избежания N+1 запросов
db.Preload("Orders").Find(&users)

// Использование prepared statements для безопасности
db.Where("name = ?", userInput).Find(&users)
```

### ❌ Неправильно

```go
// Игнорирование ошибок
db.Create(&user) // Без проверки err

// SQL injection
db.Where(fmt.Sprintf("name = '%s'", userInput)).Find(&users) // Опасно!

// N+1 проблема
for _, user := range users {
    db.Model(&user).Association("Orders").Find(&orders)
}

// Использование Select("*") без необходимости
db.Select("*").Find(&users) // Загружает все поля
```

## Вопросы с собеседований

**Вопрос:** Что такое N+1 проблема и как её решить в GORM?

**Ответ:** N+1 проблема возникает когда для каждой записи выполняется отдельный запрос для загрузки связанных данных. Решение — использовать `Preload()` для eager loading: `db.Preload("Orders").Find(&users)`.

**Вопрос:** В чем разница между мягким и жестким удалением?

**Ответ:**
- Soft Delete (мягкое) — запись помечается удаленной (поле `deleted_at`), но физически остается в БД. Используется `db.Delete()`.
- Hard Delete (жесткое) — запись физически удаляется из БД. Используется `db.Unscoped().Delete()`.

**Вопрос:** Как работают хуки в GORM?

**Ответ:** Хуки — это методы, автоматически вызываемые до или после определенных операций (Create, Update, Delete). Они определяются как методы модели: `BeforeCreate`, `AfterCreate`, `BeforeUpdate`, и т.д.

## Связанные темы

- [[PostgreSQL - Основы]]
- [[Go - Структуры (struct)]]
- [[sqlx]]
- [[PostgreSQL - Типы индексов]]
- [[SQL - JOIN операции]]