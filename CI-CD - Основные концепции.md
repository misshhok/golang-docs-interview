# CI-CD - Основные концепции

CI/CD (Continuous Integration / Continuous Delivery) - практики автоматизации разработки, тестирования и развёртывания приложений.

## Что такое CI/CD?

### Continuous Integration (CI)

**Непрерывная интеграция** - автоматическая сборка и тестирование кода при каждом коммите.

**Процесс:**
1. Разработчик делает commit в Git
2. CI система автоматически:
   - Скачивает код
   - Собирает приложение
   - Запускает тесты
   - Проверяет code quality (linters)
3. Разработчик получает feedback: успешно ✅ или ошибка ❌

**Цели:**
- Раннее обнаружение багов
- Быстрый feedback для разработчиков
- Уверенность что код работает

### Continuous Delivery (CD)

**Непрерывная доставка** - автоматическая подготовка кода к релизу.

**Процесс:**
1. После успешного CI
2. CD система автоматически:
   - Создаёт Docker image
   - Деплоит на staging окружение
   - Запускает интеграционные тесты
3. Код готов к релизу одним кликом

**Continuous Deployment** (продолжение CD) - автоматический деплой в production без ручного подтверждения.

```
Continuous Integration → Continuous Delivery → Continuous Deployment
      (автотесты)            (staging)              (production)
```

## Зачем CI/CD?

### Без CI/CD

```
Developer → Manual build → Manual tests → Manual deploy
   ↓            (errors)      (forgot)       (wrong env)
 Hours         Days           Weeks         Broken prod
```

**Проблемы:**
- Ручная сборка занимает время
- Забыли запустить тесты
- "Работает на моей машине"
- Страх деплоя в production
- Долгий release cycle

### С CI/CD

```
Developer → Auto build → Auto tests → Auto staging → Auto prod
   ↓           ✅            ✅           ✅             ✅
 Seconds      Minutes       Minutes      Minutes       Fast!
```

**Преимущества:**
- Автоматизация всего процесса
- Быстрый feedback (< 10 минут)
- Уверенность в качестве кода
- Частые небольшие релизы
- Меньше багов в production

## Основные компоненты CI/CD

### 1. Version Control (Git)

Триггер для CI/CD pipeline.

```bash
# Разработчик делает коммит
git add .
git commit -m "Add user authentication"
git push origin feature/auth

# CI/CD автоматически запускается
```

### 2. CI/CD Server

Платформа для запуска pipelines:
- **GitLab CI/CD** - встроен в GitLab
- **GitHub Actions** - встроен в GitHub
- **Jenkins** - self-hosted, популярен в enterprise
- **CircleCI** - cloud-based
- **Travis CI** - cloud-based
- **TeamCity** - JetBrains
- **Drone CI** - container-native

### 3. Pipeline

Последовательность шагов (stages) для сборки и деплоя.

**Типичный pipeline:**
```
1. Build    → Компиляция кода, сборка Docker image
2. Test     → Unit tests, integration tests
3. Quality  → Linters, code coverage, security scan
4. Deploy   → Деплой на staging/production
```

### 4. Artifacts

Результаты сборки:
- Docker images
- Бинарные файлы
- JAR/WAR файлы
- Статические сайты

Хранятся в artifact registry:
- Docker Hub
- GitHub Container Registry
- GitLab Container Registry
- Nexus, Artifactory

### 5. Environments

Окружения для деплоя:
- **Development** - локальная разработка
- **Staging** - тестовое окружение (копия production)
- **Production** - рабочее окружение для пользователей

## CI/CD Pipeline для Go приложения

### Пример структуры

```
myapp/
├── cmd/
│   └── server/
│       └── main.go
├── internal/
│   ├── handler/
│   └── service/
├── Dockerfile
├── .gitlab-ci.yml       # GitLab CI config
├── .github/workflows/   # GitHub Actions config
│   └── ci.yml
├── go.mod
├── go.sum
└── README.md
```

### GitLab CI/CD (.gitlab-ci.yml)

