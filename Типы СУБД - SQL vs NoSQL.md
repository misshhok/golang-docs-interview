# Типы СУБД - SQL vs NoSQL

Сравнение реляционных (SQL) и нереляционных (NoSQL) баз данных.

## SQL (реляционные БД)

### Характеристики

- **Структурированные данные:** Таблицы, строки, колонки
- **Схема:** Фиксированная, строгая типизация
- **Связи:** Foreign keys, JOIN
- **Транзакции:** ACID гарантии
- **Язык запросов:** SQL (стандартизирован)

### Примеры

- **PostgreSQL** - самая продвинутая open-source
- **MySQL** - популярная, простая
- **Oracle** - enterprise
- **MS SQL Server** - Microsoft

### Когда использовать SQL

✅ **Подходит для:**
- Финансовые системы (транзакции критичны)
- E-commerce (заказы, инвентарь)
- ERP, CRM системы
- Данные с четкой структурой и связями
- Сложные запросы и аналитика
- ACID критичны

❌ **Не подходит для:**
- Огромный объем данных (> 100TB)
- Очень высокая нагрузка (> 100k RPS)
- Часто меняющаяся схема
- Горизонтальное масштабирование

### Пример: E-commerce

```sql
CREATE TABLE users (
    id serial PRIMARY KEY,
    email varchar(255) UNIQUE,
    name varchar(100)
);

CREATE TABLE orders (
    id serial PRIMARY KEY,
    user_id int REFERENCES users(id),
    total decimal(10,2),
    status varchar(20),
    created_at timestamp
);

CREATE TABLE order_items (
    id serial PRIMARY KEY,
    order_id int REFERENCES orders(id),
    product_id int REFERENCES products(id),
    quantity int,
    price decimal(10,2)
);

-- Сложный запрос с JOIN
SELECT
    u.name,
    o.id AS order_id,
    SUM(oi.quantity * oi.price) AS total
FROM users u
JOIN orders o ON u.id = o.user_id
JOIN order_items oi ON o.id = oi.order_id
WHERE o.created_at > '2024-01-01'
GROUP BY u.id, o.id;
```

## NoSQL (нереляционные БД)

### Характеристики

- **Гибкая схема:** Schema-less или flexible
- **Горизонтальное масштабирование:** Sharding
- **Высокая производительность:** Оптимизировано под конкретные задачи
- **Eventual consistency:** Не всегда ACID
- **Различные модели данных:** Key-Value, Document, Column, Graph

### Типы NoSQL

## 1. Key-Value хранилища

**Модель:** Ключ → Значение (как HashMap)

**Примеры:**
- **Redis** - in-memory, очень быстрый
- **Memcached** - простой кэш
- **DynamoDB** - AWS managed

**Когда использовать:**
- Кэширование
- Сессии
- Счетчики, rate limiting
- Очереди сообщений

```go
// Redis
redis.Set(ctx, "user:1:name", "Alice", 1*time.Hour)
name := redis.Get(ctx, "user:1:name").Val()

// Нет JOIN, нет сложных запросов
```

**Плюсы:**
- ⚡ Очень быстрый (O(1))
- Простая модель
- Легко масштабируется

**Минусы:**
- Нет связей между данными
- Нет сложных запросов
- Только key lookup

## 2. Document хранилища

**Модель:** Документы (JSON/BSON)

**Примеры:**
- **MongoDB** - самый популярный
- **CouchDB** - с репликацией
- **Firestore** - Google managed

**Когда использовать:**
- Гибкая схема (часто меняется)
- Вложенные структуры данных
- Каталоги продуктов
- CMS, блоги
- Профили пользователей

```javascript
// MongoDB
db.users.insertOne({
    name: "Alice",
    email: "alice@example.com",
    address: {
        city: "Moscow",
        street: "Tverskaya 1"
    },
    hobbies: ["reading", "coding"],
    orders: [
        { id: 1, total: 100 },
        { id: 2, total: 200 }
    ]
})

// Запросы
db.users.find({ "address.city": "Moscow" })
db.users.find({ hobbies: "coding" })
```

**Плюсы:**
- Гибкая схема
- Вложенные данные (нет JOIN)
- Быстрый для чтения
- Горизонтальное масштабирование

**Минусы:**
- Денормализация (дублирование данных)
- Eventual consistency
- Сложные транзакции ограничены

## 3. Column-family хранилища

**Модель:** Колонки группируются в семейства

**Примеры:**
- **Cassandra** - распределенная, высокая доступность
- **HBase** - Hadoop ecosystem
- **ScyllaDB** - Cassandra на C++

**Когда использовать:**
- Временные ряды (логи, метрики)
- Аналитика больших данных
- Огромные объемы (> 100TB)
- Высокая запись (> 100k writes/sec)

