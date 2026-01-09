# Монолит vs Микросервисы

Сравнение монолитной и микросервисной архитектур: когда использовать каждую, преимущества, недостатки и миграция между ними.

## Монолитная архитектура

### Что это?

Монолит — это приложение, в котором весь код находится в одном кодовой базе и деплоится как единое целое.

```
┌─────────────────────────────────────┐
│         Monolithic App              │
│                                     │
│  ┌─────────┐  ┌─────────┐         │
│  │  Users  │  │ Orders  │         │
│  │ Module  │  │ Module  │         │
│  └─────────┘  └─────────┘         │
│  ┌─────────┐  ┌─────────┐         │
│  │Payments │  │  Email  │         │
│  │ Module  │  │ Module  │         │
│  └─────────┘  └─────────┘         │
│                                     │
│        One Database                 │
└─────────────────────────────────────┘
```

### Структура проекта (Монолит)

```
myapp/
├── cmd/
│   └── server/
│       └── main.go          # Одна точка входа
├── internal/
│   ├── user/
│   │   ├── handler.go
│   │   ├── service.go
│   │   └── repository.go
│   ├── order/
│   │   ├── handler.go
│   │   ├── service.go
│   │   └── repository.go
│   ├── payment/
│   │   ├── handler.go
│   │   ├── service.go
│   │   └── repository.go
│   └── database/
│       └── postgres.go      # Одна БД
├── go.mod
└── Dockerfile               # Один docker image
```

### Код (Монолит)

```go
// main.go - единая точка входа
package main

import (
    "myapp/internal/user"
    "myapp/internal/order"
    "myapp/internal/payment"
    "myapp/internal/database"
)

func main() {
    // Одна БД для всех модулей
    db := database.Connect()

    // Все модули работают с одной БД
    userService := user.NewService(db)
    orderService := order.NewService(db, userService)
    paymentService := payment.NewService(db, orderService)

    // HTTP handlers
    http.HandleFunc("/users", userService.Handler)
    http.HandleFunc("/orders", orderService.Handler)
    http.HandleFunc("/payments", paymentService.Handler)

    // Один процесс, один порт
    http.ListenAndServe(":8080", nil)
}
```

**Транзакции в монолите (легко):**

```go
// Создание заказа с оплатой в одной транзакции
func (s *OrderService) CreateOrderWithPayment(order *Order, payment *Payment) error {
    tx, err := s.db.Begin()
    if err != nil {
        return err
    }
    defer tx.Rollback()

    // Резервируем товар
    _, err = tx.Exec("UPDATE inventory SET quantity = quantity - ? WHERE product_id = ?",
        order.Quantity, order.ProductID)
    if err != nil {
        return err
    }

    // Создаем заказ
    result, err := tx.Exec("INSERT INTO orders (user_id, amount) VALUES (?, ?)",
        order.UserID, order.Amount)
    if err != nil {
        return err
    }
    orderID, _ := result.LastInsertId()

    // Создаем платеж
    _, err = tx.Exec("INSERT INTO payments (order_id, amount) VALUES (?, ?)",
        orderID, payment.Amount)
    if err != nil {
        return err
    }

    // Все или ничего - атомарность гарантирована
    return tx.Commit()
}
```

### ✅ Преимущества монолита

1. **Простота разработки**
   ```go
   // Просто вызываем функцию
   user := userService.GetUser(123)
   order := orderService.CreateOrder(user.ID, amount)
   // Нет сетевых запросов, нет латентности
   ```

2. **Простота деплоя**
   ```bash
   # Один docker image
   docker build -t myapp:v1.0 .
   docker push myapp:v1.0
   docker run myapp:v1.0
   ```

3. **Легкие транзакции (ACID)**
   - Одна БД → легко обеспечить ACID
   - Нет распределенных транзакций

