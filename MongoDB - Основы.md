# MongoDB - Основы

MongoDB - популярная документо-ориентированная NoSQL база данных с гибкой схемой и горизонтальным масштабированием.

## Что такое MongoDB?

**Document-oriented:** Данные хранятся как JSON-подобные документы (BSON).

```json
{
  "_id": ObjectId("507f1f77bcf86cd799439011"),
  "name": "Alice",
  "email": "alice@example.com",
  "age": 30,
  "address": {
    "city": "Moscow",
    "street": "Tverskaya 1"
  },
  "hobbies": ["reading", "coding"],
  "created_at": ISODate("2024-01-09T10:00:00Z")
}
```

**Основные концепции:**
- **Database** - база данных
- **Collection** - коллекция (аналог таблицы)
- **Document** - документ (аналог строки)
- **Field** - поле (аналог колонки)

## Установка и подключение

### Установка

```bash
# Docker
docker run -d -p 27017:27017 --name mongodb mongo:latest

# macOS
brew install mongodb-community
brew services start mongodb-community
```

### Подключение в Go

```go
import (
    "context"
    "go.mongodb.org/mongo-driver/mongo"
    "go.mongodb.org/mongo-driver/mongo/options"
)

func main() {
    ctx := context.Background()

    // Подключение
    client, err := mongo.Connect(ctx, options.Client().
        ApplyURI("mongodb://localhost:27017"))
    if err != nil {
        panic(err)
    }
    defer client.Disconnect(ctx)

    // Проверка подключения
    err = client.Ping(ctx, nil)
    if err != nil {
        panic(err)
    }

    // Выбрать БД и коллекцию
    db := client.Database("myapp")
    users := db.Collection("users")
}
```

## CRUD операции

### Create (вставка)

```go
import "go.mongodb.org/mongo-driver/bson"

// InsertOne - вставить один документ
user := bson.M{
    "name":  "Alice",
    "email": "alice@example.com",
    "age":   30,
}

result, err := users.InsertOne(ctx, user)
if err != nil {
    log.Fatal(err)
}

insertedID := result.InsertedID
fmt.Println("Inserted ID:", insertedID)

// InsertMany - вставить несколько
newUsers := []interface{}{
    bson.M{"name": "Bob", "age": 25},
    bson.M{"name": "Charlie", "age": 35},
}

results, err := users.InsertMany(ctx, newUsers)
fmt.Println("Inserted IDs:", results.InsertedIDs)
```

### Read (чтение)

```go
// FindOne - найти один документ
var result bson.M
err := users.FindOne(ctx, bson.M{"name": "Alice"}).Decode(&result)
if err == mongo.ErrNoDocuments {
    fmt.Println("Not found")
} else if err != nil {
    log.Fatal(err)
} else {
    fmt.Println(result)
}

// Find - найти несколько
cursor, err := users.Find(ctx, bson.M{"age": bson.M{"$gt": 25}})
if err != nil {
    log.Fatal(err)
}
defer cursor.Close(ctx)

// Итерация по результатам
for cursor.Next(ctx) {
    var user bson.M
    if err := cursor.Decode(&user); err != nil {
        log.Fatal(err)
    }
    fmt.Println(user)
}

// Структура вместо bson.M
type User struct {
    ID    primitive.ObjectID `bson:"_id,omitempty"`
    Name  string             `bson:"name"`
    Email string             `bson:"email"`
    Age   int                `bson:"age"`
}

var users []User
cursor, _ := collection.Find(ctx, bson.M{})
cursor.All(ctx, &users)
```

### Update (обновление)

```go
// UpdateOne - обновить один документ
filter := bson.M{"name": "Alice"}
update := bson.M{
    "$set": bson.M{
        "age":   31,
        "email": "alice.new@example.com",
    },
}

result, err := users.UpdateOne(ctx, filter, update)
fmt.Println("Matched:", result.MatchedCount)
fmt.Println("Modified:", result.ModifiedCount)

// UpdateMany - обновить несколько
filter = bson.M{"age": bson.M{"$lt": 30}}
update = bson.M{"$inc": bson.M{"age": 1}}  // Increment age

result, _ = users.UpdateMany(ctx, filter, update)

// ReplaceOne - заменить весь документ
replacement := bson.M{
    "name":  "Alice",
    "email": "alice@example.com",
    "age":   32,
}
users.ReplaceOne(ctx, bson.M{"name": "Alice"}, replacement)

// FindOneAndUpdate - найти и обновить атомарно
opts := options.FindOneAndUpdate().SetReturnDocument(options.After)
var updated bson.M
err = users.FindOneAndUpdate(ctx, filter, update, opts).Decode(&updated)
```

