# Docker - Многоступенчатые сборки

Multi-stage builds - техника создания Docker images с использованием нескольких промежуточных этапов (stages), где каждый этап может использовать свой базовый image.

## Проблема: большие Docker images

### Без multi-stage (один Dockerfile)

```dockerfile
FROM golang:1.21

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .

RUN go build -o main .

EXPOSE 8080

CMD ["./main"]
```

**Проблемы:**
- **Размер:** ~800MB
- Включает Go компилятор, исходный код, кэш модулей
- Всё это НЕ нужно в runtime
- Увеличивает время загрузки и уязвимости

**Что попадает в image:**
```
- Go компилятор (~300MB)
- Исходный код (.go файлы)
- Кэш модулей
- Build инструменты
- Тестовые файлы
- Документация
+ Бинарник (~10MB)
```

## Решение: Multi-stage build

```dockerfile
# ========================================
# Stage 1: Build
# ========================================
FROM golang:1.21-alpine AS builder

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .

RUN CGO_ENABLED=0 GOOS=linux go build -o main .

# ========================================
# Stage 2: Runtime
# ========================================
FROM alpine:latest

WORKDIR /root/

# Копируем ТОЛЬКО бинарник из предыдущего stage
COPY --from=builder /app/main .

EXPOSE 8080

CMD ["./main"]
```

**Результат:**
- **Размер:** ~15MB (в 50 раз меньше!)
- Только runtime зависимости
- Меньше уязвимостей
- Быстрее загрузка

## Как это работает

### Синтаксис

```dockerfile
FROM <image> AS <stage_name>
```

- `AS <stage_name>` - даём имя stage
- Можно ссылаться на файлы из предыдущих stages через `--from=<stage_name>`

### Копирование из предыдущих stages

```dockerfile
COPY --from=builder /app/main .
```

- `--from=builder` - копировать из stage с именем "builder"
- `/app/main` - путь в builder stage
- `.` - путь в текущем stage

### Пример: 3 stages

```dockerfile
# Stage 1: Dependencies
FROM golang:1.21-alpine AS deps

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download

# Stage 2: Build
FROM golang:1.21-alpine AS builder

WORKDIR /app
COPY --from=deps /go/pkg /go/pkg
COPY . .
RUN go build -o main .

# Stage 3: Runtime
FROM alpine:latest

COPY --from=builder /app/main .
CMD ["./main"]
```

## Практические примеры

### Go приложение (production-ready)

```dockerfile
# ========================================
# Stage 1: Build
# ========================================
FROM golang:1.21-alpine AS builder

# Установить build зависимости
RUN apk add --no-cache git ca-certificates

WORKDIR /app

# Кэшировать модули
COPY go.mod go.sum ./
RUN go mod download && go mod verify

# Копировать исходный код
COPY . .

# Сборка с оптимизацией
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build \
    -a \
    -installsuffix cgo \
    -ldflags="-w -s -X main.version=1.0.0" \
    -o main \
    ./cmd/server

# ========================================
# Stage 2: Runtime
# ========================================
FROM alpine:latest

# Метаданные
LABEL maintainer="team@example.com"
LABEL version="1.0.0"

# Установить runtime зависимости
RUN apk --no-cache add ca-certificates tzdata

# Создать непривилегированного пользователя
RUN adduser -D -u 1000 appuser

WORKDIR /home/appuser

# Копировать бинарник из builder stage
COPY --from=builder /app/main .

# Копировать конфигурационные файлы (если нужно)
# COPY --from=builder /app/config.yaml .

# Переключиться на appuser
USER appuser

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8080/health || exit 1

CMD ["./main"]
```

**Размер:** ~15-20MB

### Go приложение с minimal image (scratch)

```dockerfile
# Stage 1: Build
FROM golang:1.21-alpine AS builder

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .

# Статическая компиляция
RUN CGO_ENABLED=0 GOOS=linux go build \
    -a -installsuffix cgo \
    -ldflags="-w -s" \
    -o main .

# Stage 2: Runtime (пустой image)
FROM scratch

# Копировать CA сертификаты для HTTPS
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# Копировать бинарник
COPY --from=builder /app/main /main

EXPOSE 8080

CMD ["/main"]
```

**Размер:** ~5-8MB (только бинарник)

**Ограничения scratch:**
- Нет shell (нельзя `docker exec -it`)
- Нет утилит для отладки
- Только статические бинарники

### Node.js приложение