4. **Простое тестирование**
   ```go
   // Интеграционный тест - просто запускаем приложение
   func TestCreateOrder(t *testing.T) {
       app := setupTestApp()

       user := app.UserService.CreateUser(...)
       order := app.OrderService.CreateOrder(user.ID, ...)

       // Все в одном процессе
       assert.NotNil(t, order)
   }
   ```

5. **Простая отладка**
   - Один процесс → легко дебажить
   - Stack trace через все модули

6. **Меньше operational overhead**
   - Один сервис для мониторинга
   - Проще логирование

### ❌ Недостатки монолита

1. **Сложность при росте**
   ```go
   // main.go становится огромным
   // 50+ зависимостей
   userService := user.NewService(db, logger, cache, mailer, ...)
   orderService := order.NewService(db, logger, cache, userService, inventoryService, ...)
   // ...еще 20 сервисов
   ```

2. **Долгий деплой**
   - Изменили 1 строку → пересобираем весь монолит
   - Долгие CI/CD пайплайны (10-30 минут)

3. **Масштабирование только вертикальное**
   ```
   Orders - высокая нагрузка (100k RPS)
   Users - низкая нагрузка (1k RPS)

   Но масштабировать приходится весь монолит
   → нужно больше CPU/RAM для всего приложения
   ```

4. **Плохая изоляция ошибок**
   ```go
   // Утечка памяти в модуле Orders
   // → роняет весь монолит, включая Users, Payments и тд
   ```

5. **Технологический lock-in**
   - Вся кодовая база на одном языке
   - Нельзя использовать лучшие инструменты для разных задач

6. **Большие команды**
   - 50 разработчиков работают с одним репозиторием
   - Merge conflicts, медленные code reviews

## Микросервисная архитектура

### Что это?

Микросервисы — это множество независимых сервисов, каждый со своей БД и деплоем.

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│  User    │────▶│  Order   │────▶│ Payment  │
│ Service  │     │ Service  │     │ Service  │
│          │     │          │     │          │
│ Postgres │     │  MySQL   │     │ Postgres │
└──────────┘     └──────────┘     └──────────┘
     │                 │                 │
     └─────────────────┴─────────────────┘
                  Message Queue
```

### Структура проектов (Микросервисы)

```
project/
├── user-service/
│   ├── cmd/main.go
│   ├── internal/...
│   ├── go.mod
│   ├── Dockerfile
│   └── k8s/deployment.yaml
├── order-service/
│   ├── cmd/main.go
│   ├── internal/...
│   ├── go.mod
│   ├── Dockerfile
│   └── k8s/deployment.yaml
└── payment-service/
    ├── cmd/main.go
    ├── internal/...
    ├── go.mod
    ├── Dockerfile
    └── k8s/deployment.yaml
```

### Код (Микросервисы)

```go
// user-service/cmd/main.go
package main

func main() {
    db := postgres.Connect()
    userService := user.NewService(db)

    http.HandleFunc("/users", userService.Handler)
    http.ListenAndServe(":8081", nil) // Свой порт
}

// order-service/cmd/main.go
package main

