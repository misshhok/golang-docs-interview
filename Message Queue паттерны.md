# Message Queue паттерны

Распространенные паттерны использования message queues (RabbitMQ, Kafka) в микросервисной архитектуре.

## 1. Point-to-Point (Очередь задач)

Один producer, один consumer (или несколько с распределением).

```
Producer → Queue → Consumer 1 (обрабатывает msg1, msg3)
                → Consumer 2 (обрабатывает msg2, msg4)
```

**Использование:**
- Task queue (отправка email, генерация отчетов)
- Background jobs

**RabbitMQ:**

```go
// Producer
ch.Publish("", "tasks", false, false, amqp.Publishing{
    Body: []byte("Send email to user@example.com"),
})

// Consumer 1
msgs, _ := ch.Consume("tasks", "", false, false, false, false, nil)
for msg := range msgs {
    processTask(msg.Body)
    msg.Ack(false)
}

// Consumer 2 - автоматически получает другие сообщения
```

**Kafka:**

```go
// Consumer Group - автоматическое распределение partitions
group, _ := sarama.NewConsumerGroup([]string{"localhost:9092"}, "worker-group", config)
group.Consume(ctx, []string{"tasks"}, handler)
```

## 2. Publish-Subscribe (Pub/Sub)

Один producer, множество consumers (все получают все сообщения).

```
Producer → Exchange (fanout) → Queue 1 (Email)  → Consumer 1
                             → Queue 2 (SMS)    → Consumer 2
                             → Queue 3 (Push)   → Consumer 3
```

**Использование:**
- Рассылка событий (user.created → email + push + analytics)
- Notifications

**RabbitMQ (Fanout Exchange):**

```go
// Объявляем fanout exchange
ch.ExchangeDeclare("events", "fanout", true, false, false, false, nil)

// Producer
ch.Publish("events", "", false, false, amqp.Publishing{
    Body: []byte(`{"event": "user.created", "userId": 123}`),
})

// Consumer 1 (Email Service)
queue1, _ := ch.QueueDeclare("email-queue", true, false, false, false, nil)
ch.QueueBind("email-queue", "", "events", false, nil)

// Consumer 2 (SMS Service)
queue2, _ := ch.QueueDeclare("sms-queue", true, false, false, false, nil)
ch.QueueBind("sms-queue", "", "events", false, nil)

// Все consumers получат сообщение
```

**Kafka (Multiple Consumer Groups):**

```go
// Consumer Group 1 (Email Service)
emailGroup, _ := sarama.NewConsumerGroup([]string{"localhost:9092"}, "email-service", config)
emailGroup.Consume(ctx, []string{"user-events"}, emailHandler)

// Consumer Group 2 (SMS Service)
smsGroup, _ := sarama.NewConsumerGroup([]string{"localhost:9092"}, "sms-service", config)
smsGroup.Consume(ctx, []string{"user-events"}, smsHandler)

// Оба получат все сообщения
```

## 3. Request-Reply (RPC)

Синхронная коммуникация через message queue.

```
Client → Request Queue → Server
   ↑                        ↓
   └────── Reply Queue ─────┘
```

**RabbitMQ:**

```go
// Server
msgs, _ := ch.Consume("rpc_queue", "", false, false, false, false, nil)
for msg := range msgs {
    // Обрабатываем запрос
    result := processRequest(msg.Body)

    // Отправляем ответ
    ch.Publish("", msg.ReplyTo, false, false, amqp.Publishing{
        CorrelationId: msg.CorrelationId,
        Body:          result,
    })

    msg.Ack(false)
}

// Client
replyQueue, _ := ch.QueueDeclare("", false, false, true, false, nil)
corrID := uuid.New().String()

// Отправляем запрос
ch.Publish("", "rpc_queue", false, false, amqp.Publishing{
    ReplyTo:       replyQueue.Name,
    CorrelationId: corrID,
    Body:          []byte("request data"),
})

// Ждем ответ
msgs, _ := ch.Consume(replyQueue.Name, "", true, false, false, false, nil)
for msg := range msgs {
    if msg.CorrelationId == corrID {
        processResponse(msg.Body)
        break
    }
}
```

**Проблемы:**
- Блокирует клиента
- Timeout issues
- Лучше использовать gRPC для синхронного RPC

## 4. Routing (Маршрутизация)

Маршрутизация по ключу или паттерну.

```
Producer → Exchange (topic) → Queue 1 (order.*)
                            → Queue 2 (order.created)
                            → Queue 3 (*.created)
```

**RabbitMQ (Topic Exchange):**

