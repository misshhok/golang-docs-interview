# GitHub Actions

GitHub Actions - это встроенная платформа CI/CD от GitHub для автоматизации процессов разработки: сборки, тестирования, деплоя и других workflow.

## Основные концепции

### Workflow

Workflow - это автоматизированный процесс, определенный в YAML-файле в директории `.github/workflows/`. Один репозиторий может содержать множество workflow для разных задач.

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run tests
        run: go test ./...
```

### Events (Триггеры)

Events определяют, когда запускается workflow.

**Популярные события:**
- `push` - при пуше в репозиторий
- `pull_request` - при создании/обновлении PR
- `schedule` - по расписанию (cron)
- `workflow_dispatch` - ручной запуск
- `release` - при создании релиза

```yaml
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 0 * * *'  # каждый день в полночь
```

### Jobs

Job - это набор steps, выполняющихся на одном runner. Jobs могут выполняться параллельно или последовательно.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Build
        run: go build -o app

  test:
    runs-on: ubuntu-latest
    needs: build  # запустится после build
    steps:
      - name: Test
        run: go test ./...
```

### Steps

Step - это отдельная задача в job. Может быть либо shell-командой (`run`), либо action (`uses`).

```yaml
steps:
  # Использование готового action
  - uses: actions/checkout@v3

  # Выполнение shell-команды
  - name: Install dependencies
    run: go mod download

  # С условием
  - name: Deploy
    if: github.ref == 'refs/heads/main'
    run: ./deploy.sh
```

## Runners

Runner - это сервер, на котором выполняются jobs.

**GitHub-hosted runners:**
- `ubuntu-latest`
- `windows-latest`
- `macos-latest`

**Self-hosted runners:**
Можно настроить свои собственные серверы для выполнения workflow.

```yaml
jobs:
  build:
    runs-on: self-hosted  # использовать свой runner
```

## Практический пример: Go CI/CD Pipeline

```yaml
# .github/workflows/go-ci.yml
name: Go CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  GO_VERSION: '1.21'

jobs:
  lint:
    name: Lint
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Setup Go
        uses: actions/setup-go@v4
        with:
          go-version: ${{ env.GO_VERSION }}

      - name: Run golangci-lint
        uses: golangci/golangci-lint-action@v3
        with:
          version: latest

  test:
    name: Test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Setup Go
        uses: actions/setup-go@v4
        with:
          go-version: ${{ env.GO_VERSION }}

      - name: Run tests
        run: go test -v -race -coverprofile=coverage.out ./...

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage.out

  build:
    name: Build
    runs-on: ubuntu-latest
    needs: [lint, test]  # запустится после успешного lint и test
    steps:
      - uses: actions/checkout@v3

      - name: Setup Go
        uses: actions/setup-go@v4
        with:
          go-version: ${{ env.GO_VERSION }}

      - name: Build binary
        run: go build -v -o app ./cmd/server

      - name: Upload artifact
        uses: actions/upload-artifact@v3
        with:
          name: app-binary
          path: app

  deploy:
    name: Deploy
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'  # только для main ветки
    steps:
      - name: Download artifact
        uses: actions/download-artifact@v3
        with:
          name: app-binary

      - name: Deploy to production
        run: |
          echo "Deploying to production..."
          # SSH команды для деплоя
```

## GitHub Actions Marketplace

Marketplace содержит тысячи готовых actions от сообщества.

**Популярные actions:**

```yaml
# Checkout кода
- uses: actions/checkout@v3

# Setup Go
- uses: actions/setup-go@v4
  with:
    go-version: '1.21'

# Setup Docker Buildx
- uses: docker/setup-buildx-action@v2

# Login в Docker Hub
- uses: docker/login-action@v2
  with:
    username: ${{ secrets.DOCKERHUB_USERNAME }}
    password: ${{ secrets.DOCKERHUB_TOKEN }}

# Build и push Docker image
- uses: docker/build-push-action@v4
  with:
    push: true
    tags: user/app:latest
```

## Secrets