func main() {
    db := mysql.Connect()

    // HTTP Client для взаимодействия с User Service
    userClient := user.NewClient("http://user-service:8081")

    orderService := order.NewService(db, userClient)

    http.HandleFunc("/orders", orderService.Handler)
    http.ListenAndServe(":8082", nil) // Свой порт
}
```

**Транзакции в микросервисах (сложно):**

```go
// Saga Pattern - нужна компенсация при ошибке
func (s *OrderService) CreateOrderWithPayment(order *Order, payment *Payment) error {
    // 1. Резервируем товар (Inventory Service)
    err := s.inventoryClient.Reserve(order.ProductID, order.Quantity)
    if err != nil {
        return err
    }

    // 2. Создаем заказ в нашей БД
    orderID, err := s.db.CreateOrder(order)
    if err != nil {
        // Компенсация: отменяем резервацию
        s.inventoryClient.CancelReserve(order.ProductID, order.Quantity)
        return err
    }

    // 3. Проводим платеж (Payment Service)
    err = s.paymentClient.Charge(payment)
    if err != nil {
        // Компенсация: отменяем резервацию и удаляем заказ
        s.inventoryClient.CancelReserve(order.ProductID, order.Quantity)
        s.db.DeleteOrder(orderID)
        return err
    }

    return nil
}
```

### ✅ Преимущества микросервисов

1. **Независимый деплой**
   ```bash
   # Обновляем только Order Service
   cd order-service
   docker build -t order-service:v2.0 .
   kubectl set image deployment/order-service order-service=order-service:v2.0

   # User Service и Payment Service продолжают работать на старой версии
   ```

2. **Масштабирование по требованию**
   ```yaml
   # Kubernetes - масштабируем только Order Service
   kubectl scale deployment order-service --replicas=10
   kubectl scale deployment user-service --replicas=2

   # Order Service получает больше ресурсов
   # User Service остается с минимальными ресурсами
   ```

3. **Технологическое разнообразие**
   ```go
   // User Service - Go + PostgreSQL
   // Order Service - Go + MySQL
   // Analytics Service - Python + ClickHouse
   // Recommendation Service - Python + TensorFlow
   ```

4. **Изоляция ошибок**
   ```go
   // Order Service упал → User Service продолжает работать
   // Circuit Breaker защищает от каскадных сбоев
   ```

5. **Небольшие автономные команды**
   - Команда A → User Service
   - Команда B → Order Service
   - Команда C → Payment Service

6. **Легче понимать код**
   - Каждый сервис маленький и фокусируется на одном домене

### ❌ Недостатки микросервисов

1. **Сложность взаимодействия**
   ```go
   // Монолит: прямой вызов функции
   user := userService.GetUser(123)

   // Микросервисы: HTTP запрос с обработкой ошибок
   user, err := userClient.GetUser(ctx, 123)
   if err != nil {
       // Network error? Service down? Timeout?
       // Нужен retry, circuit breaker, fallback
   }
   ```

2. **Сложность транзакций**
   - Нет ACID между сервисами
   - Нужны Saga, Compensating Transactions
   - Eventual consistency

3. **Латентность**
   ```
   Монолит: 1ms (вызов функции в памяти)
   Микросервисы: 10-50ms (HTTP/gRPC через сеть)
   ```

4. **Сложность тестирования**
   ```go
   // Нужно поднимать все зависимые сервисы
   func TestCreateOrder(t *testing.T) {
       userService := startUserService()
       inventoryService := startInventoryService()
       paymentService := startPaymentService()
       orderService := startOrderService()

       // Много boilerplate
   }
   ```

5. **Operational overhead**
   - 20 микросервисов → 20 deployments, 20 логов, 20 метрик
   - Нужны: Service Discovery, API Gateway, Distributed Tracing

6. **Дублирование кода**
   ```go
   // Каждый сервис дублирует логирование, аутентификацию, метрики
   // Нужны shared libraries
   ```

## Сравнительная таблица

| Критерий | Монолит | Микросервисы |
|----------|---------|--------------|
| **Сложность разработки** | Низкая | Высокая |
| **Скорость разработки (стартап)** | Быстрая | Медленная |
| **Скорость деплоя** | Медленная | Быстрая |
| **Масштабирование** | Вертикальное | Горизонтальное |
| **Транзакции** | ACID легко | Сложно (Saga) |
| **Латентность** | Низкая (in-memory) | Выше (network) |
| **Тестирование** | Легко | Сложно |
| **Технологии** | Одна стека | Разные стеки |
| **Команды** | Большая команда | Малые автономные команды |
| **Изоляция ошибок** | Плохая | Хорошая |
| **Operational overhead** | Низкий | Высокий |
| **Подходит для** | Малые проекты, стартапы | Большие проекты, энтерпрайз |

## Когда использовать монолит

**✅ Используйте монолит когда:**

1. **Стартап / MVP**
   - Быстрый time-to-market важнее масштабируемости
   - Границы доменов неясны

2. **Малая команда (< 5-10 разработчиков)**
   - Не хватит людей для поддержки множества сервисов

3. **Простое CRUD приложение**
   - Нет сложной бизнес-логики
   - Нет высоких требований к масштабированию

4. **Нет опыта с микросервисами**
   - Команда не знакома с распределенными системами
   - Нет DevOps экспертизы

5. **Низкая/средняя нагрузка**
   - < 1000 RPS
   - Вертикальное масштабирование достаточно

**Примеры:**
- Внутренний admin panel
- Простой интернет-магазин (< 1000 заказов/день)
- Blog платформа
- CRM для малого бизнеса

## Когда использовать микросервисы

**✅ Используйте микросервисы когда:**

1. **Большая команда (> 10-20 разработчиков)**
   - Несколько команд работают параллельно
   - Каждая команда владеет своим доменом

2. **Высокая нагрузка с разными паттернами**
   - Orders: 100k RPS
   - Users: 1k RPS
   - Нужно масштабировать только Orders

3. **Четкие границы доменов**
   - E-commerce: Users, Orders, Inventory, Payments
   - Социальная сеть: Users, Posts, Comments, Notifications

4. **Нужна независимая deployability**
   - Релизы каждый день/каждую неделю
   - Разные команды релизят независимо

5. **Разные технологические требования**
   - OLTP (PostgreSQL) + OLAP (ClickHouse)
   - Web API (Go) + ML (Python)

**Примеры:**
- Uber, Яндекс, Amazon
- Крупный e-commerce (> 10k заказов/день)
- Банковская система
- Социальная сеть

## Миграция: монолит → микросервисы

### Стратегия: Strangler Fig Pattern

Постепенно "выращиваем" микросервисы вокруг монолита.

```
Этап 1: Монолит
┌─────────────────────────┐
│      Monolith           │
│  Users | Orders | Pay   │
└─────────────────────────┘

