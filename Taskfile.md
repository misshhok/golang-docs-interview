# Taskfile

Task (taskfile.dev) — это современная альтернатива Make, написанная на Go. Использует YAML синтаксис вместо Makefile и решает многие проблемы традиционного Make.

## Зачем нужен Task

1. **Простой синтаксис** — YAML вместо сложного синтаксиса Makefile
2. **Кроссплатформенность** — одинаково работает на Windows, Linux, macOS
3. **Встроенные возможности** — работа с переменными, условия, циклы
4. **Современный** — создан специально для современных workflow
5. **Нет проблем с TAB** — в YAML используются пробелы

## Установка

### macOS (Homebrew)

```bash
brew install go-task/tap/go-task
```

### Linux (snap)

```bash
snap install task --classic
```

### Go install

```bash
go install github.com/go-task/task/v3/cmd/task@latest
```

### Проверка установки

```bash
task --version
```

## Базовый Taskfile.yml

### Структура файла

```yaml
# Taskfile.yml
version: '3'

tasks:
  hello:
    desc: "Приветствие"
    cmds:
      - echo "Hello, World!"

  build:
    desc: "Собрать приложение"
    cmds:
      - go build -o bin/app ./cmd/main.go
```

Запуск:
```bash
task hello
task build
```

## Переменные

### Глобальные переменные

```yaml
version: '3'

vars:
  APP_NAME: myapp
  BUILD_DIR: ./bin
  VERSION:
    sh: git describe --tags --always

tasks:
  build:
    desc: "Собрать {{.APP_NAME}}"
    cmds:
      - go build -o {{.BUILD_DIR}}/{{.APP_NAME}} ./cmd/main.go
      - echo "Built version {{.VERSION}}"
```

### Переменные в task

```yaml
version: '3'

tasks:
  greet:
    desc: "Поприветствовать пользователя"
    vars:
      NAME: Gopher
      MESSAGE: Hello
    cmds:
      - echo "{{.MESSAGE}}, {{.NAME}}!"
```

### Переменные окружения

```yaml
version: '3'

tasks:
  run:
    desc: "Запустить приложение"
    env:
      DB_HOST: localhost
      DB_PORT: 5432
      API_PORT: 8080
    cmds:
      - go run ./cmd/main.go
```

### Переменные из .env файла

```yaml
version: '3'

dotenv: ['.env']

tasks:
  run:
    desc: "Запустить с конфигом из .env"
    cmds:
      - go run ./cmd/main.go
```

## Зависимости между tasks

### deps: зависимости выполняются параллельно

```yaml
version: '3'

tasks:
  build:
    desc: "Собрать приложение"
    deps: [fmt, lint, test]  # Выполнятся параллельно
    cmds:
      - go build -o bin/app ./cmd/main.go

  fmt:
    desc: "Форматировать код"
    cmds:
      - go fmt ./...

  lint:
    desc: "Проверить линтером"
    cmds:
      - golangci-lint run ./...

  test:
    desc: "Запустить тесты"
    cmds:
      - go test -v ./...
```

### cmds: команды выполняются последовательно

```yaml
version: '3'

tasks:
  deploy:
    desc: "Деплой приложения"
    cmds:
      - task: build  # Сначала билд
      - task: test   # Потом тесты
      - ./scripts/deploy.sh  # Потом деплой
```

## Условия

### Условное выполнение команд

```yaml
version: '3'

tasks:
  build:
    desc: "Собрать приложение"
    cmds:
      - cmd: echo "Building in debug mode"
        platforms: [darwin, linux]  # Только на macOS и Linux
      - cmd: echo "Building production"
        platforms: [windows]  # Только на Windows

  test:
    desc: "Запустить тесты"
    preconditions:
      - test -f go.mod  # Проверка что файл существует
      - sh: which go
        msg: "Go is not installed"
    cmds:
      - go test ./...
```

### status: пропустить если не нужно

```yaml
version: '3'

tasks:
  build:
    desc: "Собрать приложение"
    sources:  # Исходные файлы
      - '**/*.go'
      - 'go.mod'
      - 'go.sum'
    generates:  # Результат
      - 'bin/app'
    cmds:
      - go build -o bin/app ./cmd/main.go
```

Task пересоберет только если исходники новее результата.

## Полный Taskfile для Go проекта