```sql
-- Cassandra
CREATE TABLE metrics (
    device_id uuid,
    timestamp timestamp,
    temperature double,
    humidity double,
    PRIMARY KEY (device_id, timestamp)
);

INSERT INTO metrics (device_id, timestamp, temperature, humidity)
VALUES (uuid(), '2024-01-09 10:00:00', 22.5, 45.0);

-- Запросы по partition key + clustering key
SELECT * FROM metrics
WHERE device_id = uuid('...')
  AND timestamp > '2024-01-01';
```

**Плюсы:**
- Масштабируется до петабайт
- Высокая запись
- Отказоустойчивость (репликация)

**Минусы:**
- Денормализация обязательна
- Нет JOIN
- Eventual consistency
- Сложная настройка

## 4. Graph БД

**Модель:** Узлы (nodes) и связи (edges)

**Примеры:**
- **Neo4j** - самая популярная
- **ArangoDB** - multi-model
- **JanusGraph** - распределенная

**Когда использовать:**
- Социальные сети (друзья, подписки)
- Рекомендательные системы
- Fraud detection
- Знания графы
- Навигация, маршруты

```cypher
// Neo4j (Cypher)
CREATE (alice:User {name: "Alice"})
CREATE (bob:User {name: "Bob"})
CREATE (alice)-[:FOLLOWS]->(bob)

// Найти друзей друзей
MATCH (me:User {name: "Alice"})-[:FOLLOWS]->(friend)-[:FOLLOWS]->(fof)
WHERE NOT (me)-[:FOLLOWS]->(fof)
RETURN fof.name

// Кратчайший путь
MATCH path = shortestPath(
  (alice:User {name: "Alice"})-[:FOLLOWS*]-(charlie:User {name: "Charlie"})
)
RETURN path
```

**Плюсы:**
- Эффективный обход графа
- Сложные связи просто
- Гибкая схема

**Минусы:**
- Не для всех задач
- Сложнее масштабировать
- Специфичный язык запросов

## Сравнительная таблица

| Критерий | SQL | Key-Value | Document | Column | Graph |
|----------|-----|-----------|----------|--------|-------|
| **Схема** | Строгая | Нет | Гибкая | Гибкая | Гибкая |
| **Масштабирование** | Вертикальное | Горизонтальное | Горизонтальное | Горизонтальное | Среднее |
| **Транзакции** | ACID | Ограничены | Ограничены | Eventual | Ограничены |
| **JOIN** | ✅ | ❌ | ❌ | ❌ | ✅ (граф) |
| **Сложные запросы** | ✅ | ❌ | Средне | ❌ | ✅ (граф) |
| **Скорость (read)** | Средняя | ⚡ Очень быстро | Быстро | Быстро | Средняя |
| **Скорость (write)** | Средняя | ⚡ Очень быстро | Быстро | ⚡ Очень быстро | Средняя |
| **Объем данных** | До 10TB | До 100GB | До 100TB | До петабайт | До 10TB |
| **Примеры** | PostgreSQL | Redis | MongoDB | Cassandra | Neo4j |

## Масштабирование

### Vertical (вертикальное)

**Больше ресурсов одному серверу:** CPU, RAM, SSD

```
1 сервер: 4 CPU, 16GB RAM
↓
1 сервер: 32 CPU, 256GB RAM
```

**Плюсы:**
- Просто (нет изменений в коде)
- ACID сохраняется

**Минусы:**
- Ограничено (дорого)
- Single point of failure

**Типично для:** SQL БД

### Horizontal (горизонтальное)

**Больше серверов:**

```
1 сервер
↓
10 серверов (sharding)
```

**Плюсы:**
- Безграничное масштабирование
- Высокая доступность

**Минусы:**
- Сложность (sharding, consistency)
- Eventual consistency

**Типично для:** NoSQL БД

### Sharding (секционирование)

Разделение данных по серверам.

```
users:
  id 1-1000   → Shard 1
  id 1001-2000 → Shard 2
  id 2001-3000 → Shard 3
```

**Проблемы:**
- Как распределять? (hash, range, geo)
- JOIN между shards сложен
- Транзакции между shards сложны

## CAP теорема

**CAP = Consistency + Availability + Partition tolerance**

Можно выбрать только 2 из 3.

### C - Consistency (согласованность)

Все узлы видят одни данные в один момент.

### A - Availability (доступность)

Каждый запрос получает ответ (success/failure).

### P - Partition tolerance (устойчивость к разделению)

Система работает при сетевых сбоях.

**В реальности:** Сеть всегда может упасть (P обязательно), выбор между C и A.

### CP системы (жертвуем A)

**Пример:** MongoDB (strong consistency mode), HBase

```
При сетевом сбое → возвращают ошибку (не доступны)
Но данные всегда согласованы
```

### AP системы (жертвуем C)

**Пример:** Cassandra, DynamoDB

```
При сетевом сбое → продолжают работать
Но данные могут быть несогласованными (eventual consistency)
```

### CA системы (жертвуем P)

**Пример:** Традиционные SQL БД (single node)

```
Нет разделения сети (один узел)
Согласованность + доступность
Но нет отказоустойчивости
```

## Выбор базы данных

### Вопросы для выбора

