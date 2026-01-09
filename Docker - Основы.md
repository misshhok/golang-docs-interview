# Docker - Основы

Docker — платформа для разработки, доставки и запуска приложений в контейнерах. Контейнеризация обеспечивает изоляцию и воспроизводимость окружения.

## Зачем Docker?

### Без Docker

```
Developer: "На моей машине работает!"

Dev Machine:         Production:
- Ubuntu 20.04       - CentOS 7
- Python 3.9         - Python 3.6
- Node 16            - Node 12
- PostgreSQL 13      - PostgreSQL 11

→ Конфликты зависимостей
→ "Работает на моей машине, но не на продакшене"
```

### С Docker

```
Developer: "Работает в контейнере!"

Container (везде одинаковый):
- Ubuntu 20.04
- Python 3.9
- Node 16
- PostgreSQL 13

Dev → Staging → Production
→ Одинаковое окружение везде
```

## Основные концепции

### Image (Образ)

Read-only шаблон для создания контейнера.

```
Image = ОС + приложение + зависимости

Пример:
golang:1.21-alpine = Alpine Linux + Go 1.21
postgres:15 = Debian + PostgreSQL 15
```

### Container (Контейнер)

Запущенный экземпляр image.

```
Container = Running Image

Один image → много containers
```

### Dockerfile

Инструкции для создания image.

```dockerfile
FROM golang:1.21-alpine
WORKDIR /app
COPY . .
RUN go build -o main .
CMD ["./main"]
```

### Registry

Хранилище images (Docker Hub, GitHub Container Registry, etc).

```
docker pull postgres:15  # Скачать image
docker push myapp:v1.0   # Загрузить image
```

## Установка Docker

### Linux

```bash
# Ubuntu/Debian
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Добавить пользователя в группу docker
sudo usermod -aG docker $USER
newgrp docker

# Проверка
docker --version
docker run hello-world
```

### macOS / Windows

