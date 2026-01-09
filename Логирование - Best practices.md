# Логирование - Best practices

Логирование - это процесс записи событий, происходящих в приложении. Правильное логирование критически важно для отладки, мониторинга и аудита систем.

## Почему логирование важно

- **Отладка production issues** - понять, что пошло не так
- **Мониторинг** - обнаружить проблемы до пользователей
- **Аудит** - кто, когда и что сделал
- **Performance анализ** - найти медленные запросы
- **Security** - обнаружить атаки и подозрительную активность

## Structured Logging

Structured logging - это логирование в структурированном формате (обычно JSON), а не plain text.

### Plain Text vs Structured

```go
// ❌ Plain text - сложно парсить
log.Println("User john logged in from IP 192.168.1.1")

// ✅ Structured JSON - легко парсить и фильтровать
logger.Info("user logged in",
    "user_id", "john",
    "ip", "192.168.1.1",
    "timestamp", time.Now(),
)
```

**Вывод:**
```json
{
  "level": "info",
  "msg": "user logged in",
  "user_id": "john",
  "ip": "192.168.1.1",
  "timestamp": "2024-01-15T10:30:45Z"
}
```

**Преимущества structured logging:**
- Легко парсить (Elasticsearch, Loki, Splunk)
- Можно фильтровать по полям: `user_id="john"`
- Автоматическая индексация
- Машиночитаемый формат

## Log Levels

Log levels определяют важность сообщения.

### Стандартные уровни

```
TRACE → DEBUG → INFO → WARN → ERROR → FATAL
```

#### 1. TRACE / DEBUG

Детальная информация для отладки.

```go
logger.Debug("processing request",
    "request_id", requestID,
    "query_params", params,
)
```

**Когда использовать:**
- Входные параметры функций
- Промежуточные значения
- Детали алгоритма
- В production обычно выключен

#### 2. INFO

Информационные сообщения о нормальной работе.

```go
logger.Info("server started",
    "port", 8080,
    "environment", "production",
)

logger.Info("user logged in",
    "user_id", userID,
    "ip", clientIP,
)
```

**Когда использовать:**
- Старт/остановка сервиса
- Успешные операции
- Важные бизнес-события

#### 3. WARN

Потенциальные проблемы, но приложение продолжает работать.

```go
logger.Warn("slow database query",
    "query_time", elapsed,
    "query", query,
)

logger.Warn("deprecated API endpoint used",
    "endpoint", "/api/v1/old",
    "user_id", userID,
)
```

**Когда использовать:**
- Медленные запросы
- Использование deprecated API
- Неоптимальные параметры
- Приближение к лимитам

#### 4. ERROR

Ошибки, требующие внимания.

```go
logger.Error("failed to save user",
    "error", err,
    "user_id", userID,
)

logger.Error("database connection failed",
    "error", err,
    "retry_count", retryCount,
)
```

**Когда использовать:**
- Ошибки при обработке запросов
- Сбои при обращении к внешним системам
- Unexpected ошибки

#### 5. FATAL / PANIC

Критическая ошибка, приложение не может продолжать работу.

```go
logger.Fatal("failed to initialize database",
    "error", err,
)
// После Fatal обычно следует os.Exit(1)
```

**Когда использовать:**
- Невозможно подключиться к БД при старте
- Критические конфигурационные ошибки
- Неустранимые ошибки инициализации

## Популярные библиотеки для Go

### 1. zap (Uber)

Высокопроизводительный structured logger.

```go
package main

import (
    "go.uber.org/zap"
    "time"
)

func main() {
    // Production logger (JSON)
    logger, _ := zap.NewProduction()
    defer logger.Sync()

    logger.Info("user logged in",
        zap.String("user_id", "john"),
        zap.String("ip", "192.168.1.1"),
        zap.Time("timestamp", time.Now()),
    )

    // Development logger (human-readable)
    // logger, _ := zap.NewDevelopment()

    // С ошибкой
    err := doSomething()
    if err != nil {
        logger.Error("operation failed",
            zap.Error(err),
            zap.String("operation", "do_something"),
        )
    }
}

// Вывод (Production):
// {"level":"info","ts":1642424242.123,"msg":"user logged in","user_id":"john","ip":"192.168.1.1"}
```