### Delete (удаление)

```go
// DeleteOne - удалить один
result, err := users.DeleteOne(ctx, bson.M{"name": "Alice"})
fmt.Println("Deleted:", result.DeletedCount)

// DeleteMany - удалить несколько
result, _ = users.DeleteMany(ctx, bson.M{"age": bson.M{"$gt": 50}})
fmt.Println("Deleted:", result.DeletedCount)

// FindOneAndDelete - найти и удалить атомарно
var deleted bson.M
err = users.FindOneAndDelete(ctx, bson.M{"name": "Bob"}).Decode(&deleted)
```

## Запросы

### Операторы сравнения

```go
// $eq - равно
users.Find(ctx, bson.M{"age": bson.M{"$eq": 30}})
// Или просто
users.Find(ctx, bson.M{"age": 30})

// $ne - не равно
users.Find(ctx, bson.M{"age": bson.M{"$ne": 30}})

// $gt, $gte - больше, больше или равно
users.Find(ctx, bson.M{"age": bson.M{"$gt": 25}})
users.Find(ctx, bson.M{"age": bson.M{"$gte": 25}})

// $lt, $lte - меньше, меньше или равно
users.Find(ctx, bson.M{"age": bson.M{"$lt": 40}})

// $in - в списке
users.Find(ctx, bson.M{"name": bson.M{"$in": []string{"Alice", "Bob"}}})

// $nin - не в списке
users.Find(ctx, bson.M{"name": bson.M{"$nin": []string{"Charlie"}}})
```

### Логические операторы

```go
// $and - И
users.Find(ctx, bson.M{
    "$and": []bson.M{
        {"age": bson.M{"$gte": 25}},
        {"age": bson.M{"$lte": 35}},
    },
})

// Или просто (implicit AND)
users.Find(ctx, bson.M{
    "age":  bson.M{"$gte": 25},
    "name": "Alice",
})

// $or - ИЛИ
users.Find(ctx, bson.M{
    "$or": []bson.M{
        {"age": bson.M{"$lt": 25}},
        {"age": bson.M{"$gt": 40}},
    },
})

// $not - НЕ
users.Find(ctx, bson.M{
    "age": bson.M{"$not": bson.M{"$gt": 30}},
})

// $nor - НЕ ИЛИ
users.Find(ctx, bson.M{
    "$nor": []bson.M{
        {"age": bson.M{"$lt": 25}},
        {"name": "Charlie"},
    },
})
```

### Вложенные поля

```go
// Точечная нотация
users.Find(ctx, bson.M{"address.city": "Moscow"})

// Вложенный документ целиком
users.Find(ctx, bson.M{
    "address": bson.M{
        "city":   "Moscow",
        "street": "Tverskaya 1",
    },
})
```

### Массивы

```go
// Элемент в массиве
users.Find(ctx, bson.M{"hobbies": "coding"})

// $all - все элементы присутствуют
users.Find(ctx, bson.M{
    "hobbies": bson.M{"$all": []string{"reading", "coding"}},
})

// $size - размер массива
users.Find(ctx, bson.M{"hobbies": bson.M{"$size": 2}})

// $elemMatch - элемент массива соответствует условиям
orders.Find(ctx, bson.M{
    "items": bson.M{
        "$elemMatch": bson.M{
            "price": bson.M{"$gt": 100},
            "qty":   bson.M{"$gte": 2},
        },
    },
})
```

### Регулярные выражения

```go
// Поиск по паттерну
users.Find(ctx, bson.M{
    "email": bson.M{"$regex": ".*@gmail.com$", "$options": "i"},
})

// Или primitive.Regex
import "go.mongodb.org/mongo-driver/bson/primitive"

users.Find(ctx, bson.M{
    "name": primitive.Regex{Pattern: "^A", Options: "i"},
})
```

### Проекция (выбор полей)

```go
// Только определенные поля
opts := options.Find().SetProjection(bson.M{
    "name":  1,
    "email": 1,
    "_id":   0,  // Исключить _id
})

cursor, _ := users.Find(ctx, bson.M{}, opts)
```

## Сортировка, пагинация, лимит