```yaml
version: '3'

# Глобальные переменные
vars:
  APP_NAME: myapp
  BUILD_DIR: ./bin
  MAIN_PATH: ./cmd/main.go
  VERSION:
    sh: git describe --tags --always --dirty
  COMMIT:
    sh: git rev-parse --short HEAD
  BUILD_TIME:
    sh: date +%Y-%m-%d_%H:%M:%S

# Загрузка .env файла
dotenv: ['.env']

# Переменные окружения для всех tasks
env:
  CGO_ENABLED: 0

tasks:
  # Задача по умолчанию
  default:
    desc: "Показать доступные команды"
    cmds:
      - task --list

  # Установка зависимостей
  deps:
    desc: "Скачать зависимости"
    cmds:
      - go mod download
      - go mod verify

  # Установка инструментов
  install-tools:
    desc: "Установить инструменты разработки"
    cmds:
      - go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest
      - go install gotest.tools/gotestsum@latest
    status:
      - which golangci-lint
      - which gotestsum

  # Форматирование кода
  fmt:
    desc: "Форматировать код"
    cmds:
      - go fmt ./...

  # Линтинг
  lint:
    desc: "Запустить линтер"
    deps: [fmt]
    cmds:
      - golangci-lint run --timeout 5m ./...

  # Тесты
  test:
    desc: "Запустить тесты"
    deps: [fmt]
    cmds:
      - go test -v -race -timeout 30s ./...

  test-coverage:
    desc: "Тесты с покрытием"
    cmds:
      - go test -v -race -coverprofile=coverage.out -covermode=atomic ./...
      - go tool cover -html=coverage.out -o coverage.html
      - echo "Coverage report: coverage.html"

  test-short:
    desc: "Быстрые тесты"
    cmds:
      - go test -short -v ./...

  # Бенчмарки
  bench:
    desc: "Запустить бенчмарки"
    cmds:
      - go test -bench=. -benchmem ./...

  # Сборка
  build:
    desc: "Собрать приложение"
    deps: [test]
    sources:
      - '**/*.go'
      - 'go.mod'
      - 'go.sum'
    generates:
      - '{{.BUILD_DIR}}/{{.APP_NAME}}'
    cmds:
      - mkdir -p {{.BUILD_DIR}}
      - |
        go build -ldflags " \
          -X main.Version={{.VERSION}} \
          -X main.Commit={{.COMMIT}} \
          -X main.BuildTime={{.BUILD_TIME}}" \
          -o {{.BUILD_DIR}}/{{.APP_NAME}} {{.MAIN_PATH}}
      - echo "✓ Build complete: {{.BUILD_DIR}}/{{.APP_NAME}}"

  build-linux:
    desc: "Собрать для Linux"
    env:
      GOOS: linux
      GOARCH: amd64
    cmds:
      - mkdir -p {{.BUILD_DIR}}
      - go build -o {{.BUILD_DIR}}/{{.APP_NAME}}-linux {{.MAIN_PATH}}

  build-all:
    desc: "Собрать для всех платформ"
    cmds:
      - task: build-platform
        vars: {GOOS: linux, GOARCH: amd64}
      - task: build-platform
        vars: {GOOS: darwin, GOARCH: amd64}
      - task: build-platform
        vars: {GOOS: darwin, GOARCH: arm64}
      - task: build-platform
        vars: {GOOS: windows, GOARCH: amd64}

  build-platform:
    internal: true  # Не показывать в списке
    env:
      GOOS: '{{.GOOS}}'
      GOARCH: '{{.GOARCH}}'
    cmds:
      - mkdir -p {{.BUILD_DIR}}
      - go build -o {{.BUILD_DIR}}/{{.APP_NAME}}-{{.GOOS}}-{{.GOARCH}} {{.MAIN_PATH}}

  # Запуск
  run:
    desc: "Запустить приложение"
    cmds:
      - go run {{.MAIN_PATH}}

  run-dev:
    desc: "Запустить в dev режиме с hot reload"
    deps: [install-air]
    cmds:
      - air

  install-air:
    internal: true
    cmds:
      - go install github.com/cosmtrek/air@latest
    status:
      - which air

  # Docker
  docker-build:
    desc: "Собрать Docker образ"
    cmds:
      - docker build -t {{.APP_NAME}}:{{.VERSION}} .
      - docker tag {{.APP_NAME}}:{{.VERSION}} {{.APP_NAME}}:latest

  docker-run:
    desc: "Запустить Docker контейнер"
    cmds:
      - docker run --rm -p 8080:8080 {{.APP_NAME}}:latest

  docker-compose-up:
    desc: "Запустить docker-compose"
    cmds:
      - docker-compose up -d

  docker-compose-down:
    desc: "Остановить docker-compose"
    cmds:
      - docker-compose down

  docker-compose-logs:
    desc: "Логи docker-compose"
    cmds:
      - docker-compose logs -f

  # Очистка
  clean:
    desc: "Удалить артефакты"
    cmds:
      - rm -rf {{.BUILD_DIR}}
      - rm -f coverage.out coverage.html

  # CI pipeline
  ci:
    desc: "CI pipeline"
    cmds:
      - task: fmt
      - task: lint
      - task: test
      - task: build
      - echo "✓ CI pipeline complete"

  # Версия
  version:
    desc: "Показать версию"
    cmds:
      - echo "Version {{.VERSION}}"
      - echo "Commit  {{.COMMIT}}"
      - echo "Built   {{.BUILD_TIME}}"
```

## Продвинутые возможности

### Включение других Taskfile

```yaml
# Taskfile.yml
version: '3'

includes:
  docker: ./docker/Taskfile.yml
  db: ./database/Taskfile.yml

tasks:
  build:
    cmds:
      - go build ./...
```

Запуск:
```bash
task docker:build  # Запустит task build из docker/Taskfile.yml
task db:migrate    # Запустит task migrate из database/Taskfile.yml
```

### Циклы (for)

