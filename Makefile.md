# Makefile

Makefile — это файл с инструкциями для утилиты `make`, которая автоматизирует процесс сборки, тестирования и развертывания. Для Go-разработчика Makefile упрощает запуск типовых команд и делает workflow более удобным.

## Зачем нужен Makefile

1. **Упрощение команд** — вместо длинных команд используем короткие: `make test`, `make build`
2. **Стандартизация** — все в команде используют одни и те же команды
3. **Автоматизация** — можно выстроить зависимости между задачами
4. **Документация** — Makefile служит документацией для типовых операций

## Базовый синтаксис

### Структура правила (rule)

```makefile
target: dependencies
	command
	command
```

⚠️ **ВАЖНО:** Перед командами должен быть **TAB**, не пробелы!

### Простой пример

```makefile
# Makefile

hello:
	echo "Hello, World!"

build:
	go build -o bin/app ./cmd/main.go
```

Запуск:
```bash
make hello  # Выполнит: echo "Hello, World!"
make build  # Выполнит: go build...
```

## Makefile для Go проекта

### Базовый Makefile

```makefile
# Makefile для Go проекта

# Переменные
APP_NAME=myapp
BUILD_DIR=./bin
MAIN_PATH=./cmd/main.go

# Default target (запускается при просто `make`)
.DEFAULT_GOAL := help

# Build приложения
build:
	go build -o $(BUILD_DIR)/$(APP_NAME) $(MAIN_PATH)

# Запуск приложения
run:
	go run $(MAIN_PATH)

# Запуск тестов
test:
	go test -v ./...

# Запуск тестов с покрытием
test-coverage:
	go test -v -coverprofile=coverage.out ./...
	go tool cover -html=coverage.out

# Линтинг
lint:
	golangci-lint run ./...

# Форматирование кода
fmt:
	go fmt ./...

# Очистка
clean:
	rm -rf $(BUILD_DIR)
	rm -f coverage.out

# Помощь
help:
	@echo "Available targets:"
	@echo "  build          - Build the application"
	@echo "  run            - Run the application"
	@echo "  test           - Run tests"
	@echo "  test-coverage  - Run tests with coverage report"
	@echo "  lint           - Run linter"
	@echo "  fmt            - Format code"
	@echo "  clean          - Clean build artifacts"
```

## Переменные

### Определение переменных

```makefile
# Простое присваивание
APP_NAME=myapp
VERSION=1.0.0

# Присваивание с выполнением команды
GIT_COMMIT=$(shell git rev-parse --short HEAD)
BUILD_TIME=$(shell date +%Y-%m-%d_%H:%M:%S)

# Использование переменных
build:
	go build -ldflags "-X main.Version=$(VERSION) -X main.Commit=$(GIT_COMMIT)" \
		-o bin/$(APP_NAME) ./cmd/main.go
```

### Переменные окружения

```makefile
# Чтение из переменных окружения
DB_HOST ?= localhost  # Использовать значение из env или localhost по умолчанию
API_PORT ?= 8080

run:
	DB_HOST=$(DB_HOST) API_PORT=$(API_PORT) go run ./cmd/main.go
```

## Зависимости между targets

### Target зависит от другого target

```makefile
# test зависит от fmt и lint
test: fmt lint
	go test -v ./...

# build зависит от test
build: test
	go build -o bin/app ./cmd/main.go

# deploy зависит от build
deploy: build
	./scripts/deploy.sh
```

При запуске `make deploy` выполнится:
1. `fmt`
2. `lint`
3. `test`
4. `build`
5. `deploy`

### Зависимости от файлов

```makefile
# Пересобрать только если изменились .go файлы
bin/app: $(shell find . -name '*.go')
	go build -o bin/app ./cmd/main.go

# Или с явным указанием
bin/app: main.go handler.go model.go
	go build -o bin/app main.go
```

## .PHONY

`.PHONY` указывает, что target — это команда, а не файл.