```go
// Сортировка
opts := options.Find().SetSort(bson.M{"age": -1})  // По убыванию
cursor, _ := users.Find(ctx, bson.M{}, opts)

// Лимит
opts = options.Find().SetLimit(10)

// Пропуск (skip)
opts = options.Find().SetSkip(20)

// Комбинация (пагинация)
page := 2
pageSize := 10

opts = options.Find().
    SetSort(bson.M{"created_at": -1}).
    SetSkip(int64((page - 1) * pageSize)).
    SetLimit(int64(pageSize))

cursor, _ = users.Find(ctx, bson.M{}, opts)
```

## Агрегация

Мощный инструмент для сложных запросов.

```go
// Pipeline
pipeline := []bson.M{
    // $match - фильтр (как WHERE)
    {"$match": bson.M{"age": bson.M{"$gte": 25}}},

    // $group - группировка
    {"$group": bson.M{
        "_id":   "$city",
        "count": bson.M{"$sum": 1},
        "avgAge": bson.M{"$avg": "$age"},
    }},

    // $sort - сортировка
    {"$sort": bson.M{"count": -1}},

    // $limit - лимит
    {"$limit": 10},
}

cursor, err := users.Aggregate(ctx, pipeline)
defer cursor.Close(ctx)

var results []bson.M
cursor.All(ctx, &results)

for _, result := range results {
    fmt.Println(result)
}
```

### Агрегационные операторы

```go
// $sum - сумма
{"$group": bson.M{
    "_id": "$category",
    "total": bson.M{"$sum": "$amount"},
}}

// $avg - среднее
{"$group": bson.M{
    "_id": "$category",
    "avgPrice": bson.M{"$avg": "$price"},
}}

// $min, $max - минимум, максимум
{"$group": bson.M{
    "_id": "$category",
    "minPrice": bson.M{"$min": "$price"},
    "maxPrice": bson.M{"$max": "$price"},
}}

// $push - собрать в массив
{"$group": bson.M{
    "_id": "$userId",
    "orders": bson.M{"$push": "$orderId"},
}}

// $addToSet - уникальные значения в массив
{"$group": bson.M{
    "_id": "$userId",
    "categories": bson.M{"$addToSet": "$category"},
}}

// $lookup - JOIN (с версии 3.2)
{"$lookup": bson.M{
    "from":         "orders",
    "localField":   "_id",
    "foreignField": "userId",
    "as":           "userOrders",
}}

// $unwind - развернуть массив
{"$unwind": "$hobbies"}

// $project - выбрать/преобразовать поля
{"$project": bson.M{
    "name": 1,
    "fullName": bson.M{"$concat": []string{"$firstName", " ", "$lastName"}},
    "year": bson.M{"$year": "$createdAt"},
}}
```

## Индексы

```go
import "go.mongodb.org/mongo-driver/mongo/options"

// Создать индекс
indexModel := mongo.IndexModel{
    Keys: bson.M{"email": 1},  // 1 = возрастание, -1 = убывание
    Options: options.Index().SetUnique(true),
}

name, err := users.Indexes().CreateOne(ctx, indexModel)
fmt.Println("Index created:", name)

// Составной индекс
indexModel = mongo.IndexModel{
    Keys: bson.M{"city": 1, "age": -1},
}
users.Indexes().CreateOne(ctx, indexModel)

// Text индекс (полнотекстовый поиск)
indexModel = mongo.IndexModel{
    Keys: bson.M{"$text": "text"},
}
users.Indexes().CreateOne(ctx, indexModel)

// Поиск по тексту
users.Find(ctx, bson.M{"$text": bson.M{"$search": "mongodb"}})

// Список индексов
cursor, _ := users.Indexes().List(ctx)
var indexes []bson.M
cursor.All(ctx, &indexes)

// Удалить индекс
users.Indexes().DropOne(ctx, "email_1")
```

## Транзакции

MongoDB поддерживает multi-document транзакции (с версии 4.0).

```go
// Начать сессию
session, err := client.StartSession()
if err != nil {
    log.Fatal(err)
}
defer session.EndSession(ctx)

// Транзакция
err = mongo.WithSession(ctx, session, func(sc mongo.SessionContext) error {
    // Начать транзакцию
    if err := session.StartTransaction(); err != nil {
        return err
    }

    // Операции
    _, err := accounts.UpdateOne(sc,
        bson.M{"_id": 1},
        bson.M{"$inc": bson.M{"balance": -100}},
    )
    if err != nil {
        session.AbortTransaction(sc)
        return err
    }

    _, err = accounts.UpdateOne(sc,
        bson.M{"_id": 2},
        bson.M{"$inc": bson.M{"balance": 100}},
    )
    if err != nil {
        session.AbortTransaction(sc)
        return err
    }

    // Commit
    return session.CommitTransaction(sc)
})

if err != nil {
    log.Fatal(err)
}
```

