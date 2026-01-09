# RabbitMQ

RabbitMQ — популярный message broker (брокер сообщений), реализующий протокол AMQP. Используется для асинхронной коммуникации между микросервисами.

## Зачем нужен Message Broker?

### Без Message Broker (синхронная коммуникация)

```
Order Service → (HTTP) → Email Service
                  ↓
              Блокировка!

Проблемы:
- Order Service ждет ответа от Email Service
- Если Email Service недоступен → Order Service падает
- Tight coupling
```

### С Message Broker (асинхронная коммуникация)

```
Order Service → RabbitMQ → Email Service
                  ↓           ↓
              Возвращается  Обрабатывает
              сразу         когда готов

Преимущества:
- Loose coupling
- Email Service может быть недоступен временно
- Order Service не ждет
- Возможность retry
```

## Основные концепции

### Producer (Продюсер)

Отправляет сообщения в RabbitMQ.

```go
// Order Service - Producer
msg := OrderCreatedEvent{
    OrderID: 123,
    UserID:  456,
    Amount:  100.50,
}

data, _ := json.Marshal(msg)
channel.Publish(
    "orders",      // exchange
    "order.created", // routing key
    false, false,
    amqp.Publishing{
        ContentType: "application/json",
        Body:        data,
    },
)
```

### Queue (Очередь)

Хранит сообщения до тех пор, пока consumer их не обработает.

```
Producer → Exchange → Queue → Consumer
```

### Consumer (Потребитель)

Получает и обрабатывает сообщения из очереди.

```go
// Email Service - Consumer
msgs, _ := channel.Consume(
    "email-queue", // queue
    "",            // consumer
    true,          // auto-ack
    false, false, false, nil,
)

for msg := range msgs {
    var event OrderCreatedEvent
    json.Unmarshal(msg.Body, &event)

    // Обрабатываем событие
    sendEmail(event.UserID, "Order created!")
}
```

### Exchange (Обменник)

Маршрутизирует сообщения в очереди.

**Типы Exchange:**

1. **Direct** - по routing key
2. **Fanout** - во все привязанные очереди
3. **Topic** - по паттерну routing key
4. **Headers** - по headers

## Установка RabbitMQ

### Docker

```bash
# Запуск RabbitMQ с management UI
docker run -d --name rabbitmq \
  -p 5672:5672 \
  -p 15672:15672 \
  rabbitmq:3-management

# Management UI доступен на http://localhost:15672
# Login: guest / guest
```

### Go библиотека

```bash
go get github.com/rabbitmq/amqp091-go
```

## Producer (отправка сообщений)

```go
package main

import (
    "encoding/json"
    "log"
    "time"

    amqp "github.com/rabbitmq/amqp091-go"
)

type OrderCreatedEvent struct {
    OrderID   int       `json:"order_id"`
    UserID    int       `json:"user_id"`
    Amount    float64   `json:"amount"`
    CreatedAt time.Time `json:"created_at"`
}

func main() {
    // 1. Подключение к RabbitMQ
    conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
    if err != nil {
        log.Fatal(err)
    }
    defer conn.Close()

    // 2. Создаем канал
    ch, err := conn.Channel()
    if err != nil {
        log.Fatal(err)
    }
    defer ch.Close()

    // 3. Объявляем exchange
    err = ch.ExchangeDeclare(
        "orders",  // name
        "topic",   // type
        true,      // durable (сохраняется при перезапуске)
        false,     // auto-deleted
        false,     // internal
        false,     // no-wait
        nil,       // arguments
    )
    if err != nil {
        log.Fatal(err)
    }

    // 4. Отправляем сообщение
    event := OrderCreatedEvent{
        OrderID:   123,
        UserID:    456,
        Amount:    100.50,
        CreatedAt: time.Now(),
    }

    body, _ := json.Marshal(event)

    err = ch.Publish(
        "orders",        // exchange
        "order.created", // routing key
        false,           // mandatory
        false,           // immediate
        amqp.Publishing{
            DeliveryMode: amqp.Persistent, // Persistent message
            ContentType:  "application/json",
            Body:         body,
        },
    )
    if err != nil {
        log.Fatal(err)
    }

    log.Printf("Published: %+v", event)
}
```

## Consumer (получение сообщений)

