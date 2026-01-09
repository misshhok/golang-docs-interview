# GitLab CI-CD

GitLab CI/CD - встроенная в GitLab система непрерывной интеграции и доставки кода.

## Основы

### .gitlab-ci.yml

Конфигурация CI/CD pipeline в корне репозитория.

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - deploy

build-job:
  stage: build
  script:
    - echo "Building the app"
    - go build -o myapp .

test-job:
  stage: test
  script:
    - echo "Running tests"
    - go test ./...

deploy-job:
  stage: deploy
  script:
    - echo "Deploying to production"
  only:
    - main
```

**После push в Git:**
```bash
git push origin main
```

GitLab автоматически запустит pipeline.

### GitLab Runner

**GitLab Runner** - агент, который выполняет jobs.

**Типы runners:**
- **Shared runners** - общие для всех проектов (gitlab.com)
- **Group runners** - для группы проектов
- **Project runners** - для конкретного проекта

**Установка self-hosted runner:**
```bash
# Linux
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash
sudo apt-get install gitlab-runner

# Регистрация
sudo gitlab-runner register
```

## Структура .gitlab-ci.yml

### Stages

Этапы pipeline, выполняются последовательно.

```yaml
stages:
  - test
  - build
  - deploy

# Jobs в одном stage выполняются параллельно
unit-tests:
  stage: test
  script:
    - go test ./internal/...

integration-tests:
  stage: test
  script:
    - go test ./tests/integration/...

# build запустится только после успешного завершения test stage
build-app:
  stage: build
  script:
    - go build -o myapp .
```

### Jobs

**Job** - набор команд для выполнения.

```yaml
job-name:
  stage: test
  image: golang:1.21-alpine
  script:
    - go mod download
    - go test -v ./...
  artifacts:
    paths:
      - coverage.out
  cache:
    paths:
      - .cache/go-build
```

### Script

Команды для выполнения.

```yaml
test:
  script:
    # Одна команда на строку
    - go mod download
    - go test ./...

    # Многострочная команда
    - |
      if [ "$CI_COMMIT_BRANCH" == "main" ]; then
        echo "Main branch"
      fi

    # Команды с pipe
    - go test ./... | grep -v "no test files"
```

### Image

Docker image для job.

```yaml
# Глобальный image для всех jobs
image: golang:1.21-alpine

jobs:
  test:
    # Переопределить для конкретного job
    image: golang:1.21
    script:
      - go test ./...

  build:
    # Использует глобальный golang:1.21-alpine
    script:
      - go build -o myapp .
```

### Before_script и After_script

Выполняются до/после основного script.

```yaml
before_script:
  - echo "Starting job"
  - go mod download

script:
  - go test ./...

after_script:
  - echo "Job finished"
  - rm -rf tmp/
```

## Variables

### Predefined Variables

GitLab предоставляет множество переменных.

```yaml
test:
  script:
    - echo "Commit SHA: $CI_COMMIT_SHA"
    - echo "Branch: $CI_COMMIT_BRANCH"
    - echo "Project: $CI_PROJECT_NAME"
    - echo "Pipeline ID: $CI_PIPELINE_ID"
```

**Полезные переменные:**
- `CI_COMMIT_SHA` - commit hash
- `CI_COMMIT_SHORT_SHA` - короткий hash (8 символов)
- `CI_COMMIT_BRANCH` - имя ветки
- `CI_COMMIT_TAG` - имя тега
- `CI_PROJECT_NAME` - имя проекта
- `CI_PROJECT_URL` - URL проекта
- `CI_PIPELINE_ID` - ID pipeline
- `CI_JOB_ID` - ID job
- `CI_REGISTRY` - GitLab Container Registry URL
- `CI_REGISTRY_USER` - пользователь для registry
- `CI_REGISTRY_PASSWORD` - пароль для registry

### Custom Variables

```yaml
variables:
  DOCKER_IMAGE: "registry.gitlab.com/myorg/myapp"
  GO_VERSION: "1.21"
  DATABASE_URL: "postgres://localhost:5432/test"

test:
  script:
    - echo "Building image: $DOCKER_IMAGE"
    - go version  # Использует GO_VERSION из image
