# Apache Kafka

Apache Kafka — распределенная платформа для потоковой обработки данных. Event streaming система для построения real-time data pipelines.

## Kafka vs RabbitMQ

| Критерий | Kafka | RabbitMQ |
|----------|-------|----------|
| **Модель** | Event log (append-only) | Message queue |
| **Хранение** | На диске (долго) | В памяти/диске (временно) |
| **Производительность** | ⚡ Очень высокая (> 1M msg/sec) | Высокая (< 100k msg/sec) |
| **Порядок** | ✅ Гарантирован (в partition) | ❌ Не гарантирован |
| **Replay** | ✅ Можно читать старые события | ❌ Нельзя (удаляются после ack) |
| **Consumer groups** | ✅ Да | ✅ Да |
| **Routing** | По partition key | По routing key (гибче) |
| **Протокол** | TCP (кастомный) | AMQP |
| **Use case** | Event streaming, логи, метрики | Task queues, RPC |

**Когда использовать Kafka:**
- Event sourcing
- Логи, метрики (большие объемы)
- Real-time analytics
- Data pipelines
- Replay событий

**Когда использовать RabbitMQ:**
- Task queues
- RPC
- Сложная маршрутизация
- Малые объемы данных

## Основные концепции

### Topic

Категория, в которую публикуются сообщения.

```
orders - topic для заказов
users  - topic для пользователей
logs   - topic для логов
```

### Partition

Topic разделен на partitions для параллелизма.

```
Topic: orders
├── Partition 0: [msg1, msg4, msg7]
├── Partition 1: [msg2, msg5, msg8]
└── Partition 2: [msg3, msg6, msg9]
```

**Порядок гарантирован только внутри partition!**

### Offset

Уникальный ID сообщения в partition.

```
Partition 0:
Offset: 0    1    2    3    4
Msg:   [A] → [B] → [C] → [D] → [E]
```

### Producer

Отправляет сообщения в topic.

```go
producer.Send("orders", key, value)
```

### Consumer

Читает сообщения из topic.

```go
consumer.Subscribe("orders")
for message := range consumer.Messages() {
    process(message)
}
```

### Consumer Group

Группа consumers, которые распределяют partition между собой.

```
Topic: orders (3 partitions)

Consumer Group "email-service":
├── Consumer 1 → Partition 0
├── Consumer 2 → Partition 1
└── Consumer 3 → Partition 2
```

## Установка Kafka

### Docker Compose

```yaml
version: '3'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000

  kafka:
    image: confluentinc/cp-kafka:latest
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
```

```bash
docker-compose up -d
```

### Go библиотека

```bash
# Sarama - популярная Go библиотека
go get github.com/IBM/sarama
```

## Producer (отправка сообщений)

### Простой Producer

```go
package main

import (
    "encoding/json"
    "log"
    "time"

    "github.com/IBM/sarama"
)

type OrderCreatedEvent struct {
    OrderID   int       `json:"order_id"`
    UserID    int       `json:"user_id"`
    Amount    float64   `json:"amount"`
    CreatedAt time.Time `json:"created_at"`
}

func main() {
    // 1. Конфигурация
    config := sarama.NewConfig()
    config.Producer.Return.Successes = true
    config.Producer.RequiredAcks = sarama.WaitForAll // Ждем подтверждения от всех replicas
    config.Producer.Retry.Max = 3

    // 2. Создаем producer
    producer, err := sarama.NewSyncProducer([]string{"localhost:9092"}, config)
    if err != nil {
        log.Fatal(err)
    }
    defer producer.Close()

    // 3. Формируем сообщение
    event := OrderCreatedEvent{
        OrderID:   123,
        UserID:    456,
        Amount:    100.50,
        CreatedAt: time.Now(),
    }

    value, _ := json.Marshal(event)

    msg := &sarama.ProducerMessage{
        Topic: "orders",
        Key:   sarama.StringEncoder(fmt.Sprintf("%d", event.OrderID)), // Partition key
        Value: sarama.ByteEncoder(value),
    }

    // 4. Отправляем
    partition, offset, err := producer.SendMessage(msg)
    if err != nil {
        log.Fatal(err)
    }

    log.Printf("Message sent to partition %d at offset %d", partition, offset)
}
```

### Async Producer (производительнее)