Этап 2: Выделяем первый сервис
┌───────────────┐     ┌──────────┐
│   Monolith    │────▶│  User    │
│ Orders | Pay  │     │ Service  │
└───────────────┘     └──────────┘

Этап 3: Выделяем следующие
┌──────────┐     ┌──────────┐
│ User     │     │  Order   │
│ Service  │◀────│ Service  │
└──────────┘     └──────────┘
                      ▲
┌──────────┐          │
│ Payment  │──────────┘
│ Service  │
└──────────┘

Этап 4: Монолит исчез
```

### Пошаговый план миграции

**Шаг 1: Определите границы**

```go
// Монолит - плохо разделенный код
func CreateOrder(userID, productID int, amount float64) error {
    // Все в одной функции
    user := db.Query("SELECT * FROM users WHERE id = ?", userID)
    product := db.Query("SELECT * FROM products WHERE id = ?", productID)
    db.Exec("INSERT INTO orders ...")
    db.Exec("UPDATE inventory SET quantity = quantity - 1...")
    sendEmail(user.Email, "Order created")
}

// Шаг 1: Разделите на модули
type UserModule struct { db *sql.DB }
type OrderModule struct { db *sql.DB }
type InventoryModule struct { db *sql.DB }
type EmailModule struct { smtp *SMTP }

func (m *OrderModule) CreateOrder(...) {
    user := userModule.GetUser(userID)
    product := inventoryModule.GetProduct(productID)
    // ...
}
```

**Шаг 2: Выделите первый сервис**

```go
// Начните с независимого сервиса (например, Email)
// Email Service - не имеет зависимостей
package main

func main() {
    rabbitMQ := connectToRabbitMQ()

    // Слушаем события из монолита
    messages := rabbitMQ.Consume("email-queue")

    for msg := range messages {
        var event EmailEvent
        json.Unmarshal(msg.Body, &event)
        sendEmail(event.To, event.Subject, event.Body)
    }
}