```go
// Объявляем topic exchange
ch.ExchangeDeclare("events", "topic", true, false, false, false, nil)

// Producer с routing key
ch.Publish("events", "order.created", false, false, amqp.Publishing{
    Body: []byte(`{"orderId": 123}`),
})

ch.Publish("events", "user.created", false, false, amqp.Publishing{
    Body: []byte(`{"userId": 456}`),
})

// Consumer 1 - все события заказов
ch.QueueBind("order-events", "order.*", "events", false, nil)
// Получит: order.created, order.paid, order.shipped

// Consumer 2 - все события создания
ch.QueueBind("created-events", "*.created", "events", false, nil)
// Получит: order.created, user.created

// Consumer 3 - все события
ch.QueueBind("all-events", "#", "events", false, nil)
// Получит все
```

**Kafka (Partition Key):**

```go
// Producer с ключом
producer.Send(&sarama.ProducerMessage{
    Topic: "events",
    Key:   sarama.StringEncoder("order"),  // Все order события → в один partition
    Value: value,
})

// Consumer читает specific partitions
partitionConsumer, _ := consumer.ConsumePartition("events", 0, sarama.OffsetNewest)
```

## 5. Priority Queue (Приоритеты)

Сообщения с высоким приоритетом обрабатываются первыми.

```
Producer → Priority Queue → Consumer (обрабатывает по приоритету)
           [High: 10]
           [Normal: 5]
           [Low: 1]
```

**RabbitMQ:**

```go
// Объявляем очередь с приоритетами
ch.QueueDeclare("tasks", true, false, false, false, amqp.Table{
    "x-max-priority": 10,
})

// Отправляем с приоритетом
ch.Publish("", "tasks", false, false, amqp.Publishing{
    Priority: 10, // Высокий приоритет
    Body:     []byte("Urgent task"),
})

ch.Publish("", "tasks", false, false, amqp.Publishing{
    Priority: 1, // Низкий приоритет
    Body:     []byte("Normal task"),
})

// Consumer обработает urgent task первым
```

**Kafka:**
Нет встроенной поддержки приоритетов. Решения:
- Разные topics (high-priority-tasks, low-priority-tasks)
- Custom consumer logic

## 6. Dead Letter Queue (DLQ)

Неудачные сообщения отправляются в отдельную очередь.

```
Producer → Queue → Consumer
             ↓ (failed after N retries)
          Dead Letter Queue → Manual processing
```

**RabbitMQ:**

```go
// DLX (Dead Letter Exchange)
ch.ExchangeDeclare("dlx", "direct", true, false, false, false, nil)

// DLQ
ch.QueueDeclare("dead-letters", true, false, false, false, nil)
ch.QueueBind("dead-letters", "", "dlx", false, nil)

// Основная очередь с DLX
ch.QueueDeclare("tasks", true, false, false, false, amqp.Table{
    "x-dead-letter-exchange": "dlx",
})

// Consumer
for msg := range msgs {
    err := processMessage(msg)
    if err != nil {
        if retries < maxRetries {
            msg.Nack(false, true) // Requeue
        } else {
            msg.Reject(false) // → DLQ
        }
    } else {
        msg.Ack(false)
    }
}
```

**Kafka:**
Нужно реализовывать вручную:

```go
// Consumer
for msg := range messages {
    err := processMessage(msg)
    if err != nil {
        if retries < maxRetries {
            // Retry
            retryProducer.Send("tasks", msg.Key, msg.Value)
        } else {
            // DLQ
            dlqProducer.Send("tasks-dlq", msg.Key, msg.Value)
        }
    }
    session.MarkMessage(msg, "")
}
```

## 7. Delayed Queue (Отложенные сообщения)

Сообщение обрабатывается через определенное время.

```
Producer → Delayed Queue (TTL=5min) → Consumer (через 5 минут)
```

**RabbitMQ (TTL + DLX):**

```go
// Delay Exchange
ch.ExchangeDeclare("delayed", "direct", true, false, false, false, nil)

// Delay Queue (сообщения истекают через 5 минут)
ch.QueueDeclare("delay-5min", true, false, false, false, amqp.Table{
    "x-message-ttl":          300000, // 5 минут
    "x-dead-letter-exchange": "tasks",
})

ch.QueueBind("delay-5min", "", "delayed", false, nil)

// Producer отправляет в delay queue
ch.Publish("delayed", "", false, false, amqp.Publishing{
    Body: []byte("Process this in 5 minutes"),
})

// После TTL → сообщение попадает в "tasks" queue
```

**Kafka:**
Нет встроенной поддержки. Решения:
- Храним timestamp в сообщении, consumer проверяет
- Используем Kafka Streams для windowing