```go
package main

import (
    "encoding/json"
    "log"
    "time"

    amqp "github.com/rabbitmq/amqp091-go"
)

type OrderCreatedEvent struct {
    OrderID   int       `json:"order_id"`
    UserID    int       `json:"user_id"`
    Amount    float64   `json:"amount"`
    CreatedAt time.Time `json:"created_at"`
}

func main() {
    // 1. Подключение
    conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
    if err != nil {
        log.Fatal(err)
    }
    defer conn.Close()

    // 2. Создаем канал
    ch, err := conn.Channel()
    if err != nil {
        log.Fatal(err)
    }
    defer ch.Close()

    // 3. Объявляем exchange
    err = ch.ExchangeDeclare("orders", "topic", true, false, false, false, nil)
    if err != nil {
        log.Fatal(err)
    }

    // 4. Объявляем очередь
    queue, err := ch.QueueDeclare(
        "email-queue", // name
        true,          // durable
        false,         // delete when unused
        false,         // exclusive
        false,         // no-wait
        nil,           // arguments
    )
    if err != nil {
        log.Fatal(err)
    }

    // 5. Привязываем очередь к exchange
    err = ch.QueueBind(
        queue.Name,      // queue name
        "order.created", // routing key
        "orders",        // exchange
        false,
        nil,
    )
    if err != nil {
        log.Fatal(err)
    }

    // 6. Устанавливаем QoS (prefetch)
    err = ch.Qos(
        1,     // prefetch count - обрабатываем по 1 сообщению
        0,     // prefetch size
        false, // global
    )
    if err != nil {
        log.Fatal(err)
    }

    // 7. Начинаем потреблять сообщения
    msgs, err := ch.Consume(
        queue.Name, // queue
        "",         // consumer
        false,      // auto-ack (manual ack!)
        false,      // exclusive
        false,      // no-local
        false,      // no-wait
        nil,        // args
    )
    if err != nil {
        log.Fatal(err)
    }

    forever := make(chan bool)

    go func() {
        for msg := range msgs {
            var event OrderCreatedEvent
            if err := json.Unmarshal(msg.Body, &event); err != nil {
                log.Printf("Error unmarshaling: %v", err)
                msg.Nack(false, false) // Отклоняем сообщение
                continue
            }

            log.Printf("Received: %+v", event)

            // Обрабатываем событие
            err := sendEmail(event.UserID, event.OrderID)
            if err != nil {
                log.Printf("Error sending email: %v", err)
                msg.Nack(false, true) // Requeue message
                continue
            }

            // Подтверждаем обработку
            msg.Ack(false)
        }
    }()

    log.Println("Waiting for messages...")
    <-forever
}

func sendEmail(userID int, orderID int) error {
    // Симуляция отправки email
    log.Printf("Sending email to user %d about order %d", userID, orderID)
    time.Sleep(500 * time.Millisecond)
    return nil
}
```

## Exchange Types

### 1. Direct Exchange

Маршрутизация по точному совпадению routing key.

```
Producer → (routing_key="order.created") → Direct Exchange
                                                ↓
                          [routing_key="order.created"]
                                                ↓
                                            Queue 1

                          [routing_key="order.paid"]
                                                ↓
                                            Queue 2
```

```go
// Объявление
ch.ExchangeDeclare("orders", "direct", true, false, false, false, nil)

// Публикация
ch.Publish("orders", "order.created", false, false, amqp.Publishing{...})

// Binding
ch.QueueBind("email-queue", "order.created", "orders", false, nil)
```

### 2. Fanout Exchange

Маршрутизация во ВСЕ привязанные очереди (игнорирует routing key).

```
Producer → Fanout Exchange → Queue 1 (Email)
                          ↘ Queue 2 (SMS)
                          ↘ Queue 3 (Push)
```

```go
// Объявление
ch.ExchangeDeclare("notifications", "fanout", true, false, false, false, nil)

// Публикация (routing key игнорируется)
ch.Publish("notifications", "", false, false, amqp.Publishing{...})

// Binding
ch.QueueBind("email-queue", "", "notifications", false, nil)
ch.QueueBind("sms-queue", "", "notifications", false, nil)
ch.QueueBind("push-queue", "", "notifications", false, nil)
```

**Использование:** Рассылка одного события множеству потребителей.

### 3. Topic Exchange