Secrets - это зашифрованные переменные для хранения чувствительных данных (токены, пароли).

**Создание secret:**
Settings → Secrets and variables → Actions → New repository secret

**Использование:**

```yaml
steps:
  - name: Deploy
    env:
      API_KEY: ${{ secrets.API_KEY }}
      DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
    run: ./deploy.sh
```

## Matrix Strategy

Matrix позволяет запускать job с разными конфигурациями параллельно.

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        go: ['1.20', '1.21']
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-go@v4
        with:
          go-version: ${{ matrix.go }}
      - run: go test ./...
```

Это создаст 6 jobs (3 OS × 2 Go версии).

## Artifacts и Caching

### Artifacts

Сохранение файлов между jobs или для скачивания.

```yaml
- name: Upload artifact
  uses: actions/upload-artifact@v3
  with:
    name: coverage-report
    path: coverage.html
    retention-days: 30
```

### Caching

Кэширование зависимостей для ускорения сборки.

```yaml
- name: Cache Go modules
  uses: actions/cache@v3
  with:
    path: ~/go/pkg/mod
    key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}
    restore-keys: |
      ${{ runner.os }}-go-
```

## Условное выполнение

```yaml
steps:
  - name: Deploy to staging
    if: github.ref == 'refs/heads/develop'
    run: ./deploy-staging.sh

  - name: Deploy to production
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    run: ./deploy-production.sh

  - name: Comment on PR
    if: github.event_name == 'pull_request'
    run: gh pr comment ${{ github.event.number }} --body "Tests passed!"
```

## Environments

Environments позволяют настроить правила деплоя (approval, secrets).

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production  # требует approval
    steps:
      - name: Deploy
        run: ./deploy.sh
```

## Best Practices

1. ✅ **Используйте конкретные версии actions** (`actions/checkout@v3`, а не `@latest`)
2. ✅ **Кэшируйте зависимости** для ускорения workflow
3. ✅ **Разделяйте jobs** на независимые задачи (lint, test, build)
4. ✅ **Используйте matrix** для тестирования на разных платформах
5. ✅ **Храните secrets** в GitHub Secrets, не в коде
6. ✅ **Добавьте условия** для деплоя только из определенных веток
7. ❌ **Не храните токены** в workflow файлах
8. ❌ **Не игнорируйте ошибки** с `continue-on-error` без причины

## Типичные ошибки

### Ошибка 1: Забыли checkout

```yaml
# ❌ Неправильно
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: go test ./...  # ошибка: нет кода

# ✅ Правильно
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: go test ./...
```

### Ошибка 2: Некорректный синтаксис условий

```yaml
# ❌ Неправильно
if: ${{ github.ref }} == 'refs/heads/main'

# ✅ Правильно
if: github.ref == 'refs/heads/main'
```

## Вопросы с собеседований

**Вопрос:** В чем разница между GitHub Actions и GitLab CI/CD?

**Ответ:**
- GitHub Actions интегрирован с GitHub, GitLab CI/CD с GitLab
- GitHub Actions использует Marketplace для готовых actions
- GitLab CI/CD использует `.gitlab-ci.yml`, GitHub Actions `.github/workflows/*.yml`
- GitHub Actions поддерживает matrix strategy из коробки
- GitLab CI/CD имеет встроенный Container Registry

**Вопрос:** Как передать данные между jobs?

**Ответ:** Через artifacts:
```yaml
jobs:
  build:
    steps:
      - run: go build -o app
      - uses: actions/upload-artifact@v3
        with:
          name: binary
          path: app

  test:
    needs: build
    steps:
      - uses: actions/download-artifact@v3
        with:
          name: binary
```

**Вопрос:** Как запустить workflow вручную?

**Ответ:** Использовать событие `workflow_dispatch`:
```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy'
        required: true
        default: 'staging'
```

## Связанные темы

- [[CI-CD - Основные концепции]]
- [[GitLab CI-CD]]
- [[Docker - Основы]]
- [[Docker - Docker Compose]]
