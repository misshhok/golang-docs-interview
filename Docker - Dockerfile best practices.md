# Docker - Dockerfile best practices

Best practices для создания эффективных, безопасных и легко поддерживаемых Docker images.

## Основные принципы

### 1. Используйте Alpine images

Alpine Linux - минималистичный дистрибутив (~5MB), что уменьшает размер финального image.

```dockerfile
# Плохо (800MB)
FROM golang:1.21

# Хорошо (300MB)
FROM golang:1.21-alpine
```

**Когда НЕ использовать Alpine:**
- Нужны GNU libc-зависимые библиотеки (Alpine использует musl libc)
- Требуются специфические системные утилиты

### 2. Multi-stage builds

Разделяйте этапы сборки и runtime для минимизации размера image.

**Плохо (один stage):**
```dockerfile
FROM golang:1.21-alpine

WORKDIR /app
COPY . .
RUN go mod download
RUN go build -o main .

CMD ["./main"]
```

Проблемы:
- В финальном image остаются Go компилятор, исходный код, кэш модулей
- Размер: ~400-500MB

**Хорошо (multi-stage):**
```dockerfile
# Stage 1: Build
FROM golang:1.21-alpine AS builder

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main .

# Stage 2: Runtime
FROM alpine:latest

RUN apk --no-cache add ca-certificates

WORKDIR /root/

COPY --from=builder /app/main .

EXPOSE 8080

CMD ["./main"]
```

Результат:
- Размер: ~15-20MB
- Только бинарник и необходимые зависимости

### 3. Минимизируйте количество слоев

Каждая инструкция `RUN`, `COPY`, `ADD` создает новый слой в image.

**Плохо (5 слоев):**
```dockerfile
RUN apk update
RUN apk add git
RUN apk add curl
RUN apk add ca-certificates
RUN rm -rf /var/cache/apk/*
```

**Хорошо (1 слой):**
```dockerfile
RUN apk update && \
    apk add --no-cache git curl ca-certificates && \
    rm -rf /var/cache/apk/*
```

### 4. Кэшируйте зависимости

Docker кэширует слои. Копируйте `go.mod` и `go.sum` отдельно от исходного кода.

**Плохо:**
```dockerfile
COPY . .
RUN go mod download  # Пересобирается при любом изменении кода
```

**Хорошо:**
```dockerfile
COPY go.mod go.sum ./
RUN go mod download      # Кэшируется, пока не изменятся зависимости

COPY . .
RUN go build -o main .
```

**Порядок важен:**
- Сначала: редко меняющиеся файлы (зависимости)
- Потом: часто меняющиеся файлы (исходный код)

### 5. Используйте .dockerignore

Исключайте ненужные файлы из контекста сборки.

**.dockerignore:**
```
# Git
.git
.gitignore

# Документация
*.md
README.md
docs/

# Конфигурация
.env
.env.local
*.example

# Зависимости
node_modules/
vendor/

# Временные файлы
tmp/
*.log
*.swp
*.swo

# Тесты
*_test.go
testdata/

# CI/CD
.github/
.gitlab-ci.yml

# Docker
Dockerfile
docker-compose.yml
.dockerignore
```

Преимущества:
- Ускоряет сборку
- Уменьшает размер контекста
- Предотвращает утечку секретов

### 6. Не запускайте от root

**Проблема:**
```dockerfile
FROM alpine:latest
COPY app /app
CMD ["/app"]  # Запускается от root!
```

Если контейнер скомпрометирован, атакующий получает root-доступ.

**Решение:**
```dockerfile
FROM alpine:latest

# Создать пользователя
RUN adduser -D -u 1000 appuser

# Создать директорию и назначить владельца
WORKDIR /app
COPY --chown=appuser:appuser app /app

# Переключиться на пользователя
USER appuser

CMD ["/app"]
```

**Для Go приложений:**
```dockerfile
FROM alpine:latest

RUN adduser -D -u 1000 appuser

WORKDIR /home/appuser

COPY --from=builder --chown=appuser:appuser /app/main .

USER appuser

CMD ["./main"]
```

### 7. Используйте конкретные версии тегов

**Плохо:**
```dockerfile
FROM golang:latest       # Может сломаться в будущем
FROM postgres:latest
FROM alpine:latest
```

**Хорошо:**
```dockerfile
FROM golang:1.21-alpine  # Конкретная версия
FROM postgres:15
FROM alpine:3.18
```

**Еще лучше (digest):**
```dockerfile
FROM golang:1.21-alpine@sha256:abcd1234...
```

Digest гарантирует точно такой же image.

### 8. Один процесс на контейнер

**Плохо (несколько процессов):**
```dockerfile
CMD nginx && go run main.go && postgres
```

Проблемы:
- Сложно управлять жизненным циклом
- Проблемы с логами
- Нарушает принцип единственной ответственности