## 8. Event Sourcing (Лог событий)

Все изменения сохраняются как события.

```
Order Service → Kafka Topic (order-events)
                   ↓
              [OrderCreated]
              [OrderPaid]
              [OrderShipped]
                   ↓
              Read Model Service (восстанавливает состояние)
```

**Kafka:**

```go
// Producer - сохраняем все события
type OrderEvent struct {
    Type      string // "created", "paid", "shipped"
    OrderID   int
    Data      json.RawMessage
    Timestamp time.Time
}

producer.Send(&sarama.ProducerMessage{
    Topic: "order-events",
    Key:   sarama.StringEncoder(fmt.Sprintf("order-%d", orderID)),
    Value: marshalEvent(OrderCreatedEvent{...}),
})

producer.Send(&sarama.ProducerMessage{
    Topic: "order-events",
    Key:   sarama.StringEncoder(fmt.Sprintf("order-%d", orderID)),
    Value: marshalEvent(OrderPaidEvent{...}),
})

// Consumer - восстанавливает состояние
events := readAllEvents(orderID)
order := replayEvents(events)
```

**Kafka Log Compaction:**

```bash
# Храним только последнее состояние каждого ключа
kafka-topics.sh --alter --topic order-snapshots \
  --config cleanup.policy=compact
```

## 9. Saga Pattern (Распределенная транзакция)

Координация множественных операций между сервисами.

### Choreography-based Saga

Каждый сервис слушает события и публикует свои.

```
Order Service → order.created → Inventory Service
                                       ↓
                              inventory.reserved → Payment Service
                                                         ↓
                                                 payment.completed → Order Service
                                                                         ↓
                                                                  order.completed
```

**Kafka:**

```go
// Order Service
producer.Send("events", "", OrderCreatedEvent{OrderID: 123, UserID: 456})

// Inventory Service слушает order.created
for msg := range consumer.Messages() {
    if msg.Type == "order.created" {
        reserveInventory(msg.OrderID)
        producer.Send("events", "", InventoryReservedEvent{OrderID: 123})
    }
}

// Payment Service слушает inventory.reserved
for msg := range consumer.Messages() {
    if msg.Type == "inventory.reserved" {
        chargePayment(msg.OrderID)
        producer.Send("events", "", PaymentCompletedEvent{OrderID: 123})
    }
}

// Order Service слушает payment.completed
for msg := range consumer.Messages() {
    if msg.Type == "payment.completed" {
        completeOrder(msg.OrderID)
    }
}
```

**Компенсация при ошибке:**

```go
// Payment Service failed
producer.Send("events", "", PaymentFailedEvent{OrderID: 123})

// Inventory Service слушает payment.failed и откатывает
for msg := range consumer.Messages() {
    if msg.Type == "payment.failed" {
        cancelReservation(msg.OrderID)
        producer.Send("events", "", InventoryCancelledEvent{OrderID: 123})
    }
}
```

### Orchestration-based Saga

Центральный orchestrator управляет процессом.

```
Saga Orchestrator
    ↓ command
Inventory Service → event → Saga Orchestrator
    ↓ command
Payment Service → event → Saga Orchestrator
    ↓ command
Order Service
```

```go
// Saga Orchestrator
type SagaOrchestrator struct {
    inventoryClient *InventoryClient
    paymentClient   *PaymentClient
    orderClient     *OrderClient
}

func (s *SagaOrchestrator) CreateOrder(order *Order) error {
    // Step 1: Reserve inventory
    err := s.inventoryClient.Reserve(order.ProductID, order.Quantity)
    if err != nil {
        return err
    }

    // Step 2: Charge payment
    err = s.paymentClient.Charge(order.UserID, order.Amount)
    if err != nil {
        // Compensate: cancel reservation
        s.inventoryClient.CancelReservation(order.ProductID, order.Quantity)
        return err
    }

    // Step 3: Complete order
    err = s.orderClient.Complete(order.ID)
    if err != nil {
        // Compensate: refund + cancel reservation
        s.paymentClient.Refund(order.UserID, order.Amount)
        s.inventoryClient.CancelReservation(order.ProductID, order.Quantity)
        return err
    }

    return nil
}
```

## 10. CQRS (Command Query Responsibility Segregation)

Разделение команд (write) и запросов (read).

```
Command Service (Write) → Kafka → Read Model Service
         ↓                          ↓
     Postgres                  Elasticsearch
     (source of truth)         (optimized for queries)
```

**Kafka:**