**Преимущества zap:**
- Очень быстрый (zero-allocation)
- Structured logging
- Type-safe
- Популярен в крупных компаниях

### 2. zerolog

Минималистичный и быстрый logger с fluent API.

```go
package main

import (
    "github.com/rs/zerolog"
    "github.com/rs/zerolog/log"
    "os"
)

func main() {
    // Настройка формата
    zerolog.TimeFieldFormat = zerolog.TimeFormatUnix

    // Pretty logging для development
    // log.Logger = log.Output(zerolog.ConsoleWriter{Out: os.Stderr})

    log.Info().
        Str("user_id", "john").
        Str("ip", "192.168.1.1").
        Msg("user logged in")

    // С ошибкой
    err := doSomething()
    if err != nil {
        log.Error().
            Err(err).
            Str("operation", "do_something").
            Msg("operation failed")
    }

    // Вложенные структуры
    log.Info().
        Dict("user", zerolog.Dict().
            Str("id", "john").
            Int("age", 30),
        ).
        Msg("user details")
}

// Вывод:
// {"level":"info","user_id":"john","ip":"192.168.1.1","message":"user logged in"}
```

**Преимущества zerolog:**
- Zero allocation
- Fluent API
- Чистый синтаксис
- Легковесный

### 3. logrus

Старый популярный logger с hook'ами.

```go
package main

import (
    "github.com/sirupsen/logrus"
)

func main() {
    // Настройка формата
    logrus.SetFormatter(&logrus.JSONFormatter{})
    logrus.SetLevel(logrus.InfoLevel)

    logrus.WithFields(logrus.Fields{
        "user_id": "john",
        "ip":      "192.168.1.1",
    }).Info("user logged in")

    // С ошибкой
    err := doSomething()
    if err != nil {
        logrus.WithError(err).Error("operation failed")
    }
}
```

**Недостатки logrus:**
- Медленнее чем zap/zerolog
- Не type-safe
- Maintainer объявил библиотеку в maintenance mode

**Рекомендация:** Используйте zap или zerolog для новых проектов.

## Correlation IDs (Trace IDs)

Correlation ID - уникальный идентификатор запроса, который прокидывается через все сервисы.

### Зачем нужны

```
User Request → API Gateway → Auth Service → User Service → Database
    │              │              │              │
    └──────────────┴──────────────┴──────────────┘
              Один и тот же request_id
```

Позволяет:
- Отследить запрос через все сервисы
- Найти все логи, относящиеся к одному запросу
- Понять, где произошла ошибка

### Реализация с middleware

```go
package main

import (
    "context"
    "github.com/google/uuid"
    "go.uber.org/zap"
    "net/http"
)

type contextKey string

const requestIDKey contextKey = "request_id"

// Middleware для добавления request ID
func RequestIDMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Получаем из заголовка или генерируем новый
        requestID := r.Header.Get("X-Request-ID")
        if requestID == "" {
            requestID = uuid.New().String()
        }

        // Добавляем в context
        ctx := context.WithValue(r.Context(), requestIDKey, requestID)

        // Добавляем в response headers
        w.Header().Set("X-Request-ID", requestID)

        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// Logger middleware для логирования с request ID
func LoggerMiddleware(logger *zap.Logger) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            requestID, _ := r.Context().Value(requestIDKey).(string)

            // Создаем logger с request_id
            reqLogger := logger.With(
                zap.String("request_id", requestID),
                zap.String("method", r.Method),
                zap.String("path", r.URL.Path),
            )

            // Добавляем logger в context
            ctx := context.WithValue(r.Context(), "logger", reqLogger)

            reqLogger.Info("request started")
            next.ServeHTTP(w, r.WithContext(ctx))
            reqLogger.Info("request completed")
        })
    }
}

// Использование в handler
func handleUser(w http.ResponseWriter, r *http.Request) {
    logger := r.Context().Value("logger").(*zap.Logger)

    userID := r.URL.Query().Get("user_id")

    logger.Info("fetching user",
        zap.String("user_id", userID),
    )

    // ... обработка запроса

    logger.Info("user fetched successfully")
}

func main() {
    logger, _ := zap.NewProduction()
    defer logger.Sync()

    mux := http.NewServeMux()
    mux.HandleFunc("/api/user", handleUser)

    // Применяем middleware
    handler := RequestIDMiddleware(LoggerMiddleware(logger)(mux))

    http.ListenAndServe(":8080", handler)
}
```