// Монолит публикует события вместо прямого вызова
func (m *OrderModule) CreateOrder(...) {
    // ...
    // Вместо: emailModule.Send(...)
    // Публикуем событие
    rabbitMQ.Publish("email-queue", EmailEvent{
        To: user.Email,
        Subject: "Order created",
    })
}
```

**Шаг 3: API Gateway для роутинга**

```go
// API Gateway перенаправляет на монолит или микросервис
func (gw *APIGateway) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    switch {
    case r.URL.Path == "/users":
        // Пока в монолите
        gw.proxyTo(monolithURL, w, r)
    case r.URL.Path == "/orders":
        // Уже выделен в микросервис
        gw.proxyTo(orderServiceURL, w, r)
    }
}
```

**Шаг 4: Разделите базу данных**

```sql
-- Монолит: одна БД
CREATE TABLE users (id, name, email);
CREATE TABLE orders (id, user_id, amount);
CREATE TABLE products (id, name, price);

-- Этап 1: Разделите схемы
CREATE SCHEMA users;
CREATE SCHEMA orders;

CREATE TABLE users.users (id, name, email);
CREATE TABLE orders.orders (id, user_id, amount);

-- Этап 2: Разные БД
-- users_db: users table
-- orders_db: orders table (без FK на users!)
```

**Шаг 5: Замените прямые вызовы на HTTP/gRPC**

```go
// До: прямой вызов функции
user := userModule.GetUser(userID)

// После: HTTP запрос
user, err := userServiceClient.GetUser(ctx, userID)
if err != nil {
    // Обработка ошибок сети
    return err
}
```

## Модульный монолит (компромисс)

Модульный монолит — это монолит с четкими границами модулей.

```go
// Модульный монолит
myapp/
├── cmd/main.go              # Одна точка входа
├── modules/
│   ├── user/
│   │   ├── api/             # Public API
│   │   │   └── service.go
│   │   └── internal/        # Скрыто от других модулей
│   │       ├── repository.go
│   │       └── domain.go
│   ├── order/
│   │   ├── api/
│   │   │   └── service.go
│   │   └── internal/
│   └── payment/
│       ├── api/
│       └── internal/
```

**Преимущества:**
- Четкие границы между модулями
- Легко мигрировать в микросервисы позже
- Простота монолита + структура микросервисов

```go
// Order module может использовать только API User module
import "myapp/modules/user/api"

// ❌ НЕЛЬЗЯ:
// import "myapp/modules/user/internal" // Запрещено!

func (s *OrderService) CreateOrder(order *Order) error {
    // Используем только публичный API
    user, err := userapi.GetUser(order.UserID)
    // ...
}
```

## Best Practices

### Для монолита

1. ✅ **Разделяйте на модули** - подготовка к возможной миграции
2. ✅ **Используйте интерфейсы** - абстракция от реализации
3. ✅ **Избегайте циклических зависимостей**
4. ✅ **Используйте feature flags** для независимого включения функций

### Для микросервисов

1. ✅ **Start with monolith** - не начинайте с микросервисов
2. ✅ **Database per service** - каждый сервис со своей БД
3. ✅ **Async communication** - предпочитайте message queues
4. ✅ **Distributed tracing** - обязательно для отладки
5. ✅ **Circuit breakers** - защита от каскадных сбоев

### Общие

1. ✅ **Conway's Law** - архитектура отражает структуру команды
2. ✅ **Domain-Driven Design** - разделяйте по бизнес-доменам
3. ✅ **Не переоптимизируйте заранее** - решайте реальные проблемы

## Связанные темы

- [[Микросервисная архитектура]]
- [[Service Mesh]]
- [[REST API - Основы]]
- [[gRPC]]
- [[RabbitMQ]]
- [[Apache Kafka]]
- [[Docker - Основы]]
- [[Kubernetes - Основы]]
- [[CAP теорема]]