Маршрутизация по паттерну routing key.

**Wildcards:**
- `*` - одно слово
- `#` - ноль или больше слов

```
Producer → Topic Exchange → Queue 1 (binding="order.*")
                                      ↑
                          Matches: order.created, order.paid
                                      ↓
                          Queue 2 (binding="*.created")
                                      ↑
                          Matches: order.created, user.created
```

```go
// Объявление
ch.ExchangeDeclare("events", "topic", true, false, false, false, nil)

// Публикация
ch.Publish("events", "order.created", false, false, amqp.Publishing{...})
ch.Publish("events", "user.created", false, false, amqp.Publishing{...})
ch.Publish("events", "order.paid", false, false, amqp.Publishing{...})

// Binding
ch.QueueBind("order-events", "order.*", "events", false, nil)       // order.created, order.paid
ch.QueueBind("all-created", "*.created", "events", false, nil)      // order.created, user.created
ch.QueueBind("all-events", "#", "events", false, nil)               // все события
```

### 4. Headers Exchange

Маршрутизация по headers (не routing key).

```go
// Объявление
ch.ExchangeDeclare("headers-exchange", "headers", true, false, false, false, nil)

// Публикация с headers
ch.Publish("headers-exchange", "", false, false, amqp.Publishing{
    Headers: amqp.Table{
        "format": "json",
        "type":   "order",
    },
    Body: body,
})

// Binding
ch.QueueBind("json-orders", "", "headers-exchange", false, amqp.Table{
    "x-match": "all",   // Все headers должны совпадать
    "format":  "json",
    "type":    "order",
})
```

## Message Acknowledgement (Подтверждение)

### Auto-Ack (автоматическое)

```go
msgs, _ := ch.Consume(
    "email-queue",
    "",
    true,  // auto-ack = true
    false, false, false, nil,
)

for msg := range msgs {
    // Сообщение автоматически удаляется из очереди
    // Даже если обработка упадет с ошибкой!
    processMessage(msg)
}
```

**Проблема:** Если consumer упадет во время обработки → сообщение потеряно.

### Manual Ack (ручное подтверждение)

```go
msgs, _ := ch.Consume(
    "email-queue",
    "",
    false, // auto-ack = false (manual!)
    false, false, false, nil,
)

for msg := range msgs {
    err := processMessage(msg)

    if err != nil {
        // Ошибка - возвращаем в очередь
        msg.Nack(false, true) // requeue = true
    } else {
        // Успех - подтверждаем
        msg.Ack(false)
    }
}
```

### Reject (отклонение)

```go
// Reject - отклонить одно сообщение
msg.Reject(true) // requeue = true

// Nack - отклонить одно или несколько сообщений
msg.Nack(
    false, // multiple = false (одно сообщение)
    true,  // requeue = true
)
```

## Prefetch (QoS)

Ограничивает количество неподтвержденных сообщений у consumer.

```go
// Обрабатываем по 1 сообщению за раз
ch.Qos(1, 0, false)

// Обрабатываем по 10 сообщений за раз
ch.Qos(10, 0, false)
```

**Без QoS:**

```
RabbitMQ → Consumer 1 (100 сообщений)
        → Consumer 2 (0 сообщений)

Consumer 1 перегружен!
```

**С QoS:**

```
ch.Qos(1, 0, false)

RabbitMQ → Consumer 1 (1 сообщение)
        → Consumer 2 (1 сообщение)
        → Consumer 1 (следующее после Ack)

Равномерное распределение!
```

## Dead Letter Exchange (DLX)

Отправляет неудачные сообщения в отдельную очередь.

```go
// Создаем DLX
ch.ExchangeDeclare("dlx", "direct", true, false, false, false, nil)

// Создаем DLQ (Dead Letter Queue)
ch.QueueDeclare("dead-letters", true, false, false, false, nil)
ch.QueueBind("dead-letters", "", "dlx", false, nil)

// Основная очередь с DLX
ch.QueueDeclare("email-queue", true, false, false, false, amqp.Table{
    "x-dead-letter-exchange": "dlx",  // DLX
})

// Сообщения попадут в DLQ если:
// - TTL истек
// - Превышен лимит очереди
// - Rejected без requeue
msg.Reject(false) // requeue = false → в DLQ
```