```makefile
# Без .PHONY make будет искать файл с таким именем
.PHONY: build test clean run lint fmt help

build:
	go build -o bin/app ./cmd/main.go

test:
	go test -v ./...

clean:
	rm -rf bin/
```

**Зачем нужно:** Если в проекте есть файл `test`, то без `.PHONY` команда `make test` может не выполниться.

## Продвинутый Makefile для Go

```makefile
# Makefile для production-ready Go проекта

# Переменные
APP_NAME := myapp
VERSION := $(shell git describe --tags --always --dirty)
COMMIT := $(shell git rev-parse --short HEAD)
BUILD_TIME := $(shell date +%Y-%m-%d_%H:%M:%S)
BUILD_DIR := ./bin
MAIN_PATH := ./cmd/main.go

# Go параметры
GOCMD := go
GOBUILD := $(GOCMD) build
GOTEST := $(GOCMD) test
GOGET := $(GOCMD) get
GOMOD := $(GOCMD) mod
GOFMT := $(GOCMD) fmt

# Линтер
LINTER := golangci-lint

# Флаги сборки
LDFLAGS := -ldflags "\
	-X main.Version=$(VERSION) \
	-X main.Commit=$(COMMIT) \
	-X main.BuildTime=$(BUILD_TIME)"

# Цвета для вывода
RED := \033[0;31m
GREEN := \033[0;32m
YELLOW := \033[0;33m
NC := \033[0m # No Color

.PHONY: all build test clean run lint fmt help deps install docker-build docker-run

# Default target
.DEFAULT_GOAL := help

## all: Запустить все проверки и сборку
all: fmt lint test build

## build: Собрать приложение
build:
	@echo "$(GREEN)Building $(APP_NAME)...$(NC)"
	@mkdir -p $(BUILD_DIR)
	$(GOBUILD) $(LDFLAGS) -o $(BUILD_DIR)/$(APP_NAME) $(MAIN_PATH)
	@echo "$(GREEN)✓ Build complete: $(BUILD_DIR)/$(APP_NAME)$(NC)"

## build-linux: Собрать для Linux
build-linux:
	@echo "$(GREEN)Building for Linux...$(NC)"
	@mkdir -p $(BUILD_DIR)
	GOOS=linux GOARCH=amd64 $(GOBUILD) $(LDFLAGS) -o $(BUILD_DIR)/$(APP_NAME)-linux $(MAIN_PATH)
	@echo "$(GREEN)✓ Linux build complete$(NC)"

## build-all: Собрать для всех платформ
build-all:
	@echo "$(GREEN)Building for all platforms...$(NC)"
	@mkdir -p $(BUILD_DIR)
	GOOS=linux GOARCH=amd64 $(GOBUILD) $(LDFLAGS) -o $(BUILD_DIR)/$(APP_NAME)-linux-amd64 $(MAIN_PATH)
	GOOS=darwin GOARCH=amd64 $(GOBUILD) $(LDFLAGS) -o $(BUILD_DIR)/$(APP_NAME)-darwin-amd64 $(MAIN_PATH)
	GOOS=darwin GOARCH=arm64 $(GOBUILD) $(LDFLAGS) -o $(BUILD_DIR)/$(APP_NAME)-darwin-arm64 $(MAIN_PATH)
	GOOS=windows GOARCH=amd64 $(GOBUILD) $(LDFLAGS) -o $(BUILD_DIR)/$(APP_NAME)-windows-amd64.exe $(MAIN_PATH)
	@echo "$(GREEN)✓ All builds complete$(NC)"

## run: Запустить приложение
run:
	@echo "$(GREEN)Running $(APP_NAME)...$(NC)"
	$(GOCMD) run $(MAIN_PATH)

## test: Запустить тесты
test:
	@echo "$(YELLOW)Running tests...$(NC)"
	$(GOTEST) -v -race -timeout 30s ./...
	@echo "$(GREEN)✓ Tests passed$(NC)"

## test-coverage: Тесты с покрытием
test-coverage:
	@echo "$(YELLOW)Running tests with coverage...$(NC)"
	$(GOTEST) -v -race -coverprofile=coverage.out -covermode=atomic ./...
	$(GOCMD) tool cover -html=coverage.out -o coverage.html
	@echo "$(GREEN)✓ Coverage report: coverage.html$(NC)"

## test-short: Быстрые тесты
test-short:
	@echo "$(YELLOW)Running short tests...$(NC)"
	$(GOTEST) -short -v ./...

## bench: Запустить бенчмарки
bench:
	@echo "$(YELLOW)Running benchmarks...$(NC)"
	$(GOTEST) -bench=. -benchmem ./...

## lint: Запустить линтер
lint:
	@echo "$(YELLOW)Running linter...$(NC)"
	$(LINTER) run --timeout 5m ./...
	@echo "$(GREEN)✓ Lint passed$(NC)"

## fmt: Форматировать код
fmt:
	@echo "$(YELLOW)Formatting code...$(NC)"
	$(GOFMT) ./...
	@echo "$(GREEN)✓ Code formatted$(NC)"

## deps: Скачать зависимости
deps:
	@echo "$(YELLOW)Downloading dependencies...$(NC)"
	$(GOMOD) download
	$(GOMOD) verify
	@echo "$(GREEN)✓ Dependencies ready$(NC)"

## tidy: Очистить go.mod
tidy:
	@echo "$(YELLOW)Tidying go.mod...$(NC)"
	$(GOMOD) tidy
	@echo "$(GREEN)✓ go.mod tidied$(NC)"

## install: Установить утилиты
install:
	@echo "$(YELLOW)Installing tools...$(NC)"
	go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest
	@echo "$(GREEN)✓ Tools installed$(NC)"

## clean: Удалить артефакты сборки
clean:
	@echo "$(YELLOW)Cleaning...$(NC)"
	rm -rf $(BUILD_DIR)
	rm -f coverage.out coverage.html
	@echo "$(GREEN)✓ Clean complete$(NC)"

## docker-build: Собрать Docker образ
docker-build:
	@echo "$(GREEN)Building Docker image...$(NC)"
	docker build -t $(APP_NAME):$(VERSION) .
	docker tag $(APP_NAME):$(VERSION) $(APP_NAME):latest
	@echo "$(GREEN)✓ Docker image built$(NC)"

## docker-run: Запустить Docker контейнер
docker-run:
	@echo "$(GREEN)Running Docker container...$(NC)"
	docker run --rm -p 8080:8080 $(APP_NAME):latest

## ci: CI pipeline (lint + test + build)
ci: fmt lint test build
	@echo "$(GREEN)✓ CI pipeline complete$(NC)"

## version: Показать версию
version:
	@echo "Version: $(VERSION)"
	@echo "Commit:  $(COMMIT)"
	@echo "Built:   $(BUILD_TIME)"

## help: Показать помощь
help:
	@echo "$(GREEN)$(APP_NAME) Makefile$(NC)"
	@echo ""
	@echo "$(YELLOW)Available targets:$(NC)"
	@sed -n 's/^##//p' ${MAKEFILE_LIST} | column -t -s ':' | sed -e 's/^/ /'
```

