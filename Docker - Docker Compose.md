# Docker - Docker Compose

Docker Compose - инструмент для определения и запуска multi-container Docker приложений через YAML конфигурацию.

## Зачем Docker Compose?

### Без Docker Compose

```bash
# Создать сеть
docker network create myapp-network

# Запустить PostgreSQL
docker run -d \
  --name postgres \
  --network myapp-network \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=mydb \
  -v postgres-data:/var/lib/postgresql/data \
  -p 5432:5432 \
  postgres:15

# Запустить Redis
docker run -d \
  --name redis \
  --network myapp-network \
  -p 6379:6379 \
  redis:7-alpine

# Запустить приложение
docker run -d \
  --name myapp \
  --network myapp-network \
  -e DATABASE_URL=postgres://postgres:secret@postgres:5432/mydb \
  -e REDIS_URL=redis://redis:6379 \
  -p 8080:8080 \
  myapp:latest
```

Проблемы:
- Много команд
- Сложно воспроизвести
- Нет единого управления

### С Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: mydb
    volumes:
      - postgres-data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  app:
    build: .
    environment:
      DATABASE_URL: postgres://postgres:secret@postgres:5432/mydb
      REDIS_URL: redis://redis:6379
    ports:
      - "8080:8080"
    depends_on:
      - postgres
      - redis

volumes:
  postgres-data:
```

Запуск:
```bash
docker-compose up -d
```

Преимущества:
- Декларативная конфигурация
- Одна команда для всех сервисов
- Легко версионируется в Git
- Автоматическое создание сетей

## Установка Docker Compose

### Linux
```bash
# Установить Docker Compose v2
sudo apt-get update
sudo apt-get install docker-compose-plugin

# Проверка
docker compose version
```

### macOS / Windows
Docker Compose включен в Docker Desktop.

## Основные команды

```bash
# Запустить все сервисы в фоне
docker-compose up -d

# Запустить с пересборкой images
docker-compose up -d --build

# Запустить конкретный сервис
docker-compose up -d postgres

# Остановить все сервисы
docker-compose down

# Остановить и удалить volumes
docker-compose down -v

# Просмотр логов
docker-compose logs

# Логи конкретного сервиса
docker-compose logs -f app

# Список запущенных сервисов
docker-compose ps

# Выполнить команду в сервисе
docker-compose exec app sh

# Перезапустить сервис
docker-compose restart app

# Остановить сервис
docker-compose stop app

# Запустить остановленный сервис
docker-compose start app

# Масштабировать сервис (запустить N копий)
docker-compose up -d --scale app=3

# Проверить конфигурацию
docker-compose config

# Показать используемые порты
docker-compose port app 8080
```

## Структура docker-compose.yml

### Базовая структура

```yaml
version: '3.8'  # Версия формата

services:       # Определение сервисов
  service1:
    # ...
  service2:
    # ...

networks:       # Определение сетей (опционально)
  # ...

volumes:        # Определение volumes (опционально)
  # ...

configs:        # Конфигурации (опционально)
  # ...

secrets:        # Секреты (опционально)
  # ...
```

### Пример: Go приложение с PostgreSQL и Redis

```yaml
version: '3.8'

services:
  # PostgreSQL database
  postgres:
    image: postgres:15-alpine
    container_name: myapp-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: ${DB_PASSWORD:-secret}
      POSTGRES_DB: mydb
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./scripts/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - backend

  # Redis cache
  redis:
    image: redis:7-alpine
    container_name: myapp-redis
    restart: unless-stopped
    command: redis-server --appendonly yes
    volumes:
      - redis-data:/data
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3
    networks:
      - backend

  # Go application
  app:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        - GO_VERSION=1.21
    container_name: myapp
    restart: unless-stopped
    environment:
      - PORT=8080
      - DATABASE_URL=postgres://postgres:${DB_PASSWORD:-secret}@postgres:5432/mydb?sslmode=disable
      - REDIS_URL=redis://redis:6379
      - LOG_LEVEL=info
    ports:
      - "8080:8080"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "--spider", "http://localhost:8080/health"]
      interval: 30s
      timeout: 3s
      retries: 3
      start_period: 10s
    networks:
      - backend
    volumes:
      - ./logs:/app/logs

networks:
  backend:
    driver: bridge

volumes:
  postgres-data:
  redis-data:
```

**.env файл:**
```env
DB_PASSWORD=super_secret_password
```

## Ключевые параметры services

### image

Указать готовый image:
```yaml
services:
  nginx:
    image: nginx:1.25-alpine
```

### build

Собрать image из Dockerfile:
```yaml
services:
  app:
    build: .  # Dockerfile в текущей директории