```

### Project Variables

**Settings → CI/CD → Variables**

Создать переменные:
- `DATABASE_PASSWORD` = `secret123`
- `API_KEY` = `abc123xyz`

Использование:
```yaml
deploy:
  script:
    - kubectl create secret generic db-secret --from-literal=password=$DATABASE_PASSWORD
    - curl -H "Authorization: Bearer $API_KEY" https://api.example.com
```

**Типы переменных:**
- **Variable** - обычная переменная
- **File** - сохраняется как файл, `$VAR` содержит путь к файлу
- **Protected** - доступна только в protected branches (main, release/*)
- **Masked** - скрывается в логах

## Artifacts

Файлы, создаваемые job и доступные в следующих jobs/stages.

```yaml
build:
  stage: build
  script:
    - go build -o myapp .
  artifacts:
    paths:
      - myapp
    expire_in: 1 week

test:
  stage: test
  script:
    - ./myapp --version  # myapp из artifacts предыдущего stage
```

### Скачать artifacts

```bash
# Из UI: Job → Browse → Download
# Или через API
curl --header "PRIVATE-TOKEN: $TOKEN" \
  "https://gitlab.com/api/v4/projects/123/jobs/456/artifacts" \
  -o artifacts.zip
```

### Reports

Специальные artifacts для отчётов.

```yaml
test:
  script:
    - go test -coverprofile=coverage.out ./...
    - go tool cover -func=coverage.out > coverage.txt
  artifacts:
    reports:
      # Code coverage
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml

      # JUnit tests
      junit: report.xml

    # Обычные artifacts
    paths:
      - coverage.out
      - coverage.txt

  # Отображение coverage в MR
  coverage: '/total:.*?(\d+\.\d+)%/'
```

## Cache

Кэширование зависимостей между pipeline runs.

```yaml
variables:
  CACHE_KEY: ${CI_COMMIT_REF_SLUG}

test:
  cache:
    key: $CACHE_KEY
    paths:
      - .cache/go-build
      - go/pkg/mod
  script:
    - go mod download
    - go test ./...

build:
  cache:
    key: $CACHE_KEY
    paths:
      - .cache/go-build
      - go/pkg/mod
  script:
    - go build -o myapp .
```

**Artifacts vs Cache:**
- **Artifacts** - результаты job, передаются между stages
- **Cache** - оптимизация, ускоряет повторные сборки

## Rules

Условия для запуска jobs.

```yaml
deploy:production:
  script:
    - kubectl apply -f k8s/
  rules:
    # Запускать только на main
    - if: $CI_COMMIT_BRANCH == "main"

    # Или при наличии тега
    - if: $CI_COMMIT_TAG

deploy:staging:
  script:
    - kubectl apply -f k8s/ -n staging
  rules:
    # Запускать на develop
    - if: $CI_COMMIT_BRANCH == "develop"

    # Или на merge requests
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
```

**Операторы:**
- `==` - равно
- `!=` - не равно
- `=~` - regex match
- `!~` - regex not match

```yaml
rules:
  # Только на feature/* ветках
  - if: $CI_COMMIT_BRANCH =~ /^feature\/.*/

  # Исключить draft MR
  - if: $CI_MERGE_REQUEST_TITLE =~ /^Draft:/
    when: never

  # Запускать вручную на main
  - if: $CI_COMMIT_BRANCH == "main"
    when: manual
```

## Only / Except (deprecated, используйте rules)

```yaml
# Only - запускать только если
deploy:
  script:
    - kubectl apply -f k8s/
  only:
    - main
    - tags

# Except - запускать кроме
test:
  script:
    - go test ./...
  except:
    - main
```

## Environments

Определение окружений для деплоя.

```yaml
deploy:staging:
  stage: deploy
  script:
    - kubectl apply -f k8s/ -n staging
  environment:
    name: staging
    url: https://staging.example.com
    on_stop: stop:staging  # Job для остановки окружения
  rules:
    - if: $CI_COMMIT_BRANCH == "develop"