Скачать [Docker Desktop](https://www.docker.com/products/docker-desktop)

## Основные команды

### Работа с images

```bash
# Список images
docker images
docker image ls

# Скачать image
docker pull nginx:latest
docker pull postgres:15

# Удалить image
docker rmi nginx:latest
docker image rm nginx:latest

# Удалить неиспользуемые images
docker image prune

# Построить image из Dockerfile
docker build -t myapp:v1.0 .

# Показать историю слоев image
docker history nginx:latest

# Информация об image
docker inspect nginx:latest
```

### Работа с containers

```bash
# Запустить container
docker run nginx

# Запустить в фоне (-d = detached)
docker run -d nginx

# С именем
docker run -d --name my-nginx nginx

# С портами (-p host:container)
docker run -d -p 8080:80 nginx

# С переменными окружения
docker run -d -e POSTGRES_PASSWORD=secret postgres:15

# С volumes
docker run -d -v /host/path:/container/path nginx

# Интерактивный режим (-it)
docker run -it ubuntu bash

# Список запущенных containers
docker ps

# Все containers (включая остановленные)
docker ps -a

# Остановить container
docker stop my-nginx

# Запустить остановленный container
docker start my-nginx

# Перезапустить container
docker restart my-nginx

# Удалить container
docker rm my-nginx

# Удалить запущенный container (force)
docker rm -f my-nginx

# Удалить все остановленные containers
docker container prune

# Логи container
docker logs my-nginx
docker logs -f my-nginx  # Follow (tail)

# Войти в запущенный container
docker exec -it my-nginx bash

# Статистика ресурсов
docker stats

# Информация о container
docker inspect my-nginx
```

## Dockerfile для Go приложения

### Простой Dockerfile

```dockerfile
# Базовый image
FROM golang:1.21-alpine

# Рабочая директория
WORKDIR /app

# Копируем go.mod и go.sum
COPY go.mod go.sum ./

# Скачиваем зависимости
RUN go mod download

# Копируем исходный код
COPY . .

# Компилируем приложение
RUN go build -o main .

# Порт, который слушает приложение
EXPOSE 8080

# Команда запуска
CMD ["./main"]
```

### Сборка и запуск

```bash
# Построить image
docker build -t myapp:v1.0 .

# Запустить container
docker run -d -p 8080:8080 --name myapp myapp:v1.0

# Проверить
curl http://localhost:8080

# Логи
docker logs myapp

# Остановить
docker stop myapp
```

## Работа с volumes (данные)

### Bind Mount

Монтирует директорию с хоста в container.

```bash
# Монтируем текущую директорию в /app
docker run -d -v $(pwd):/app -p 8080:8080 myapp

# На Windows
docker run -d -v %cd%:/app -p 8080:8080 myapp
```

### Named Volume

Docker управляет volume (предпочтительный вариант).

```bash
# Создать volume
docker volume create mydata

# Использовать volume
docker run -d -v mydata:/data postgres:15

# Список volumes
docker volume ls

# Удалить volume
docker volume rm mydata

# Удалить неиспользуемые volumes
docker volume prune
```

### Пример: PostgreSQL с volume

```bash
# Создать volume для данных
docker volume create postgres-data

# Запустить PostgreSQL
docker run -d \
  --name postgres \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=mydb \
  -v postgres-data:/var/lib/postgresql/data \
  -p 5432:5432 \
  postgres:15

# Подключиться к БД
docker exec -it postgres psql -U postgres -d mydb
```

## Сети (Networks)

### Типы сетей

```bash
# Bridge (по умолчанию) - containers на одном хосте
docker network create mynetwork

# Список сетей
docker network ls

# Информация о сети
docker network inspect mynetwork

# Удалить сеть
docker network rm mynetwork
```

### Пример: Go app + PostgreSQL

```bash
# Создать сеть
docker network create myapp-network

# Запустить PostgreSQL
docker run -d \
  --name postgres \
  --network myapp-network \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=mydb \
  postgres:15

# Запустить Go приложение
docker run -d \
  --name myapp \
  --network myapp-network \
  -e DATABASE_URL="postgres://postgres:secret@postgres:5432/mydb" \
  -p 8080:8080 \
  myapp:v1.0

# Теперь myapp может обращаться к postgres по имени "postgres"
```

## Environment Variables

```bash
# Передать переменные окружения
docker run -d \
  -e PORT=8080 \
  -e DATABASE_URL="postgres://..." \
  -e DEBUG=true \
  myapp

# Из файла
docker run -d --env-file .env myapp
```

**.env файл:**

```
PORT=8080
DATABASE_URL=postgres://postgres:secret@postgres:5432/mydb
DEBUG=false
```

## Docker в Go приложении

### main.go

```go
package main

import (
    "database/sql"
    "fmt"
    "log"
    "net/http"
    "os"

    _ "github.com/lib/pq"
)

func main() {
    // Читаем переменные окружения
    port := os.Getenv("PORT")
    if port == "" {
        port = "8080"
    }

    dbURL := os.Getenv("DATABASE_URL")
    if dbURL == "" {
        log.Fatal("DATABASE_URL is required")
    }

    // Подключаемся к БД
    db, err := sql.Open("postgres", dbURL)
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    // Health check endpoint
    http.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
        if err := db.Ping(); err != nil {
            http.Error(w, "Database unavailable", http.StatusServiceUnavailable)
            return
        }
        fmt.Fprintf(w, "OK")
    })

    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Hello from Docker!")
    })

    log.Printf("Server listening on port %s", port)
    log.Fatal(http.ListenAndServe(":"+port, nil))
}
```

### Dockerfile

```dockerfile
FROM golang:1.21-alpine AS builder

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN go build -o main .

FROM alpine:latest

# Установить ca-certificates для HTTPS
RUN apk --no-cache add ca-certificates

WORKDIR /root/

# Копировать бинарник из builder
COPY --from=builder /app/main .

EXPOSE 8080

CMD ["./main"]
```

### docker-compose.yml

```yaml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      PORT: "8080"
      DATABASE_URL: "postgres://postgres:secret@postgres:5432/mydb"
    depends_on:
      - postgres
    networks:
      - myapp-network

  postgres:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: mydb
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - myapp-network

networks:
  myapp-network:

volumes:
  postgres-data:
```

**Запуск:**

```bash
# Построить и запустить
docker-compose up -d

# Логи
docker-compose logs -f

# Остановить
docker-compose down

# Остановить и удалить volumes
docker-compose down -v
```

## .dockerignore

Исключает файлы из контекста сборки.

```
# .dockerignore
.git
.env
*.md
node_modules/
vendor/
tmp/
*.log
Dockerfile
docker-compose.yml
```

## Health Checks

```dockerfile
FROM golang:1.21-alpine

WORKDIR /app
COPY . .
RUN go build -o main .

EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:8080/health || exit 1

CMD ["./main"]
```

**Проверка:**

```bash
# Статус health check
docker ps

# Логи health check
docker inspect --format='{{json .State.Health}}' myapp | jq
```

## Ограничение ресурсов

```bash
# Ограничить CPU и память
docker run -d \
  --name myapp \
  --cpus="0.5" \
  --memory="512m" \
  -p 8080:8080 \
  myapp

# CPU shares (относительная weight)
docker run -d --cpu-shares=512 myapp

# Memory limit
docker run -d --memory="1g" --memory-swap="2g" myapp
```

## Debugging

### Войти в container

```bash
# Bash
docker exec -it myapp bash

# Sh (для alpine)
docker exec -it myapp sh

# Запустить другую команду
docker exec -it myapp ps aux
docker exec -it myapp env
```

### Копировать файлы

```bash
# Из container на хост
docker cp myapp:/app/logs/app.log ./app.log

# С хоста в container
docker cp ./config.yaml myapp:/app/config.yaml
```

### Логи

```bash
# Последние логи
docker logs myapp

# Follow (tail -f)
docker logs -f myapp

# Последние N строк
docker logs --tail 100 myapp

# С timestamp
docker logs -t myapp
```

## Best Practices

1. ✅ **Используйте alpine images** - меньше размер
   ```dockerfile
   FROM golang:1.21-alpine  # ~300MB
   # vs
   FROM golang:1.21         # ~800MB
   ```

2. ✅ **Multi-stage builds** - уменьшают размер финального image
   ```dockerfile
   FROM golang:1.21-alpine AS builder
   # ... сборка
   FROM alpine:latest
   COPY --from=builder /app/main .
   ```

3. ✅ **Минимизируйте слои** - объединяйте команды
   ```dockerfile
   # Плохо (3 слоя)
   RUN apt-get update
   RUN apt-get install -y git
   RUN apt-get clean

   # Хорошо (1 слой)
   RUN apt-get update && \
       apt-get install -y git && \
       apt-get clean
   ```

4. ✅ **Кэшируйте зависимости** - копируйте go.mod перед кодом
   ```dockerfile
   COPY go.mod go.sum ./
   RUN go mod download  # Кэшируется
   COPY . .             # Код меняется чаще
   RUN go build
   ```

5. ✅ **Не запускайте от root**
   ```dockerfile
   RUN adduser -D -u 1000 appuser
   USER appuser
   ```

6. ✅ **Используйте .dockerignore**

7. ✅ **Один процесс на container**

8. ✅ **Health checks**

9. ❌ **Не храните secrets в image**

10. ❌ **Не устанавливайте ненужные пакеты**

## Очистка

```bash
# Удалить все остановленные containers
docker container prune

# Удалить неиспользуемые images
docker image prune

# Удалить неиспользуемые volumes
docker volume prune

# Удалить неиспользуемые networks
docker network prune

# Удалить ВСЁ неиспользуемое
docker system prune -a

# С volumes
docker system prune -a --volumes
```

## Связанные темы

- [[Docker - Dockerfile best practices]]
- [[Docker - Docker Compose]]
- [[Docker - Многоступенчатые сборки]]
- [[Kubernetes - Основы]]
- [[Микросервисная архитектура]]
- [[CI-CD - Основные концепции]]