```yaml
version: '3'

tasks:
  test-all:
    desc: "Тестировать все сервисы"
    vars:
      SERVICES: "api worker scheduler"
    cmds:
      - for: {var: SERVICES}
        cmd: go test ./{{.ITEM}}/...
```

### Интерактивный ввод

```yaml
version: '3'

tasks:
  deploy:
    desc: "Деплой на сервер"
    prompt: "Вы уверены что хотите задеплоить на production?"
    cmds:
      - ./scripts/deploy.sh
```

### Silent mode

```yaml
version: '3'

tasks:
  build:
    desc: "Тихая сборка"
    silent: true  # Не выводить команды
    cmds:
      - go build -o bin/app ./cmd/main.go
```

## Task vs Make

### Преимущества Task

✅ **Простой синтаксис** — YAML вместо сложного Makefile синтаксиса
✅ **Кроссплатформенность** — работает одинаково на всех ОС
✅ **Нет TAB проблем** — YAML использует пробелы
✅ **Встроенные функции** — не нужен bash для простых операций
✅ **Параллельное выполнение** — deps выполняются параллельно
✅ **Checksum проверка** — умное определение изменений

### Когда использовать Make

- Проект уже использует Makefile
- Команда привыкла к Make
- Нужна совместимость со старыми системами
- Очень простой проект

### Когда использовать Task

- Новый проект
- Нужна кроссплатформенность
- Команда предпочитает YAML
- Нужны современные возможности (параллелизм, conditions, loops)

## Примеры из реальных проектов

### Backend API проект

```yaml
version: '3'

dotenv: ['.env']

vars:
  APP_NAME: api-server
  DB_CONTAINER: postgres-dev

tasks:
  db-start:
    desc: "Запустить PostgreSQL в Docker"
    cmds:
      - |
        docker run -d \
          --name {{.DB_CONTAINER}} \
          -e POSTGRES_PASSWORD=secret \
          -e POSTGRES_DB=myapp \
          -p 5432:5432 \
          postgres:15
    status:
      - docker ps | grep {{.DB_CONTAINER}}

  db-stop:
    desc: "Остановить PostgreSQL"
    cmds:
      - docker stop {{.DB_CONTAINER}}
      - docker rm {{.DB_CONTAINER}}

  migrate:
    desc: "Применить миграции"
    deps: [db-start]
    cmds:
      - go run ./cmd/migrate/main.go up

  seed:
    desc: "Наполнить БД тестовыми данными"
    deps: [migrate]
    cmds:
      - go run ./cmd/seed/main.go

  dev:
    desc: "Запустить dev сервер"
    deps: [db-start, migrate]
    cmds:
      - air
```

### Микросервисы

```yaml
version: '3'

tasks:
  services:up:
    desc: "Запустить все сервисы"
    cmds:
      - task: service:up
        vars: {SERVICE: api}
      - task: service:up
        vars: {SERVICE: worker}
      - task: service:up
        vars: {SERVICE: scheduler}

  service:up:
    internal: true
    dir: './services/{{.SERVICE}}'
    cmds:
      - go run ./cmd/main.go

  test:all:
    desc: "Тестировать все сервисы"
    cmds:
      - task: test:service
        vars: {SERVICE: api}
      - task: test:service
        vars: {SERVICE: worker}
      - task: test:service
        vars: {SERVICE: scheduler}

  test:service:
    internal: true
    dir: './services/{{.SERVICE}}'
    cmds:
      - go test -v ./...
```

## Best Practices

1. ✅ **Добавляйте desc** к каждому task для документации
2. ✅ **Используйте deps** для зависимостей
3. ✅ **Группируйте tasks** через двоеточие: `docker:build`, `db:migrate`
4. ✅ **Используйте dotenv** для локальной конфигурации
5. ✅ **Добавьте task default** для вывода help
6. ✅ **Используйте internal** для вспомогательных tasks
7. ✅ **Документируйте переменные** комментариями
8. ❌ **Не делайте слишком сложную логику** в Taskfile

## Вопросы с собеседований

**Вопрос:** Чем Task отличается от Make?

**Ответ:**
- Task использует YAML синтаксис вместо сложного Makefile
- Task кроссплатформенный из коробки
- Task использует пробелы вместо TAB
- Task имеет встроенную поддержку параллелизма (deps)
- Task написан на Go и легко устанавливается через go install

**Вопрос:** Как Task определяет нужно ли пересобирать?

**Ответ:** Task использует несколько механизмов:
- `sources` и `generates` — сравнивает время модификации файлов
- `status` — выполняет команду проверки
- Checksum файлов для точного определения изменений
- Можно комбинировать эти подходы

**Вопрос:** Можно ли использовать Task в CI/CD?

**Ответ:** Да! Task отлично работает в CI/CD:
```yaml
# .gitlab-ci.yml
test:
  script:
    - task ci
```

Task обеспечивает консистентность между локальной разработкой и CI/CD.

## Связанные темы

- [[Makefile]]
- [[Bash - Основы скриптинга]]
- [[CI-CD - Основные концепции]]
- [[Docker - Docker Compose]]
- [[GitLab CI-CD]]