**Ограничения:**
- Требует replica set
- Производительность хуже обычных операций
- Таймаут по умолчанию 60 секунд

## Репликация

Копии данных на нескольких серверах.

```
Primary (master) ← Writes
   ↓
Secondary (replicas) ← Reads
```

**Преимущества:**
- Высокая доступность
- Отказоустойчивость
- Read scaling

**Настройка:**

```bash
# Инициализация replica set
mongosh
rs.initiate()
rs.add("mongodb2.example.com:27017")
rs.add("mongodb3.example.com:27017")
rs.status()
```

**Подключение в Go:**

```go
client, err := mongo.Connect(ctx, options.Client().
    ApplyURI("mongodb://host1:27017,host2:27017,host3:27017/?replicaSet=myReplicaSet"))
```

## Sharding (секционирование)

Распределение данных по серверам для горизонтального масштабирования.

```
Data:
  users 1-1000   → Shard 1
  users 1001-2000 → Shard 2
  users 2001-3000 → Shard 3
```

**Компоненты:**
- **Shard** - сервер с частью данных
- **Config server** - метаданные
- **Mongos** - роутер запросов

**Sharding key:** Поле для распределения (важно выбрать правильно!).

```javascript
// Включить sharding для БД
sh.enableSharding("mydb")

// Sharding для коллекции
sh.shardCollection("mydb.users", { "userId": 1 })
```

## Производительность

### Best Practices

1. ✅ **Индексы на частые запросы**
   ```go
   users.Indexes().CreateOne(ctx, mongo.IndexModel{
       Keys: bson.M{"email": 1},
   })
   ```

2. ✅ **Projection** - выбирать только нужные поля
   ```go
   opts := options.Find().SetProjection(bson.M{"name": 1, "email": 1})
   ```

3. ✅ **Limit** для больших результатов
   ```go
   opts := options.Find().SetLimit(100)
   ```

4. ✅ **Батчинг** для массовых операций
   ```go
   models := []mongo.WriteModel{
       mongo.NewInsertOneModel().SetDocument(bson.M{"name": "Alice"}),
       mongo.NewInsertOneModel().SetDocument(bson.M{"name": "Bob"}),
   }
   users.BulkWrite(ctx, models)
   ```

5. ❌ Избегайте `$where` (выполняет JavaScript)

6. ✅ Используйте **covered queries** (только индекс, без чтения документов)

7. ✅ **Connection pooling** (go-driver делает автоматически)

### Explain

```go
// Анализ запроса
cursor, _ := users.Find(ctx, bson.M{"age": bson.M{"$gt": 25}})
cursor.Close(ctx)

// Получить explain
// (в mongo shell)
db.users.find({age: {$gt: 25}}).explain("executionStats")
```

## Ограничения

- Документ максимум 16MB
- Имя БД до 64 символов
- Имя коллекции до 120 символов
- Максимум 100 уровней вложенности
- Транзакции только в replica set
- JOIN (lookup) медленнее SQL

## Когда использовать MongoDB

✅ **Подходит для:**
- Гибкая схема (часто меняется)
- Вложенные структуры данных
- Каталоги, CMS, блоги
- Логи, события
- Горизонтальное масштабирование

❌ **Не подходит для:**
- Сложные транзакции (ACID критичны)
- Много JOIN между коллекциями
- Строгая схема данных
- Финансовые системы

## Сравнение с PostgreSQL

| Критерий | MongoDB | PostgreSQL |
|----------|---------|------------|
| Схема | Гибкая | Строгая |
| Транзакции | Ограничены | ACID |
| Масштабирование | Горизонтальное | Вертикальное |
| JOIN | Медленный | Быстрый |
| Вложенные данные | ✅ Нативно | JSON/JSONB |
| Производительность (write) | ⚡ Быстро | Средняя |
| Консистентность | Eventual | Strong |

## Связанные темы

- [[Типы СУБД - SQL vs NoSQL]]
- [[PostgreSQL - Основы]]
- [[Redis - Key-Value хранилище]]
- [[CAP теорема]]