## TTL (Time To Live)

### TTL для очереди

```go
// Все сообщения в очереди удаляются через 60 секунд
ch.QueueDeclare("email-queue", true, false, false, false, amqp.Table{
    "x-message-ttl": 60000, // 60 секунд (в миллисекундах)
})
```

### TTL для сообщения

```go
// Конкретное сообщение удаляется через 30 секунд
ch.Publish("orders", "order.created", false, false, amqp.Publishing{
    Expiration: "30000", // 30 секунд (строка!)
    Body:       body,
})
```

## Priority Queue (приоритеты)

```go
// Очередь с приоритетами (0-10)
ch.QueueDeclare("priority-queue", true, false, false, false, amqp.Table{
    "x-max-priority": 10,
})

// Отправляем с приоритетом
ch.Publish("", "priority-queue", false, false, amqp.Publishing{
    Priority: 5, // Приоритет 0-10
    Body:     []byte("Normal priority message"),
})

ch.Publish("", "priority-queue", false, false, amqp.Publishing{
    Priority: 10, // Высокий приоритет
    Body:     []byte("High priority message"),
})
```

## Retry Mechanism

```go
func processWithRetry(msg amqp.Delivery, maxRetries int) {
    retries := 0
    if msg.Headers != nil {
        if r, ok := msg.Headers["x-retry-count"].(int32); ok {
            retries = int(r)
        }
    }

    err := processMessage(msg)

    if err != nil {
        if retries < maxRetries {
            // Retry
            log.Printf("Retry %d/%d", retries+1, maxRetries)

            headers := amqp.Table{"x-retry-count": int32(retries + 1)}
            ch.Publish("", "email-queue", false, false, amqp.Publishing{
                Headers: headers,
                Body:    msg.Body,
            })

            msg.Ack(false) // Удаляем оригинальное сообщение
        } else {
            // Max retries exceeded → DLQ
            log.Printf("Max retries exceeded, sending to DLQ")
            msg.Reject(false) // Отправляем в DLX
        }
    } else {
        msg.Ack(false)
    }
}
```

## RPC (Request-Reply)

```go
// Server
func rpcServer() {
    msgs, _ := ch.Consume("rpc_queue", "", false, false, false, false, nil)

    for msg := range msgs {
        // Обрабатываем запрос
        result := processRequest(msg.Body)

        // Отправляем ответ
        ch.Publish(
            "",             // exchange
            msg.ReplyTo,    // queue (из запроса)
            false, false,
            amqp.Publishing{
                CorrelationId: msg.CorrelationId, // ID из запроса
                Body:          result,
            },
        )

        msg.Ack(false)
    }
}

// Client
func rpcClient() {
    // Создаем временную очередь для ответов
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
            log.Printf("Response: %s", msg.Body)
            return
        }
    }
}
```

## Monitoring

### Management HTTP API

```bash
# Список очередей
curl -u guest:guest http://localhost:15672/api/queues

# Статистика очереди
curl -u guest:guest http://localhost:15672/api/queues/%2F/email-queue

# Список connections
curl -u guest:guest http://localhost:15672/api/connections
```

### Health Check

```go
func healthCheck() error {
    conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
    if err != nil {
        return err
    }
    defer conn.Close()

    ch, err := conn.Channel()
    if err != nil {
        return err
    }
    defer ch.Close()

    return nil
}
```

## Best Practices

1. ✅ **Manual Ack** - используйте ручное подтверждение
2. ✅ **Prefetch (QoS)** - ограничивайте неподтвержденные сообщения
3. ✅ **Durable queues** - сохраняются при перезапуске
4. ✅ **Persistent messages** - сообщения на диске
5. ✅ **Dead Letter Queue** - для failed сообщений
6. ✅ **Idempotency** - обработка может повторяться
7. ✅ **Retry mechanism** - с ограничением попыток
8. ✅ **Connection pooling** - переиспользуйте соединения
9. ❌ **Не блокируйте consumer** - обрабатывайте быстро
10. ❌ **Не теряйте сообщения** - используйте persistent + manual ack

## Связанные темы

- [[Apache Kafka]]
- [[Message Queue паттерны]]
- [[Микросервисная архитектура]]
- [[Монолит vs Микросервисы]]
- [[gRPC]]
- [[REST API - Основы]]