```go
func asyncProducer() {
    config := sarama.NewConfig()
    config.Producer.Return.Successes = true

    producer, _ := sarama.NewAsyncProducer([]string{"localhost:9092"}, config)
    defer producer.Close()

    // Горутина для обработки успехов
    go func() {
        for success := range producer.Successes() {
            log.Printf("Message sent: partition=%d, offset=%d", success.Partition, success.Offset)
        }
    }()

    // Горутина для обработки ошибок
    go func() {
        for err := range producer.Errors() {
            log.Printf("Error: %v", err)
        }
    }()

    // Отправляем сообщения
    for i := 0; i < 100; i++ {
        event := OrderCreatedEvent{
            OrderID: i,
            UserID:  456,
            Amount:  100.50,
        }

        value, _ := json.Marshal(event)

        producer.Input() <- &sarama.ProducerMessage{
            Topic: "orders",
            Key:   sarama.StringEncoder(fmt.Sprintf("%d", i)),
            Value: sarama.ByteEncoder(value),
        }
    }
}
```

## Consumer (получение сообщений)

### Простой Consumer

```go
package main

import (
    "context"
    "encoding/json"
    "log"
    "os"
    "os/signal"

    "github.com/IBM/sarama"
)

type OrderCreatedEvent struct {
    OrderID int     `json:"order_id"`
    UserID  int     `json:"user_id"`
    Amount  float64 `json:"amount"`
}

func main() {
    // 1. Конфигурация
    config := sarama.NewConfig()
    config.Consumer.Return.Errors = true
    config.Consumer.Offsets.Initial = sarama.OffsetNewest // Читаем новые сообщения
    // config.Consumer.Offsets.Initial = sarama.OffsetOldest // Читаем с начала

    // 2. Создаем consumer
    consumer, err := sarama.NewConsumer([]string{"localhost:9092"}, config)
    if err != nil {
        log.Fatal(err)
    }
    defer consumer.Close()

    // 3. Подписываемся на partition
    partitionConsumer, err := consumer.ConsumePartition("orders", 0, sarama.OffsetNewest)
    if err != nil {
        log.Fatal(err)
    }
    defer partitionConsumer.Close()

    // 4. Обрабатываем сообщения
    signals := make(chan os.Signal, 1)
    signal.Notify(signals, os.Interrupt)

    for {
        select {
        case msg := <-partitionConsumer.Messages():
            var event OrderCreatedEvent
            if err := json.Unmarshal(msg.Value, &event); err != nil {
                log.Printf("Error unmarshaling: %v", err)
                continue
            }

            log.Printf("Received: partition=%d, offset=%d, event=%+v",
                msg.Partition, msg.Offset, event)

            // Обрабатываем событие
            processEvent(event)

        case err := <-partitionConsumer.Errors():
            log.Printf("Error: %v", err)

        case <-signals:
            log.Println("Shutting down...")
            return
        }
    }
}

func processEvent(event OrderCreatedEvent) {
    log.Printf("Processing order %d", event.OrderID)
}
```

### Consumer Group (рекомендуется)

```go
package main

import (
    "context"
    "encoding/json"
    "log"
    "os"
    "os/signal"

    "github.com/IBM/sarama"
)

type Consumer struct {
    ready chan bool
}

func (c *Consumer) Setup(sarama.ConsumerGroupSession) error {
    close(c.ready)
    return nil
}

func (c *Consumer) Cleanup(sarama.ConsumerGroupSession) error {
    return nil
}

func (c *Consumer) ConsumeClaim(session sarama.ConsumerGroupSession, claim sarama.ConsumerGroupClaim) error {
    for message := range claim.Messages() {
        var event OrderCreatedEvent
        if err := json.Unmarshal(message.Value, &event); err != nil {
            log.Printf("Error unmarshaling: %v", err)
            continue
        }

        log.Printf("Received: topic=%s, partition=%d, offset=%d, event=%+v",
            message.Topic, message.Partition, message.Offset, event)

        // Обрабатываем событие
        processEvent(event)

        // Отмечаем сообщение как обработанное
        session.MarkMessage(message, "")
    }

    return nil
}

func main() {
    config := sarama.NewConfig()
    config.Consumer.Group.Rebalance.Strategy = sarama.NewBalanceStrategyRoundRobin()
    config.Consumer.Offsets.Initial = sarama.OffsetOldest

    // Создаем consumer group
    group, err := sarama.NewConsumerGroup(
        []string{"localhost:9092"},
        "email-service",  // Group ID
        config,
    )
    if err != nil {
        log.Fatal(err)
    }
    defer group.Close()

    consumer := &Consumer{
        ready: make(chan bool),
    }

    ctx, cancel := context.WithCancel(context.Background())

    // Горутина для потребления
    go func() {
        for {
            if err := group.Consume(ctx, []string{"orders"}, consumer); err != nil {
                log.Printf("Error: %v", err)
            }

            if ctx.Err() != nil {
                return
            }
        }
    }()

    <-consumer.ready
    log.Println("Consumer started")

    // Graceful shutdown
    signals := make(chan os.Signal, 1)
    signal.Notify(signals, os.Interrupt)

    <-signals
    log.Println("Shutting down...")
    cancel()
}
```