**Хорошо:**
```yaml
# docker-compose.yml
services:
  nginx:
    image: nginx:alpine

  app:
    image: myapp:latest

  postgres:
    image: postgres:15
```

Каждый сервис в отдельном контейнере.

### 9. Оптимизируйте размер бинарника

**Go флаги компиляции:**
```dockerfile
RUN CGO_ENABLED=0 GOOS=linux go build \
    -a \
    -installsuffix cgo \
    -ldflags="-w -s" \
    -o main .
```

Флаги:
- `CGO_ENABLED=0` - отключить CGO (статическая линковка)
- `-a` - пересобрать все пакеты
- `-installsuffix cgo` - суффикс для установки
- `-ldflags="-w -s"` - убрать debug информацию и таблицу символов

**UPX компрессия (опционально):**
```dockerfile
FROM golang:1.21-alpine AS builder

WORKDIR /app
COPY . .
RUN go build -ldflags="-w -s" -o main .

# Сжать бинарник
RUN apk add --no-cache upx && \
    upx --best --lzma main

FROM alpine:latest
COPY --from=builder /app/main .
CMD ["./main"]
```

Результат: уменьшение размера на 50-70%.

### 10. Безопасность: не храните секреты в image

**Плохо:**
```dockerfile
ENV DATABASE_PASSWORD=secret123  # Секрет в image!
ENV API_KEY=abcd1234
```

Любой с доступом к image может извлечь секреты:
```bash
docker history myapp:latest
```

**Хорошо (runtime secrets):**
```bash
# Передать через environment variables
docker run -e DATABASE_PASSWORD=secret123 myapp

# Через Docker secrets (Swarm/Kubernetes)
docker secret create db_password password.txt
```

**Build-time secrets (BuildKit):**
```dockerfile
# syntax=docker/dockerfile:1

FROM alpine

# Монтировать секрет только на время сборки
RUN --mount=type=secret,id=github_token \
    git clone https://$(cat /run/secrets/github_token)@github.com/private/repo.git
```

Сборка:
```bash
docker build --secret id=github_token,src=token.txt .
```

### 11. Health checks

Добавляйте health checks для мониторинга состояния контейнера.

```dockerfile
FROM golang:1.21-alpine

WORKDIR /app
COPY . .
RUN go build -o main .

EXPOSE 8080

# Health check каждые 30 секунд
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8080/health || exit 1

CMD ["./main"]
```

**В docker-compose:**
```yaml
services:
  app:
    image: myapp
    healthcheck:
      test: ["CMD", "wget", "--spider", "http://localhost:8080/health"]
      interval: 30s
      timeout: 3s
      retries: 3
      start_period: 10s
```

**Go health check endpoint:**
```go
http.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
    // Проверить подключение к БД
    if err := db.Ping(); err != nil {
        http.Error(w, "Database unavailable", http.StatusServiceUnavailable)
        return
    }

    // Проверить другие зависимости (Redis, etc)

    w.WriteHeader(http.StatusOK)
    fmt.Fprint(w, "OK")
})
```

### 12. Используйте COPY вместо ADD

**`ADD`** имеет дополнительную магию:
- Автоматически распаковывает tar архивы
- Может скачивать файлы по URL

**`COPY`** - предсказуемое копирование файлов.

**Плохо:**
```dockerfile
ADD . /app  # Неявное поведение
```

**Хорошо:**
```dockerfile
COPY . /app  # Явное копирование
```

Используйте `ADD` только если нужна распаковка:
```dockerfile
ADD app.tar.gz /app/
```

### 13. Группируйте связанные команды

**Плохо:**
```dockerfile
RUN apt-get update
RUN apt-get install -y git
RUN git clone https://github.com/...
RUN cd repo && make install
RUN rm -rf /var/lib/apt/lists/*
```

Проблемы:
- Много слоев
- Кэш `apt` остается в промежуточных слоях

**Хорошо:**
```dockerfile
RUN apt-get update && \
    apt-get install -y --no-install-recommends git && \
    git clone https://github.com/... && \
    cd repo && make install && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

### 14. Метаданные с LABEL

Добавляйте метаданные для документирования image.

```dockerfile
FROM golang:1.21-alpine

LABEL maintainer="team@example.com"
LABEL version="1.0.0"
LABEL description="My awesome Go application"
LABEL org.opencontainers.image.source="https://github.com/myorg/myapp"

WORKDIR /app
# ...
```

Просмотр:
```bash
docker inspect myapp:latest | jq '.[0].Config.Labels'
```

### 15. Порядок инструкций для оптимизации кэша

От менее изменяемых к более изменяемым:

```dockerfile
# 1. Базовый image (меняется редко)
FROM golang:1.21-alpine AS builder

# 2. Системные зависимости (меняются редко)
RUN apk add --no-cache git ca-certificates

# 3. Рабочая директория
WORKDIR /app

# 4. Go модули (меняются иногда)
COPY go.mod go.sum ./
RUN go mod download