deploy:production:
  stage: deploy
  script:
    - kubectl apply -f k8s/ -n production
  environment:
    name: production
    url: https://example.com
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
  when: manual  # Ручное подтверждение

stop:staging:
  stage: deploy
  script:
    - kubectl delete namespace staging
  environment:
    name: staging
    action: stop
  rules:
    - if: $CI_COMMIT_BRANCH == "develop"
  when: manual
```

**UI:** Deployments → Environments → список окружений с URL

## Services

Docker containers, запускаемые вместе с job (databases, etc).

```yaml
test:
  image: golang:1.21
  services:
    # PostgreSQL
    - name: postgres:15
      alias: postgres
    # Redis
    - name: redis:7-alpine
      alias: redis
  variables:
    POSTGRES_DB: test
    POSTGRES_USER: postgres
    POSTGRES_PASSWORD: secret
  script:
    # Доступ по alias
    - psql -h postgres -U postgres -d test -c "CREATE TABLE users(id INT);"
    - go test ./tests/integration/...
```

## Includes

Переиспользование конфигурации.

**templates/build.yml:**
```yaml
.build_template:
  image: golang:1.21-alpine
  script:
    - go mod download
    - go build -o myapp .
  artifacts:
    paths:
      - myapp
```

**.gitlab-ci.yml:**
```yaml
include:
  - local: 'templates/build.yml'

stages:
  - build
  - test

build:
  extends: .build_template
  stage: build
```

**Include из remote:**
```yaml
include:
  # Другой проект
  - project: 'myorg/ci-templates'
    file: '/templates/build.yml'

  # Remote URL
  - remote: 'https://example.com/ci-template.yml'
```

## Docker-in-Docker

Сборка Docker images в GitLab CI.

```yaml
build:
  stage: build
  image: docker:latest
  services:
    - docker:dind  # Docker-in-Docker
  variables:
    DOCKER_IMAGE: registry.gitlab.com/myorg/myapp
    DOCKER_TLS_CERTDIR: "/certs"
  before_script:
    # Login в GitLab Container Registry
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    # Собрать image
    - docker build -t $DOCKER_IMAGE:$CI_COMMIT_SHORT_SHA .
    - docker tag $DOCKER_IMAGE:$CI_COMMIT_SHORT_SHA $DOCKER_IMAGE:latest

    # Push image
    - docker push $DOCKER_IMAGE:$CI_COMMIT_SHORT_SHA
    - docker push $DOCKER_IMAGE:latest
```

## Production Pipeline для Go приложения

```yaml
# .gitlab-ci.yml
stages:
  - test
  - build
  - deploy

variables:
  DOCKER_IMAGE: $CI_REGISTRY_IMAGE
  GO_VERSION: "1.21"

# ====================
# Шаблоны
# ====================
.go_template:
  image: golang:${GO_VERSION}-alpine
  before_script:
    - apk add --no-cache git
    - go mod download

# ====================
# Test Stage
# ====================
lint:
  extends: .go_template
  stage: test
  script:
    - go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest
    - golangci-lint run --timeout 5m
  allow_failure: false

unit-tests:
  extends: .go_template
  stage: test
  script:
    - go test -v -race -coverprofile=coverage.out ./...
    - go tool cover -func=coverage.out
  coverage: '/total:.*?(\d+\.\d+)%/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
    paths:
      - coverage.out
    expire_in: 30 days

integration-tests:
  extends: .go_template
  stage: test
  services:
    - postgres:15
    - redis:7-alpine
  variables:
    POSTGRES_DB: test
    POSTGRES_USER: postgres
    POSTGRES_PASSWORD: secret
    DATABASE_URL: "postgres://postgres:secret@postgres:5432/test"
    REDIS_URL: "redis://redis:6379"
  script:
    - go test -v ./tests/integration/...
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
    - if: $CI_MERGE_REQUEST_ID