**Логи:**
```json
{"level":"info","request_id":"550e8400-e29b-41d4-a716-446655440000","method":"GET","path":"/api/user","msg":"request started"}
{"level":"info","request_id":"550e8400-e29b-41d4-a716-446655440000","user_id":"john","msg":"fetching user"}
{"level":"info","request_id":"550e8400-e29b-41d4-a716-446655440000","msg":"user fetched successfully"}
{"level":"info","request_id":"550e8400-e29b-41d4-a716-446655440000","msg":"request completed"}
```

## Best Practices

### 1. Что логировать

✅ **Логировать:**
- Старт/остановка приложения
- Входящие HTTP запросы
- Вызовы внешних API
- Database queries (медленные)
- Ошибки и исключения
- Важные бизнес-события (регистрация, покупка)
- Security события (неудачные попытки входа)

❌ **НЕ логировать:**
- Пароли, токены, API keys
- Номера кредитных карт, персональные данные
- Слишком частые события (каждую итерацию цикла)
- Весь request/response body (только summary)

### 2. Structured Fields

```go
// ✅ Хорошо - структурированные поля
logger.Info("user registered",
    zap.String("user_id", userID),
    zap.String("email", email),
    zap.Time("registered_at", time.Now()),
)

// ❌ Плохо - все в одной строке
logger.Info(fmt.Sprintf("User %s with email %s registered at %s", userID, email, time.Now()))
```

### 3. Log Sampling для high-volume логов

```go
// Логировать только 10% успешных запросов
logger := zap.NewProduction(zap.WrapCore(func(core zapcore.Core) zapcore.Core {
    return zapcore.NewSamplerWithOptions(
        core,
        time.Second,
        10,  // первые 10 логов в секунду
        10,  // потом каждый 10-й
    )
}))
```

### 4. Context Logging

```go
// Добавляем общие поля в context logger
func handleRequest(ctx context.Context, userID string) {
    logger := loggerFromContext(ctx).With(
        zap.String("user_id", userID),
        zap.String("function", "handleRequest"),
    )

    logger.Info("processing request")
    // Все последующие логи автоматически будут содержать user_id
}
```

### 5. Rotation и Retention

```go
import "gopkg.in/natefinch/lumberjack.v2"

logger := zap.New(zapcore.NewCore(
    zapcore.NewJSONEncoder(zap.NewProductionEncoderConfig()),
    zapcore.AddSync(&lumberjack.Logger{
        Filename:   "/var/log/app.log",
        MaxSize:    100, // MB
        MaxBackups: 3,
        MaxAge:     28, // days
        Compress:   true,
    }),
    zap.InfoLevel,
))
```

### 6. Разные уровни для разных окружений

```go
func NewLogger(env string) *zap.Logger {
    if env == "production" {
        logger, _ := zap.NewProduction() // INFO и выше
        return logger
    }
    logger, _ := zap.NewDevelopment() // DEBUG и выше
    return logger
}
```

## Интеграция с системами логирования

### ELK Stack (Elasticsearch, Logstash, Kibana)