## Partitioning

### Выбор partition

Kafka определяет partition по:

1. **По ключу (key):**
   ```go
   msg := &sarama.ProducerMessage{
       Topic: "orders",
       Key:   sarama.StringEncoder("user123"), // Hash(key) % num_partitions
       Value: value,
   }
   ```
   - Все сообщения с одним ключом → в один partition
   - Гарантирует порядок для одного ключа

2. **Round-robin (без ключа):**
   ```go
   msg := &sarama.ProducerMessage{
       Topic: "orders",
       Key:   nil, // Round-robin
       Value: value,
   }
   ```

3. **Явное указание:**
   ```go
   msg := &sarama.ProducerMessage{
       Topic:     "orders",
       Partition: 2, // Конкретный partition
       Value:     value,
   }
   ```

### Когда использовать key

```go
// Все заказы одного пользователя → в один partition → порядок гарантирован
msg := &sarama.ProducerMessage{
    Topic: "orders",
    Key:   sarama.StringEncoder(fmt.Sprintf("user-%d", userID)),
    Value: value,
}
```

## Offset Management

### Auto Commit (по умолчанию)

```go
config.Consumer.Offsets.AutoCommit.Enable = true
config.Consumer.Offsets.AutoCommit.Interval = 1 * time.Second

// Offset автоматически commit'ится каждую секунду
```

**Проблема:** Если consumer упадет после получения, но до обработки → сообщение потеряно.

### Manual Commit (рекомендуется)

```go
config.Consumer.Offsets.AutoCommit.Enable = false

func (c *Consumer) ConsumeClaim(session sarama.ConsumerGroupSession, claim sarama.ConsumerGroupClaim) error {
    for message := range claim.Messages() {
        processMessage(message)

        // Commit offset ПОСЛЕ обработки
        session.MarkMessage(message, "")
        session.Commit() // Явный commit
    }
    return nil
}
```

### Commit по батчам

```go
batch := make([]*sarama.ConsumerMessage, 0, 100)

for message := range claim.Messages() {
    batch = append(batch, message)

    if len(batch) >= 100 {
        // Обрабатываем батч
        processBatch(batch)

        // Commit последнего offset
        session.MarkMessage(batch[len(batch)-1], "")
        session.Commit()

        batch = batch[:0] // Очищаем батч
    }
}
```

## Idempotent Producer

Гарантирует exactly-once доставку (нет дубликатов).

```go
config := sarama.NewConfig()
config.Producer.Idempotent = true
config.Producer.RequiredAcks = sarama.WaitForAll
config.Producer.Return.Errors = true
config.Net.MaxOpenRequests = 1

producer, _ := sarama.NewSyncProducer([]string{"localhost:9092"}, config)
```

**Как работает:**
- Kafka присваивает каждому producer уникальный ID
- Каждое сообщение имеет sequence number
- Kafka дедуплицирует сообщения с одинаковым ID + sequence

## Transactions

```go
config := sarama.NewConfig()
config.Producer.Idempotent = true
config.Producer.Transaction.ID = "my-transaction-id"
config.Producer.RequiredAcks = sarama.WaitForAll

producer, _ := sarama.NewSyncProducer([]string{"localhost:9092"}, config)
defer producer.Close()

// Начинаем транзакцию
producer.BeginTxn()

// Отправляем сообщения
producer.SendMessage(&sarama.ProducerMessage{Topic: "orders", Value: value1})
producer.SendMessage(&sarama.ProducerMessage{Topic: "payments", Value: value2})

// Коммитим транзакцию (атомарно оба сообщения или ни одного)
err := producer.CommitTxn()
if err != nil {
    producer.AbortTxn()
}
```

## Retention Policy

Kafka хранит сообщения определенное время.

```bash
# retention.ms - время хранения (по умолчанию 7 дней)
kafka-topics.sh --alter --topic orders \
  --config retention.ms=604800000  # 7 дней

# retention.bytes - максимальный размер
kafka-topics.sh --alter --topic orders \
  --config retention.bytes=1073741824  # 1 GB

# Compaction - хранить только последнее значение по ключу
kafka-topics.sh --alter --topic users \
  --config cleanup.policy=compact
```

### Log Compaction

Полезно для снапшотов состояния (event sourcing).

```
До compaction:
Key: user1 → [created] → [updated] → [updated] → [updated]

После compaction:
Key: user1 → [updated] (последнее состояние)
```

## Replay Events

Kafka позволяет читать старые события.

```go
// Читаем с начала topic
config.Consumer.Offsets.Initial = sarama.OffsetOldest

// Или с конкретного offset
partitionConsumer.ConsumePartition("orders", 0, 100) // С offset 100
```