1. **Структура данных:**
   - Фиксированная схема? → SQL
   - Гибкая, вложенная? → Document
   - Связи критичны? → SQL или Graph

2. **Объем данных:**
   - < 1TB? → SQL
   - 1-100TB? → Document (MongoDB)
   - > 100TB? → Column (Cassandra)

3. **Нагрузка:**
   - Read-heavy? → Redis (кэш) + SQL
   - Write-heavy? → Column (Cassandra)
   - Balanced? → SQL или Document

4. **Консистентность:**
   - ACID критично? → SQL
   - Eventual OK? → NoSQL

5. **Масштабирование:**
   - Вертикальное достаточно? → SQL
   - Нужно горизонтальное? → NoSQL

6. **Запросы:**
   - Сложные JOIN? → SQL
   - Простые key lookup? → Key-Value
   - Граф обход? → Graph

### Примеры выбора

**E-commerce:**
- PostgreSQL (заказы, инвентарь - ACID)
- Redis (кэш, сессии)
- Elasticsearch (поиск)

**Социальная сеть:**
- PostgreSQL (профили, аутентификация)
- Neo4j (граф друзей)
- Redis (кэш, счетчики)
- Cassandra (лента новостей)

**Аналитика:**
- ClickHouse (аналитические запросы)
- Cassandra (сбор метрик)
- PostgreSQL (агрегаты)

**IoT:**
- Cassandra (метрики с устройств)
- TimescaleDB (временные ряды)
- Redis (реальное время)

## Полиглот персистентность

Использование разных БД для разных задач.

```go
type UserService struct {
    postgres *sql.DB      // Главные данные (ACID)
    redis    *redis.Client // Кэш
    mongo    *mongo.Client // Документы (настройки)
    elastic  *elastic.Client // Полнотекстовый поиск
}

func (s *UserService) GetUser(id int) (*User, error) {
    // 1. Попытка из кэша (Redis)
    cached, err := s.redis.Get(ctx, fmt.Sprintf("user:%d", id)).Result()
    if err == nil {
        var user User
        json.Unmarshal([]byte(cached), &user)
        return &user, nil
    }

    // 2. Из PostgreSQL
    user, err := s.postgres.QueryUser(id)
    if err != nil {
        return nil, err
    }

    // 3. Сохранить в кэш
    data, _ := json.Marshal(user)
    s.redis.Set(ctx, fmt.Sprintf("user:%d", id), data, 1*time.Hour)

    return user, nil
}

func (s *UserService) Search(query string) ([]*User, error) {
    // Поиск через Elasticsearch
    results, err := s.elastic.Search(query)
    return results, err
}
```

## SQL в Go

```go
import "database/sql"
import _ "github.com/lib/pq"

db, _ := sql.Open("postgres", "connection_string")

// Простой запрос
var name string
err := db.QueryRow("SELECT name FROM users WHERE id = $1", 1).Scan(&name)

// Множественные строки
rows, _ := db.Query("SELECT id, name FROM users WHERE age > $1", 18)
defer rows.Close()

for rows.Next() {
    var id int
    var name string
    rows.Scan(&id, &name)
    // ...
}

// Транзакция
tx, _ := db.Begin()
tx.Exec("UPDATE accounts SET balance = balance - 100 WHERE id = $1", 1)
tx.Exec("UPDATE accounts SET balance = balance + 100 WHERE id = $2", 2)
tx.Commit()
```

## NoSQL в Go

```go
import "github.com/go-redis/redis/v8"
import "go.mongodb.org/mongo-driver/mongo"

// Redis
rdb := redis.NewClient(&redis.Options{Addr: "localhost:6379"})
rdb.Set(ctx, "key", "value", 0)
val, _ := rdb.Get(ctx, "key").Result()

// MongoDB
client, _ := mongo.Connect(ctx, options.Client().ApplyURI("mongodb://localhost:27017"))
collection := client.Database("mydb").Collection("users")

// Вставка
collection.InsertOne(ctx, bson.M{"name": "Alice", "age": 30})

// Поиск
var result bson.M
collection.FindOne(ctx, bson.M{"name": "Alice"}).Decode(&result)

// Обновление
collection.UpdateOne(ctx, bson.M{"name": "Alice"}, bson.M{"$set": bson.M{"age": 31}})
```

## Best Practices

1. ✅ Начинайте с SQL (проще, надежнее)
2. ✅ NoSQL когда SQL не справляется
3. ✅ Полиглот персистентность для сложных систем
4. ✅ Redis для кэша всегда
5. ✅ Профилируйте перед выбором (не по хайпу!)
6. ❌ Не переносите реляционную модель в NoSQL
7. ✅ Денормализация в NoSQL - это нормально
8. ✅ Eventual consistency в NoSQL - это нормально
9. ✅ ACID для критичных операций (платежи)
10. ✅ Тестируйте с реальными объемами данных

## Связанные темы

- [[PostgreSQL - Основы]]
- [[Redis - Key-Value хранилище]]
- [[MongoDB - Основы]]
- [[CAP теорема]]
- [[PostgreSQL - ACID]]