```go
// Логи в JSON → Filebeat → Logstash → Elasticsearch → Kibana

logger, _ := zap.NewProduction() // JSON формат
logger.Info("event", zap.String("key", "value"))

// В Kibana можно фильтровать: key:"value"
```

### Loki (Grafana)

```go
// Loki - как Prometheus, но для логов

logger.Info("http request",
    zap.String("method", "GET"),
    zap.String("path", "/api/users"),
    zap.Int("status", 200),
)

// LogQL query: {job="api"} |= "error" | json
```

### Sentry (для ошибок)

```go
import "github.com/getsentry/sentry-go"

sentry.CaptureException(err)

// Sentry группирует похожие ошибки, отслеживает частоту, показывает stack traces
```

## Производительность логирования

### Benchmark: zap vs zerolog vs logrus

```
BenchmarkZap       5000000    250 ns/op    0 B/op    0 allocs/op
BenchmarkZerolog   5000000    220 ns/op    0 B/op    0 allocs/op
BenchmarkLogrus    1000000   1500 ns/op  512 B/op   16 allocs/op
```

**Вывод:** zap и zerolog в ~7 раз быстрее logrus.

### Советы по оптимизации

1. ✅ Используйте zap или zerolog
2. ✅ Не логируйте в hot path (внутри tight loops)
3. ✅ Используйте sampling для high-volume логов
4. ✅ Асинхронная запись в файл
5. ❌ Избегайте fmt.Sprintf в логах

## Типичные ошибки

### Ошибка 1: Логирование чувствительных данных

```go
// ❌ Плохо
logger.Info("user login", zap.String("password", password))

// ✅ Хорошо
logger.Info("user login", zap.String("user_id", userID))
```

### Ошибка 2: Слишком много логов

```go
// ❌ Плохо - логируем каждую итерацию
for i := 0; i < 1000000; i++ {
    logger.Debug("processing item", zap.Int("index", i))
}

// ✅ Хорошо - логируем батчами
for i := 0; i < 1000000; i++ {
    if i%10000 == 0 {
        logger.Info("progress", zap.Int("processed", i))
    }
}
```

### Ошибка 3: Игнорирование logger.Sync()

```go
// ❌ Плохо - последние логи могут потеряться
logger.Info("shutting down")
os.Exit(0)

// ✅ Хорошо - сбрасываем буфер
logger.Info("shutting down")
logger.Sync()
os.Exit(0)
```

## Вопросы с собеседований

**Вопрос:** В чем разница между логированием и метриками?

**Ответ:**
- **Логи** - дискретные события с деталями ("user X logged in at Y")
- **Метрики** - агрегированные числа ("1000 requests/sec")
- Логи хороши для debugging, метрики - для alerting
- Метрики дешевле хранить долго

**Вопрос:** Как логировать в микросервисной архитектуре?

**Ответ:**
- Использовать Correlation ID для трейсинга запросов
- Централизованное хранилище логов (ELK, Loki)
- Structured logging для легкого парсинга
- Единый формат логов across всех сервисов
- Логировать границы сервисов (входящие/исходящие запросы)

**Вопрос:** Когда использовать DEBUG vs INFO?

**Ответ:**
- **DEBUG** - детальная информация для разработчиков (выключен в prod)
- **INFO** - важные события для операторов (всегда включен)
- Если лог нужен только при отладке - DEBUG
- Если лог нужен для понимания работы системы - INFO

**Вопрос:** Как обрабатывать логи в Kubernetes?

**Ответ:**
- Логировать в stdout/stderr (Docker/Kubernetes собирают автоматически)
- Использовать sidecar контейнеры для пересылки логов
- Или DaemonSet (Fluentd, Filebeat) для сбора с нод
- Централизованное хранилище (Elasticsearch, Loki)

## Связанные темы

- [[Метрики и наблюдаемость]]
- [[Prometheus]]
- [[Grafana]]
- [[Микросервисная архитектура]]
- [[Go - Context]]