# ====================
# Build Stage
# ====================
build:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    # Собрать multi-stage Docker image
    - |
      docker build \
        --cache-from $DOCKER_IMAGE:latest \
        --tag $DOCKER_IMAGE:$CI_COMMIT_SHORT_SHA \
        --tag $DOCKER_IMAGE:latest \
        --build-arg GO_VERSION=$GO_VERSION \
        .

    # Push images
    - docker push $DOCKER_IMAGE:$CI_COMMIT_SHORT_SHA
    - docker push $DOCKER_IMAGE:latest

    # Для tags - также push с версией
    - |
      if [ -n "$CI_COMMIT_TAG" ]; then
        docker tag $DOCKER_IMAGE:$CI_COMMIT_SHORT_SHA $DOCKER_IMAGE:$CI_COMMIT_TAG
        docker push $DOCKER_IMAGE:$CI_COMMIT_TAG
      fi
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
    - if: $CI_COMMIT_BRANCH == "develop"
    - if: $CI_COMMIT_TAG

# ====================
# Deploy Stage
# ====================
deploy:staging:
  stage: deploy
  image: bitnami/kubectl:latest
  before_script:
    - mkdir -p ~/.kube
    - echo "$KUBE_CONFIG_STAGING" | base64 -d > ~/.kube/config
  script:
    - kubectl config use-context staging

    # Обновить image
    - kubectl set image deployment/myapp myapp=$DOCKER_IMAGE:$CI_COMMIT_SHORT_SHA -n staging

    # Дождаться rollout
    - kubectl rollout status deployment/myapp -n staging --timeout=5m

    # Проверить health
    - kubectl get pods -n staging -l app=myapp
  environment:
    name: staging
    url: https://staging.example.com
    on_stop: stop:staging
  rules:
    - if: $CI_COMMIT_BRANCH == "develop"

deploy:production:
  stage: deploy
  image: bitnami/kubectl:latest
  before_script:
    - mkdir -p ~/.kube
    - echo "$KUBE_CONFIG_PROD" | base64 -d > ~/.kube/config
  script:
    - kubectl config use-context production

    # Blue-Green deployment
    - |
      # Создать green deployment
      kubectl apply -f k8s/deployment-green.yaml -n production

      # Дождаться готовности
      kubectl rollout status deployment/myapp-green -n production --timeout=5m

      # Переключить Service на green
      kubectl patch service myapp -n production -p '{"spec":{"selector":{"version":"green"}}}'

      # Удалить blue deployment после проверки
      sleep 60
      kubectl delete deployment myapp-blue -n production || true

      # Переименовать green в blue для следующего деплоя
      kubectl label deployment myapp-green version=blue -n production --overwrite
  environment:
    name: production
    url: https://example.com
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
  when: manual  # Требует ручного подтверждения

stop:staging:
  stage: deploy
  image: bitnami/kubectl:latest
  before_script:
    - mkdir -p ~/.kube
    - echo "$KUBE_CONFIG_STAGING" | base64 -d > ~/.kube/config
  script:
    - kubectl scale deployment/myapp --replicas=0 -n staging
  environment:
    name: staging
    action: stop
  rules:
    - if: $CI_COMMIT_BRANCH == "develop"
  when: manual
```

## Best Practices

### 1. Используйте extends для DRY

```yaml
.go_base:
  image: golang:1.21-alpine
  before_script:
    - go mod download

test:
  extends: .go_base
  script:
    - go test ./...

build:
  extends: .go_base
  script:
    - go build -o myapp .
```

### 2. Cache зависимости

```yaml
variables:
  GOPATH: $CI_PROJECT_DIR/.go

cache:
  paths:
    - .go/pkg/mod/
```

### 3. Fail fast

```yaml
stages:
  - lint       # Быстро
  - test       # Средне
  - build      # Медленно
  - deploy     # Долго
```

### 4. Protected variables для production

**Settings → CI/CD → Variables → Protected**

### 5. Manual jobs для production

```yaml
deploy:production:
  when: manual
```

### 6. Retry при временных ошибках

```yaml
test:
  script:
    - go test ./...
  retry:
    max: 2
    when:
      - runner_system_failure
      - stuck_or_timeout_failure
```

## Связанные темы

- [[CI-CD - Основные концепции]]
- [[GitHub Actions]]
- [[Docker - Основы]]
- [[Kubernetes - Основы]]
- [[Git - Основные команды]]