## Условия в Makefile

```makefile
# Проверка переменной окружения
ifdef DEBUG
GOBUILD_FLAGS += -gcflags="all=-N -l"
endif

# Проверка ОС
ifeq ($(OS),Windows_NT)
    RM = del /Q
    BINARY_EXT = .exe
else
    RM = rm -f
    BINARY_EXT =
endif

build:
	go build -o bin/app$(BINARY_EXT) ./cmd/main.go
```

## Функции в Makefile

```makefile
# Найти все Go файлы
GO_FILES := $(shell find . -name '*.go' -not -path "./vendor/*")

# Найти все директории с тестами
TEST_DIRS := $(shell go list ./... | grep -v /vendor/)

# Проверка что команда существует
check-tool = $(shell command -v $(1) 2> /dev/null)

lint:
	@if [ -z "$(call check-tool,golangci-lint)" ]; then \
		echo "golangci-lint not found. Installing..."; \
		go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest; \
	fi
	golangci-lint run ./...
```

## Интеграция с Docker

```makefile
DOCKER_IMAGE := myapp
DOCKER_TAG := latest

docker-build:
	docker build -t $(DOCKER_IMAGE):$(DOCKER_TAG) .

docker-run:
	docker run --rm -p 8080:8080 \
		-e DB_HOST=postgres \
		-e REDIS_HOST=redis \
		$(DOCKER_IMAGE):$(DOCKER_TAG)

docker-push:
	docker push $(DOCKER_IMAGE):$(DOCKER_TAG)

docker-compose-up:
	docker-compose up -d

docker-compose-down:
	docker-compose down

docker-compose-logs:
	docker-compose logs -f
```