```yaml
# Определить stages
stages:
  - test
  - build
  - deploy

# Переменные
variables:
  DOCKER_IMAGE: registry.gitlab.com/myorg/myapp
  DOCKER_TAG: $CI_COMMIT_SHORT_SHA

# Шаг 1: Тесты
test:
  stage: test
  image: golang:1.21-alpine
  script:
    # Скачать зависимости
    - go mod download

    # Запустить тесты
    - go test -v ./...

    # Code coverage
    - go test -coverprofile=coverage.out ./...
    - go tool cover -func=coverage.out

    # Linter
    - go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest
    - golangci-lint run
  coverage: '/total:.*?(\d+\.\d+)%/'

  # Артефакты
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml

# Шаг 2: Сборка Docker image
build:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  script:
    # Логин в registry
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY

    # Собрать image
    - docker build -t $DOCKER_IMAGE:$DOCKER_TAG .
    - docker tag $DOCKER_IMAGE:$DOCKER_TAG $DOCKER_IMAGE:latest

    # Push image
    - docker push $DOCKER_IMAGE:$DOCKER_TAG
    - docker push $DOCKER_IMAGE:latest
  only:
    - main
    - develop

# Шаг 3: Деплой на staging
deploy:staging:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    # Установить kubeconfig
    - echo $KUBE_CONFIG | base64 -d > kubeconfig.yaml
    - export KUBECONFIG=kubeconfig.yaml

    # Обновить image в Kubernetes
    - kubectl set image deployment/myapp myapp=$DOCKER_IMAGE:$DOCKER_TAG -n staging

    # Дождаться rollout
    - kubectl rollout status deployment/myapp -n staging
  environment:
    name: staging
    url: https://staging.example.com
  only:
    - develop

# Шаг 4: Деплой на production (ручное подтверждение)
deploy:production:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - echo $KUBE_CONFIG | base64 -d > kubeconfig.yaml
    - export KUBECONFIG=kubeconfig.yaml
    - kubectl set image deployment/myapp myapp=$DOCKER_IMAGE:$DOCKER_TAG -n production
    - kubectl rollout status deployment/myapp -n production
  environment:
    name: production
    url: https://example.com
  when: manual  # Требует ручного подтверждения
  only:
    - main
```

### GitHub Actions (.github/workflows/ci.yml)

```yaml
name: CI/CD Pipeline

# Триггеры
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  DOCKER_IMAGE: ghcr.io/${{ github.repository }}
  GO_VERSION: '1.21'

jobs:
  # Job 1: Тесты
  test:
    name: Test
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Go
        uses: actions/setup-go@v4
        with:
          go-version: ${{ env.GO_VERSION }}

      - name: Cache Go modules
        uses: actions/cache@v3
        with:
          path: ~/go/pkg/mod
          key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}

      - name: Download dependencies
        run: go mod download

      - name: Run tests
        run: go test -v -coverprofile=coverage.out ./...

      - name: Coverage
        run: go tool cover -func=coverage.out

      - name: Lint
        uses: golangci/golangci-lint-action@v3
        with:
          version: latest

  # Job 2: Сборка Docker image
  build:
    name: Build
    runs-on: ubuntu-latest
    needs: test
    if: github.event_name == 'push'
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.DOCKER_IMAGE }}
          tags: |
            type=ref,event=branch
            type=sha,prefix={{branch}}-

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=registry,ref=${{ env.DOCKER_IMAGE }}:buildcache
          cache-to: type=registry,ref=${{ env.DOCKER_IMAGE }}:buildcache,mode=max

  # Job 3: Деплой на staging
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/develop'
    environment:
      name: staging
      url: https://staging.example.com
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup kubectl
        uses: azure/setup-kubectl@v3

      - name: Configure kubectl
        run: |
          echo "${{ secrets.KUBE_CONFIG }}" | base64 -d > kubeconfig.yaml
          echo "KUBECONFIG=kubeconfig.yaml" >> $GITHUB_ENV

      - name: Deploy to Kubernetes
        run: |
          kubectl set image deployment/myapp \
            myapp=${{ env.DOCKER_IMAGE }}:develop-${{ github.sha }} \
            -n staging
          kubectl rollout status deployment/myapp -n staging

  # Job 4: Деплой на production
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
      url: https://example.com
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup kubectl
        uses: azure/setup-kubectl@v3

      - name: Configure kubectl
        run: |
          echo "${{ secrets.KUBE_CONFIG }}" | base64 -d > kubeconfig.yaml
          echo "KUBECONFIG=kubeconfig.yaml" >> $GITHUB_ENV

      - name: Deploy to Kubernetes
        run: |
          kubectl set image deployment/myapp \
            myapp=${{ env.DOCKER_IMAGE }}:main-${{ github.sha }} \
            -n production
          kubectl rollout status deployment/myapp -n production
```

## Стратегии деплоя

### 1. Rolling Deployment (по умолчанию)

Постепенная замена старых подов новыми.

```yaml
# Kubernetes Deployment
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2         # +2 новых пода
      maxUnavailable: 1   # -1 старый под
```

**Процесс:**
```
v1: [■][■][■][■][■][■][■][■][■][■]  (10 подов v1)
    [■][■][■][■][■][■][■][■][□][□]  (8 v1, 2 v2)
    [■][■][■][■][■][■][□][□][□][□]  (6 v1, 4 v2)
    ...
v2: [□][□][□][□][□][□][□][□][□][□]  (10 подов v2)
```

**Плюсы:** Нет downtime
**Минусы:** Одновременно работают 2 версии

### 2. Blue-Green Deployment

Два полных окружения: Blue (текущая версия) и Green (новая версия).

```yaml
# Service переключается между Blue и Green
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
    version: blue  # Переключить на green после проверки
```

**Процесс:**
1. Blue (v1) в production
2. Деплой Green (v2) параллельно
3. Тестирование Green
4. Переключить трафик на Green
5. Удалить Blue

**Плюсы:** Быстрый rollback, тестирование в production окружении
**Минусы:** Требует 2x ресурсов