# 5. Исходный код (меняется часто)
COPY . .

# 6. Сборка
RUN go build -o main .

# Runtime stage
FROM alpine:latest

RUN apk --no-cache add ca-certificates

WORKDIR /root/

# 7. Копирование бинарника
COPY --from=builder /app/main .

# 8. Runtime конфигурация
EXPOSE 8080

# 9. Команда запуска
CMD ["./main"]
```

## Полный пример: production-ready Dockerfile

```dockerfile
# syntax=docker/dockerfile:1

# ========================================
# Stage 1: Dependencies
# ========================================
FROM golang:1.21-alpine AS deps

# Установить git для приватных модулей
RUN apk add --no-cache git ca-certificates

WORKDIR /app

# Кэшировать зависимости
COPY go.mod go.sum ./
RUN go mod download && go mod verify

# ========================================
# Stage 2: Build
# ========================================
FROM golang:1.21-alpine AS builder

WORKDIR /app

# Копировать зависимости из предыдущего stage
COPY --from=deps /go/pkg /go/pkg
COPY --from=deps /app/go.mod /app/go.sum ./

# Копировать исходный код
COPY . .

# Сборка с оптимизацией
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build \
    -a \
    -installsuffix cgo \
    -ldflags="-w -s -X main.version=1.0.0 -X main.commit=$(git rev-parse --short HEAD)" \
    -o main \
    ./cmd/server

# Сжать бинарник (опционально)
RUN apk add --no-cache upx && \
    upx --best --lzma main

# ========================================
# Stage 3: Runtime
# ========================================
FROM alpine:3.18

# Метаданные
LABEL maintainer="devops@company.com"
LABEL version="1.0.0"
LABEL description="Production Go microservice"

# Установить ca-certificates для HTTPS
RUN apk --no-cache add ca-certificates tzdata && \
    # Создать пользователя
    adduser -D -u 1000 appuser

# Копировать временные зоны
COPY --from=builder /usr/share/zoneinfo /usr/share/zoneinfo

WORKDIR /home/appuser

# Копировать бинарник
COPY --from=builder --chown=appuser:appuser /app/main .

# Копировать конфигурацию (если нужно)
# COPY --chown=appuser:appuser config.yaml .

# Переключиться на непривилегированного пользователя
USER appuser

# Открыть порт
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8080/health || exit 1

# Запуск
CMD ["./main"]
```

**Сборка:**
```bash
# С BuildKit для ускорения
DOCKER_BUILDKIT=1 docker build -t myapp:1.0.0 .

# Multi-platform сборка
docker buildx build --platform linux/amd64,linux/arm64 -t myapp:1.0.0 .
```

## Сравнение размеров

| Подход | Размер | Описание |
|--------|--------|----------|
| `FROM golang:1.21` | ~800MB | Полный Go image |
| `FROM golang:1.21-alpine` | ~300MB | Alpine с Go |
| Multi-stage (alpine runtime) | ~20MB | Только бинарник + Alpine |
| Multi-stage + UPX | ~10-15MB | Сжатый бинарник |
| `FROM scratch` | ~5-8MB | Только статический бинарник |

### Dockerfile с scratch

Для минимального размера:

```dockerfile
FROM golang:1.21-alpine AS builder

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .

# Статическая сборка
RUN CGO_ENABLED=0 GOOS=linux go build \
    -a -installsuffix cgo \
    -ldflags="-w -s" \
    -o main .

# Пустой базовый image
FROM scratch

# Копировать CA сертификаты для HTTPS
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# Копировать бинарник
COPY --from=builder /app/main /main

EXPOSE 8080

CMD ["/main"]
```

**Ограничения `scratch`:**
- Нет shell (нельзя использовать `docker exec`)
- Нет утилит для отладки
- Требуется статическая сборка (CGO_ENABLED=0)

## Best practices чеклист

- [ ] Используется Alpine или минимальный базовый image
- [ ] Multi-stage build для разделения сборки и runtime
- [ ] Зависимости копируются отдельно для кэширования
- [ ] Минимизировано количество слоев (объединены RUN команды)
- [ ] Создан .dockerignore
- [ ] Приложение запускается НЕ от root
- [ ] Используются конкретные версии тегов (не latest)
- [ ] Один процесс на контейнер
- [ ] Секреты НЕ хранятся в image
- [ ] Добавлен HEALTHCHECK
- [ ] Используется COPY вместо ADD
- [ ] Добавлены LABEL метаданные
- [ ] Порядок инструкций оптимизирован для кэша
- [ ] Бинарник скомпилирован с оптимизациями (-ldflags="-w -s")

## Связанные темы

- [[Docker - Основы]]
- [[Docker - Docker Compose]]
- [[Docker - Многоступенчатые сборки]]
- [[Go - Простой web-сервер]]
- [[Kubernetes - Основы]]
- [[CI-CD - Основные концепции]]