```

С дополнительными параметрами:
```yaml
services:
  app:
    build:
      context: ./backend
      dockerfile: Dockerfile.prod
      args:
        - GO_VERSION=1.21
        - BUILD_ENV=production
      target: production  # Multi-stage build stage
```

### container_name

Явное имя контейнера:
```yaml
services:
  app:
    container_name: myapp-backend
```

По умолчанию: `<project>_<service>_<replica_number>` (например `myapp_app_1`)

### restart

Политика перезапуска:
```yaml
services:
  app:
    restart: unless-stopped
```

Опции:
- `no` - не перезапускать (по умолчанию)
- `always` - всегда перезапускать
- `on-failure` - только при ошибке
- `unless-stopped` - всегда, кроме явной остановки

### environment

Переменные окружения:
```yaml
services:
  app:
    environment:
      - PORT=8080
      - DEBUG=true
      DATABASE_URL: postgres://...  # Альтернативный синтаксис
```

Из файла:
```yaml
services:
  app:
    env_file:
      - .env
      - .env.production
```

### ports

Маппинг портов `host:container`:
```yaml
services:
  app:
    ports:
      - "8080:8080"        # host 8080 → container 8080
      - "127.0.0.1:8081:8080"  # Только localhost
      - "9000-9005:9000"   # Диапазон портов
```

### expose

Открыть порты только для других сервисов (не для хоста):
```yaml
services:
  backend:
    expose:
      - "8080"
```

### volumes

**Named volume:**
```yaml
services:
  postgres:
    volumes:
      - postgres-data:/var/lib/postgresql/data

volumes:
  postgres-data:
```

**Bind mount:**
```yaml
services:
  app:
    volumes:
      - ./logs:/app/logs
      - ./config.yaml:/app/config.yaml:ro  # Read-only
```

**Tmpfs (в памяти):**
```yaml
services:
  app:
    tmpfs:
      - /tmp
```

### depends_on

Определить зависимости запуска:
```yaml
services:
  app:
    depends_on:
      - postgres
      - redis
```

С проверкой health check:
```yaml
services:
  app:
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
```

Условия:
- `service_started` - сервис запущен
- `service_healthy` - сервис здоров (требует healthcheck)
- `service_completed_successfully` - сервис завершился успешно

### networks

Подключить к сетям:
```yaml
services:
  app:
    networks:
      - frontend
      - backend

networks:
  frontend:
  backend:
```

С алиасами:
```yaml
services:
  app:
    networks:
      backend:
        aliases:
          - api
          - backend-api
```

### healthcheck

Проверка здоровья контейнера:
```yaml
services:
  app:
    healthcheck:
      test: ["CMD", "wget", "--spider", "http://localhost:8080/health"]
      interval: 30s
      timeout: 3s
      retries: 3
      start_period: 10s
```

Отключить healthcheck из базового image:
```yaml
services:
  app:
    healthcheck:
      disable: true
```

### command

Переопределить CMD из Dockerfile:
```yaml
services:
  app:
    command: ./main --config production.yaml
```

Множественные команды:
```yaml
services:
  app:
    command: sh -c "sleep 5 && ./main"
```

### entrypoint

Переопределить ENTRYPOINT:
```yaml
services:
  app:
    entrypoint: /app/custom-entrypoint.sh
```

## Пример: Микросервисная архитектура

```yaml
version: '3.8'

services:
  # API Gateway (Nginx)
  gateway:
    image: nginx:alpine
    container_name: api-gateway
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    ports:
      - "80:80"
    depends_on:
      - user-service
      - order-service
    networks:
      - frontend
      - backend

  # User Service
  user-service:
    build:
      context: ./services/user
    container_name: user-service
    environment:
      - PORT=8081
      - DATABASE_URL=postgres://postgres:secret@postgres:5432/users
      - REDIS_URL=redis://redis:6379/0
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - backend
    restart: unless-stopped

  # Order Service
  order-service:
    build:
      context: ./services/order
    container_name: order-service
    environment:
      - PORT=8082
      - DATABASE_URL=postgres://postgres:secret@postgres:5432/orders
      - KAFKA_BROKERS=kafka:9092
    depends_on:
      - postgres
      - kafka
    networks:
      - backend
    restart: unless-stopped

  # PostgreSQL
  postgres:
    image: postgres:15-alpine
    container_name: postgres
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./init-db:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - backend

  # Redis
  redis:
    image: redis:7-alpine
    container_name: redis
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3
    networks:
      - backend

  # Kafka
  kafka:
    image: confluentinc/cp-kafka:7.5.0
    container_name: kafka
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    depends_on:
      - zookeeper
    networks:
      - backend

  # Zookeeper
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    container_name: zookeeper
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    networks:
      - backend

  # Prometheus
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    ports:
      - "9090:9090"
    networks:
      - backend

  # Grafana
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana-data:/var/lib/grafana
    ports:
      - "3000:3000"
    depends_on:
      - prometheus
    networks:
      - backend

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge

volumes:
  postgres-data:
  prometheus-data:
  grafana-data:
```

## Переменные окружения

### Использование .env файла

Docker Compose автоматически читает `.env` файл в той же директории:

**.env:**
```env
# Database
POSTGRES_USER=postgres
POSTGRES_PASSWORD=super_secret
POSTGRES_DB=mydb

# Application
APP_PORT=8080
APP_ENV=production
LOG_LEVEL=info

# External services
REDIS_URL=redis://redis:6379
KAFKA_BROKERS=kafka:9092
```

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  postgres:
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}

  app:
    environment:
      - PORT=${APP_PORT}
      - DATABASE_URL=postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}
      - REDIS_URL=${REDIS_URL}
    ports:
      - "${APP_PORT}:${APP_PORT}"
```

### Использование нескольких .env файлов

```bash
# Development
docker-compose --env-file .env.dev up

# Production
docker-compose --env-file .env.prod up
```

## Override файлы

### docker-compose.override.yml

Автоматически применяется поверх `docker-compose.yml`:

**docker-compose.yml (базовая конфигурация):**
```yaml
version: '3.8'

services:
  app:
    build: .
    environment:
      - DATABASE_URL=postgres://postgres:secret@postgres:5432/mydb
```

**docker-compose.override.yml (локальная разработка):**
```yaml
version: '3.8'

services:
  app:
    volumes:
      - .:/app  # Live reload
    environment:
      - DEBUG=true
    ports:
      - "8080:8080"
      - "2345:2345"  # Delve debugger
```

Запуск:
```bash
# Применяет оба файла
docker-compose up
```

### Production конфигурация

**docker-compose.prod.yml:**
```yaml
version: '3.8'

services:
  app:
    image: myapp:1.0.0  # Использовать готовый image
    restart: always
    environment:
      - DEBUG=false
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '0.50'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
```

Запуск:
```bash
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up
```

## Ограничение ресурсов

```yaml
version: '3.8'

services:
  app:
    deploy:
      resources:
        limits:
          cpus: '1.0'      # Максимум 1 CPU
          memory: 1G       # Максимум 1GB RAM
        reservations:
          cpus: '0.5'      # Гарантированно 0.5 CPU
          memory: 512M     # Гарантированно 512MB RAM
```

Для Docker Compose v2 (без Swarm):
```yaml
services:
  app:
    mem_limit: 1g
    mem_reservation: 512m
    cpus: 1.0
```

## Логирование

```yaml
services:
  app:
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

Драйверы:
- `json-file` - JSON файлы (по умолчанию)
- `syslog` - Syslog
- `journald` - Systemd журнал
- `gelf` - Graylog Extended Log Format
- `fluentd` - Fluentd
- `awslogs` - AWS CloudWatch

## Секреты (Docker Swarm)

```yaml
version: '3.8'

services:
  app:
    secrets:
      - db_password
      - api_key

secrets:
  db_password:
    file: ./secrets/db_password.txt
  api_key:
    external: true
```

В контейнере секреты доступны в `/run/secrets/`:
```go
password, _ := os.ReadFile("/run/secrets/db_password")
```

## Best Practices

### 1. Именование сервисов

```yaml
# Плохо
services:
  s1:
  s2:

# Хорошо
services:
  user-service:
  order-service:
  postgres:
```

### 2. Версионирование images

```yaml
# Плохо
services:
  postgres:
    image: postgres:latest

# Хорошо
services:
  postgres:
    image: postgres:15-alpine
```

### 3. Health checks

Всегда добавляйте health checks для критичных сервисов.

### 4. Volumes для данных

```yaml
services:
  postgres:
    volumes:
      - postgres-data:/var/lib/postgresql/data  # Named volume

volumes:
  postgres-data:  # Данные сохранятся после docker-compose down
```

### 5. .env для секретов

Не коммитьте `.env` в Git:
```gitignore
.env
.env.local
.env.*.local
```

Создайте `.env.example`:
```.env
POSTGRES_PASSWORD=change_me
API_KEY=your_api_key_here
```

### 6. Сети для изоляции

```yaml
services:
  frontend:
    networks:
      - frontend

  backend:
    networks:
      - frontend
      - backend

  postgres:
    networks:
      - backend  # Доступна только backend

networks:
  frontend:
  backend:
```

## Связанные темы

- [[Docker - Основы]]
- [[Docker - Dockerfile best practices]]
- [[Docker - Многоступенчатые сборки]]
- [[Микросервисная архитектура]]
- [[Kubernetes - Основы]]
- [[CI-CD - Основные концепции]]