```dockerfile
# ========================================
# Stage 1: Dependencies
# ========================================
FROM node:18-alpine AS deps

WORKDIR /app

COPY package.json package-lock.json ./

RUN npm ci --only=production

# ========================================
# Stage 2: Build
# ========================================
FROM node:18-alpine AS builder

WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci

COPY . .

# Сборка (TypeScript, Webpack, etc)
RUN npm run build

# ========================================
# Stage 3: Runtime
# ========================================
FROM node:18-alpine

WORKDIR /app

# Копировать зависимости из deps stage
COPY --from=deps /app/node_modules ./node_modules

# Копировать собранное приложение из builder stage
COPY --from=builder /app/dist ./dist
COPY package.json .

RUN adduser -D -u 1000 appuser && chown -R appuser:appuser /app
USER appuser

EXPOSE 3000

CMD ["node", "dist/index.js"]
```

### Python приложение

```dockerfile
# ========================================
# Stage 1: Build
# ========================================
FROM python:3.11-slim AS builder

WORKDIR /app

# Установить build зависимости
RUN apt-get update && \
    apt-get install -y --no-install-recommends gcc && \
    rm -rf /var/lib/apt/lists/*

# Установить Python зависимости
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# ========================================
# Stage 2: Runtime
# ========================================
FROM python:3.11-slim

WORKDIR /app

# Копировать установленные пакеты из builder
COPY --from=builder /root/.local /root/.local

# Добавить путь к пакетам
ENV PATH=/root/.local/bin:$PATH

# Копировать приложение
COPY . .

RUN useradd -m -u 1000 appuser && chown -R appuser:appuser /app
USER appuser

EXPOSE 8000

CMD ["python", "app.py"]
```

## Продвинутые техники

### Условные stages с build args

```dockerfile
# Stage 1: Base
FROM golang:1.21-alpine AS base

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download

# Stage 2: Development
FROM base AS development

COPY . .
RUN go build -o main .

CMD ["./main"]

# Stage 3: Production
FROM base AS production

COPY . .
RUN CGO_ENABLED=0 go build -ldflags="-w -s" -o main .

FROM alpine:latest AS production-runtime

COPY --from=production /app/main .
CMD ["./main"]
```

Сборка для разных окружений:
```bash
# Development
docker build --target development -t myapp:dev .

# Production
docker build --target production-runtime -t myapp:prod .
```

### Копирование из внешних images

```dockerfile
FROM nginx:alpine

# Копировать из официального image
COPY --from=golang:1.21-alpine /usr/local/go/bin/go /usr/local/bin/

# Копировать из конкретного image
COPY --from=myregistry.com/myapp:1.0.0 /app/config.yaml /config/
```

### Использование named stages для переиспользования

```dockerfile
# ========================================
# Stage: Базовая конфигурация
# ========================================
FROM golang:1.21-alpine AS base

RUN apk add --no-cache git ca-certificates
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download

# ========================================
# Stage: Тесты
# ========================================
FROM base AS test

COPY . .
RUN go test -v ./...

# ========================================
# Stage: Build
# ========================================
FROM base AS builder

COPY . .
RUN CGO_ENABLED=0 go build -o main .

# ========================================
# Stage: Runtime
# ========================================
FROM alpine:latest

COPY --from=builder /app/main .
CMD ["./main"]
```

Запустить только тесты:
```bash
docker build --target test .
```

### Кэширование с BuildKit

**Кэширование модулей Go:**
```dockerfile
# syntax=docker/dockerfile:1

FROM golang:1.21-alpine AS builder

WORKDIR /app

# Использовать кэш для go mod
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=bind,source=go.mod,target=go.mod \
    --mount=type=bind,source=go.sum,target=go.sum \
    go mod download

COPY . .

RUN --mount=type=cache,target=/go/pkg/mod \
    CGO_ENABLED=0 go build -o main .

FROM alpine:latest

COPY --from=builder /app/main .
CMD ["./main"]
```

Сборка:
```bash
DOCKER_BUILDKIT=1 docker build .
```

**Кэширование apt пакетов:**
```dockerfile
# syntax=docker/dockerfile:1

FROM ubuntu:22.04

RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && \
    apt-get install -y git curl
```

## Сравнение размеров

| Подход | Размер | Описание |
|--------|--------|----------|
| Single-stage (golang:1.21) | ~800MB | Полный Go image |
| Single-stage (golang:1.21-alpine) | ~300MB | Alpine с Go |
| Multi-stage (alpine runtime) | ~15-20MB | Только бинарник + Alpine |
| Multi-stage (scratch runtime) | ~5-8MB | Только статический бинарник |
| Multi-stage + UPX | ~3-5MB | Сжатый бинарник |

### Пример сравнения