```go
// Command Service - пишет в Postgres + публикует событие
func (s *CommandService) CreateUser(user *User) error {
    // 1. Сохраняем в Postgres
    err := s.db.Insert(user)
    if err != nil {
        return err
    }

    // 2. Публикуем событие
    s.producer.Send("user-events", user.ID, UserCreatedEvent{
        UserID: user.ID,
        Name:   user.Name,
        Email:  user.Email,
    })

    return nil
}

// Read Model Service - слушает события и обновляет Elasticsearch
func (s *ReadModelService) Start() {
    for msg := range s.consumer.Messages() {
        var event UserCreatedEvent
        json.Unmarshal(msg.Value, &event)

        // Обновляем read model
        s.elasticsearch.Index("users", event)
    }
}

// Query Service - читает из Elasticsearch
func (s *QueryService) SearchUsers(query string) ([]*User, error) {
    return s.elasticsearch.Search("users", query)
}
```

## Best Practices

### 1. Idempotency (Идемпотентность)

Обработка одного сообщения несколько раз дает тот же результат.

```go
// Плохо - не идемпотентно
func processOrder(msg *Message) {
    balance += msg.Amount // Повторная обработка увеличит баланс повторно!
}

// Хорошо - идемпотентно
func processOrder(msg *Message) {
    if s.db.OrderProcessed(msg.OrderID) {
        return // Уже обработано
    }

    balance += msg.Amount
    s.db.MarkAsProcessed(msg.OrderID)
}
```

### 2. At-Least-Once Delivery

Гарантия доставки хотя бы один раз.

```go
// Manual Ack после обработки
for msg := range msgs {
    err := processMessage(msg)
    if err != nil {
        msg.Nack(false, true) // Requeue
    } else {
        msg.Ack(false) // Только после успешной обработки
    }
}
```

### 3. Message Schema Versioning

Версионирование формата сообщений.

```go
type MessageV1 struct {
    Version int    `json:"version"` // 1
    OrderID int    `json:"order_id"`
    Amount  float64 `json:"amount"`
}

type MessageV2 struct {
    Version  int    `json:"version"` // 2
    OrderID  int    `json:"order_id"`
    Amount   float64 `json:"amount"`
    Currency string `json:"currency"` // Новое поле
}

// Consumer поддерживает обе версии
func processMessage(data []byte) {
    var base struct {
        Version int `json:"version"`
    }
    json.Unmarshal(data, &base)

    switch base.Version {
    case 1:
        var msg MessageV1
        json.Unmarshal(data, &msg)
        processV1(msg)
    case 2:
        var msg MessageV2
        json.Unmarshal(data, &msg)
        processV2(msg)
    }
}
```

### 4. Correlation ID

Трейсинг запроса через микросервисы.

```go
// Producer добавляет correlation ID
ch.Publish("events", "order.created", false, false, amqp.Publishing{
    CorrelationId: uuid.New().String(),
    Headers: amqp.Table{
        "trace-id": traceID,
    },
    Body: body,
})

// Consumer логирует с correlation ID
for msg := range msgs {
    log.WithFields(log.Fields{
        "correlation_id": msg.CorrelationId,
        "trace_id":       msg.Headers["trace-id"],
    }).Info("Processing message")

    processMessage(msg)
}
```

### 5. Circuit Breaker для Producer

Защита от перегрузки.

```go
type ProducerWithCircuitBreaker struct {
    producer *Producer
    breaker  *CircuitBreaker
}

func (p *ProducerWithCircuitBreaker) Send(msg *Message) error {
    return p.breaker.Execute(func() error {
        return p.producer.Send(msg)
    })
}
```

## Сравнение паттернов

| Паттерн | Use Case | Coupling | Сложность |
|---------|----------|----------|-----------|
| **Point-to-Point** | Task queues | Низкое | Низкая |
| **Pub/Sub** | Event broadcasting | Низкое | Низкая |
| **Request-Reply** | Sync RPC | Высокое | Средняя |
| **Routing** | Conditional routing | Среднее | Средняя |
| **Priority Queue** | Priority tasks | Низкое | Низкая |
| **DLQ** | Error handling | Низкое | Средняя |
| **Delayed Queue** | Scheduled tasks | Низкое | Средняя |
| **Event Sourcing** | Audit trail, replay | Низкое | Высокая |
| **Saga** | Distributed transactions | Среднее | Высокая |
| **CQRS** | Read/Write separation | Низкое | Высокая |

## Связанные темы

- [[RabbitMQ]]
- [[Apache Kafka]]
- [[Микросервисная архитектура]]
- [[Монолит vs Микросервисы]]
- [[REST API - Основы]]
- [[gRPC]]
- [[Service Mesh]]