### 3. Canary Deployment

Постепенное переключение трафика на новую версию.

```yaml
# Service Mesh (Istio) routing
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  http:
  - route:
    - destination:
        host: myapp
        subset: v1
      weight: 90  # 90% трафика на v1
    - destination:
        host: myapp
        subset: v2
      weight: 10  # 10% трафика на v2 (canary)
```

**Процесс:**
1. 100% трафика на v1
2. Деплой v2, направить 10% трафика
3. Мониторинг ошибок
4. Постепенно увеличивать: 25% → 50% → 75% → 100%
5. Удалить v1

**Плюсы:** Минимизация риска, A/B тестирование
**Минусы:** Сложнее настройка, требует monitoring

### 4. Feature Flags

Включение/выключение фич через конфигурацию.

```go
// Feature flag в коде
if featureFlags.IsEnabled("new-checkout-flow") {
    // Новый код
    return newCheckoutFlow(user)
} else {
    // Старый код
    return oldCheckoutFlow(user)
}
```

```yaml
# ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: feature-flags
data:
  new-checkout-flow: "false"  # Выключено по умолчанию
```

**Плюсы:** Деплой без риска, gradual rollout, A/B testing
**Минусы:** Tech debt (старый + новый код)

## Секреты в CI/CD

### GitLab CI/CD Variables

```yaml
# Settings → CI/CD → Variables
DOCKER_PASSWORD: ********
KUBE_CONFIG: ********
DATABASE_URL: ********
```

Использование:
```yaml
script:
  - echo $DATABASE_URL  # Автоматически masked в логах
  - docker login -u $CI_REGISTRY_USER -p $DOCKER_PASSWORD
```

### GitHub Actions Secrets

```yaml
# Settings → Secrets and variables → Actions
DOCKER_PASSWORD: ********
KUBE_CONFIG: ********
```

Использование:
```yaml
- name: Login to Docker
  run: docker login -u ${{ secrets.DOCKER_USERNAME }} -p ${{ secrets.DOCKER_PASSWORD }}
```

### Best Practices для секретов

1. ✅ **Никогда не коммитьте секреты** в Git
2. ✅ **Используйте CI/CD variables/secrets**
3. ✅ **Разные секреты для разных окружений**
4. ✅ **Ротация секретов** регулярно
5. ✅ **Минимальные права** для CI/CD токенов

## Caching для ускорения

### GitLab CI cache

```yaml
test:
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths:
      - .cache/go-build
      - go/pkg/mod
  script:
    - go test ./...
```

### GitHub Actions cache

```yaml
- name: Cache Go modules
  uses: actions/cache@v3
  with:
    path: ~/go/pkg/mod
    key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}
```

### Docker layer caching

```yaml
# BuildKit cache
- docker build --cache-from $IMAGE:buildcache -t $IMAGE:latest .
```

## Мониторинг CI/CD

### Метрики

- **Build success rate** - % успешных сборок
- **Build duration** - время сборки
- **Deployment frequency** - как часто деплоим
- **Lead time** - время от commit до production
- **MTTR** (Mean Time To Recovery) - время восстановления после ошибки

### Уведомления

```yaml
# GitLab: Settings → Integrations → Slack
# GitHub: Settings → Webhooks

# Уведомления в Slack при ошибках
notifications:
  slack:
    on_success: change
    on_failure: always
```

## Best Practices

### 1. Быстрый feedback

```yaml
# ✅ Хорошо: быстрые тесты первыми
stages:
  - lint        # 30 секунд
  - unit-test   # 2 минуты
  - build       # 5 минут
  - e2e-test    # 10 минут

# ❌ Плохо: долгие тесты первыми
stages:
  - e2e-test    # 10 минут (fail часто)
  - lint        # Ждём 10 минут чтобы узнать о typo
```

### 2. Параллельные jobs

```yaml
# ✅ Запускать параллельно
test-backend:
  script: go test ./...

test-frontend:
  script: npm test
```

### 3. Immutable artifacts

```yaml
# ✅ Хорошо: один Docker image для всех окружений
build:
  script:
    - docker build -t myapp:$CI_COMMIT_SHA .

deploy:staging:
  script:
    - kubectl set image deployment/myapp myapp=myapp:$CI_COMMIT_SHA

# ❌ Плохо: разные сборки для разных окружений
```

### 4. Автоматические тесты

```yaml
# Минимум:
- Unit tests
- Integration tests
- Linters
- Security scan

# Опционально:
- E2E tests
- Performance tests
- Load tests
```

### 5. Small, frequent deploys

```
❌ Плохо: 1 большой релиз в месяц (100 изменений)
✅ Хорошо: 20 маленьких релизов в месяц (5 изменений каждый)
```

## Связанные темы

- [[GitLab CI-CD]]
- [[GitHub Actions]]
- [[Docker - Основы]]
- [[Kubernetes - Основы]]
- [[Git - Gitflow и trunk-based development]]
- [[Prometheus]]
- [[Микросервисная архитектура]]