**Single-stage:**
```dockerfile
FROM golang:1.21

WORKDIR /app
COPY . .
RUN go build -o main .

CMD ["./main"]
```
```bash
$ docker images
myapp-single   latest   812MB
```

**Multi-stage:**
```dockerfile
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN go build -o main .

FROM alpine:latest
COPY --from=builder /app/main .
CMD ["./main"]
```
```bash
$ docker images
myapp-multi   latest   18MB
```

**Экономия: 794MB (97.8%)**

## Real-world пример: Микросервис с миграциями

```dockerfile
# ========================================
# Stage 1: Dependencies
# ========================================
FROM golang:1.21-alpine AS deps

WORKDIR /app

RUN apk add --no-cache git

COPY go.mod go.sum ./
RUN go mod download

# ========================================
# Stage 2: Build application
# ========================================
FROM deps AS app-builder

COPY . .

RUN CGO_ENABLED=0 GOOS=linux go build \
    -ldflags="-w -s" \
    -o /bin/app \
    ./cmd/server

# ========================================
# Stage 3: Build migrations tool
# ========================================
FROM deps AS migrate-builder

RUN go install -tags 'postgres' github.com/golang-migrate/migrate/v4/cmd/migrate@latest

# ========================================
# Stage 4: Runtime
# ========================================
FROM alpine:latest

# Установить runtime зависимости
RUN apk --no-cache add ca-certificates postgresql-client

# Создать пользователя
RUN adduser -D -u 1000 appuser

WORKDIR /home/appuser

# Копировать бинарник приложения
COPY --from=app-builder /bin/app .

# Копировать migrate tool
COPY --from=migrate-builder /go/bin/migrate /usr/local/bin/

# Копировать SQL миграции
COPY --chown=appuser:appuser ./migrations ./migrations

# Копировать entrypoint скрипт
COPY --chown=appuser:appuser ./docker-entrypoint.sh .
RUN chmod +x ./docker-entrypoint.sh

USER appuser

EXPOSE 8080

ENTRYPOINT ["./docker-entrypoint.sh"]
CMD ["./app"]
```

**docker-entrypoint.sh:**
```bash
#!/bin/sh
set -e

# Запустить миграции
echo "Running database migrations..."
migrate -path ./migrations \
        -database "$DATABASE_URL" \
        up

# Запустить приложение
exec "$@"
```

## Best Practices

### 1. Именуйте stages

```dockerfile
# Плохо
FROM golang:1.21-alpine
# ...

FROM alpine:latest
COPY --from=0 /app/main .  # Неясно, откуда копируем
```

```dockerfile
# Хорошо
FROM golang:1.21-alpine AS builder
# ...

FROM alpine:latest
COPY --from=builder /app/main .  # Понятно
```

### 2. Минимизируйте final stage

Финальный stage должен содержать только runtime зависимости:
```dockerfile
FROM alpine:latest

# Только необходимое для runtime
RUN apk --no-cache add ca-certificates

COPY --from=builder /app/main .

CMD ["./main"]
```

### 3. Используйте .dockerignore

Исключайте ненужные файлы из контекста:
```
.git
.env
*.md
vendor/
tmp/
*.log
```

### 4. Кэшируйте зависимости отдельно

```dockerfile
# Сначала зависимости
COPY go.mod go.sum ./
RUN go mod download

# Потом код
COPY . .
RUN go build -o main .
```

### 5. Статическая компиляция для scratch

```dockerfile
RUN CGO_ENABLED=0 GOOS=linux go build \
    -a -installsuffix cgo \
    -ldflags="-w -s" \
    -o main .
```

### 6. Копируйте CA сертификаты для HTTPS

```dockerfile
FROM scratch

COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /app/main .

CMD ["/main"]
```

### 7. Используйте BuildKit для кэширования

```dockerfile
# syntax=docker/dockerfile:1

RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download
```

## Debugging multi-stage builds

### Посмотреть промежуточные images

```bash
# Показать все images, включая intermediate
docker images -a
```

### Сохранить промежуточный stage

```bash
# Собрать только до определённого stage
docker build --target builder -t myapp:builder .

# Запустить контейнер с builder stage для отладки
docker run -it myapp:builder sh
```

### Использовать BuildKit для вывода

```bash
DOCKER_BUILDKIT=1 docker build --progress=plain .
```

## Связанные темы

- [[Docker - Основы]]
- [[Docker - Dockerfile best practices]]
- [[Docker - Docker Compose]]
- [[Go - Простой web-сервер]]
- [[Kubernetes - Основы]]
- [[CI-CD - Основные концепции]]