**Use case:**
- Восстановление после сбоя
- Rebuilding state
- Debugging

## Monitoring

### Consumer Lag

```go
import "github.com/IBM/sarama"

func checkLag() {
    client, _ := sarama.NewClient([]string{"localhost:9092"}, sarama.NewConfig())
    defer client.Close()

    // Последние offsets в topic
    partitions, _ := client.Partitions("orders")
    for _, partition := range partitions {
        newest, _ := client.GetOffset("orders", partition, sarama.OffsetNewest)
        log.Printf("Partition %d: newest offset = %d", partition, newest)
    }

    // Consumer group offsets
    coordinator, _ := client.Coordinator("email-service")
    request := &sarama.OffsetFetchRequest{
        ConsumerGroup: "email-service",
        Version:       1,
    }
    response, _ := coordinator.FetchOffset(request)
    log.Printf("Consumer offsets: %+v", response)
}
```

### Prometheus Metrics

```go
import "github.com/prometheus/client_golang/prometheus"

var (
    messagesProcessed = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "kafka_messages_processed_total",
        },
        []string{"topic", "partition"},
    )

    processingDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name: "kafka_message_processing_duration_seconds",
        },
        []string{"topic"},
    )
)

func processMessage(msg *sarama.ConsumerMessage) {
    start := time.Now()

    // Обработка
    // ...

    messagesProcessed.WithLabelValues(msg.Topic, fmt.Sprintf("%d", msg.Partition)).Inc()
    processingDuration.WithLabelValues(msg.Topic).Observe(time.Since(start).Seconds())
}
```

## Use Cases

### 1. Event Sourcing

```go
// Сохраняем все события
type OrderEvent struct {
    Type      string // "created", "paid", "shipped"
    OrderID   int
    Data      json.RawMessage
    Timestamp time.Time
}

// Producer
producer.Send("order-events", orderID, OrderCreatedEvent{...})
producer.Send("order-events", orderID, OrderPaidEvent{...})
producer.Send("order-events", orderID, OrderShippedEvent{...})

// Consumer восстанавливает состояние
events := readAllEvents(orderID)
order := replayEvents(events)
```

### 2. Логи и Метрики

```go
// Логирование в Kafka
type LogEntry struct {
    Level     string
    Service   string
    Message   string
    Timestamp time.Time
}

producer.Send("logs", serviceID, logEntry)

// Consumer → Elasticsearch
for msg := range consumer.Messages() {
    elasticsearch.Index("logs", msg.Value)
}
```

### 3. Real-Time Analytics

```go
// Стримминг метрик
type PageViewEvent struct {
    UserID    string
    Page      string
    Timestamp time.Time
}

// Kafka Streams (или Flink) агрегирует в реальном времени
// Подсчет просмотров за последние 5 минут
```

### 4. CDC (Change Data Capture)

```go
// Debezium отправляет изменения БД в Kafka
// PostgreSQL → Kafka → Microservices

type DBChange struct {
    Operation string // INSERT, UPDATE, DELETE
    Table     string
    Before    json.RawMessage
    After     json.RawMessage
}
```

## Best Practices

1. ✅ **Используйте Consumer Groups** - для масштабирования
2. ✅ **Partition key** - для порядка сообщений одного entity
3. ✅ **Manual offset commit** - после обработки
4. ✅ **Idempotent processing** - обработка может повторяться
5. ✅ **Batch processing** - для производительности
6. ✅ **Monitor consumer lag** - метрика отставания
7. ✅ **Graceful shutdown** - commit offsets перед выходом
8. ✅ **Достаточное количество partitions** - для параллелизма
9. ❌ **Не создавайте слишком много partitions** - overhead
10. ❌ **Не храните большие сообщения** (> 1MB) - используйте S3 + reference

## Kafka Streams (stream processing)

```go
// Обработка потока в реальном времени
// Подсчет заказов по статусу за последние 5 минут

import "github.com/lovoo/goka"

func main() {
    group := goka.DefineGroup("order-stats",
        goka.Input("orders", new(codec.String), func(ctx goka.Context, msg interface{}) {
            var order Order
            json.Unmarshal([]byte(msg.(string)), &order)

            // Агрегация
            count := ctx.Value().(int)
            count++
            ctx.SetValue(count)
        }),
        goka.Persist(new(codec.Int64)),
    )

    processor, _ := goka.NewProcessor([]string{"localhost:9092"}, group)
    processor.Run(context.Background())
}
```

## Связанные темы

- [[RabbitMQ]]
- [[Message Queue паттерны]]
- [[Микросервисная архитектура]]
- [[Монолит vs Микросервисы]]
- [[System Design - Основы]]