## Интеграция с CI/CD

```makefile
# .gitlab-ci.yml будет вызывать эти targets
.PHONY: ci-lint ci-test ci-build

ci-lint:
	@echo "Running linters in CI..."
	golangci-lint run --out-format=colored-line-number ./...

ci-test:
	@echo "Running tests in CI..."
	go test -v -race -coverprofile=coverage.out ./...
	go tool cover -func=coverage.out

ci-build:
	@echo "Building in CI..."
	CGO_ENABLED=0 go build -ldflags="-w -s" -o bin/app ./cmd/main.go
```

## Типичные ошибки

### Ошибка 1: Использование пробелов вместо TAB

```makefile
# ❌ Неправильно (пробелы)
build:
    go build ./...

# ✅ Правильно (TAB)
build:
	go build ./...
```

### Ошибка 2: Отсутствие .PHONY

```makefile
# ❌ Если существует файл test, команда не выполнится
test:
	go test ./...

# ✅ Правильно
.PHONY: test
test:
	go test ./...
```

### Ошибка 3: Неправильное использование переменных

```makefile
# ❌ Неправильно - переменная не раскроется в shell
APP_NAME=myapp
run:
	echo $APP_NAME  # Выведет пустую строку

# ✅ Правильно
APP_NAME=myapp
run:
	echo $(APP_NAME)  # Выведет: myapp
```

## Best Practices

1. ✅ **Используйте .PHONY** для всех команд
2. ✅ **Добавьте target help** с описанием команд
3. ✅ **Группируйте связанные переменные** в начале файла
4. ✅ **Используйте @** перед командами для подавления вывода команды
5. ✅ **Добавьте цвета** для важных сообщений
6. ✅ **Создайте target all** для полного pipeline
7. ✅ **Документируйте targets** комментариями
8. ❌ **Не делайте слишком сложную логику** в Makefile

## Альтернативы

- **Task (Taskfile)** — современная альтернатива на YAML
- **Just** — более простой синтаксис
- **Bash скрипты** — для сложной логики

Makefile идеален для простых задач, но для сложных pipeline лучше использовать специализированные инструменты.

## Вопросы с собеседований

**Вопрос:** Чем отличается `=` от `:=` в Makefile?

**Ответ:**
- `VAR = $(shell date)` — lazy evaluation, выполняется каждый раз при использовании
- `VAR := $(shell date)` — immediate evaluation, выполняется один раз при объявлении

**Вопрос:** Зачем нужен .PHONY?

**Ответ:** `.PHONY` указывает make, что target — это команда, а не файл. Без него, если в проекте существует файл с именем target (например, `test`), make не выполнит команду, считая что файл уже существует.

**Вопрос:** Как передать аргументы в Makefile target?

**Ответ:** Через переменные окружения:
```bash
make run PORT=8080
```

В Makefile:
```makefile
run:
	PORT=$(PORT) go run ./cmd/main.go
```

## Связанные темы

- [[Bash - Основы скриптинга]]
- [[Taskfile]]
- [[CI-CD - Основные концепции]]
- [[Docker - Основы]]
- [[GitLab CI-CD]]
