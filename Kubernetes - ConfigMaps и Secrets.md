# Kubernetes - ConfigMaps и Secrets

ConfigMaps и Secrets - механизмы для управления конфигурацией и секретами в Kubernetes.

## Проблема: хардкод конфигурации

**Плохо:**
```go
// main.go
const dbHost = "postgres.production.svc.cluster.local"
const dbPort = "5432"
const apiKey = "super-secret-key-12345"
```

Проблемы:
- Нельзя изменить без пересборки image
- Секреты в исходном коде
- Разные значения для dev/staging/production

**Решение:** Хранить конфигурацию отдельно от кода.

## ConfigMap

**ConfigMap** - key-value хранилище для несекретной конфигурации.

### Создание ConfigMap

#### Способ 1: Из literals

```bash
kubectl create configmap app-config \
  --from-literal=database.host=postgres \
  --from-literal=database.port=5432 \
  --from-literal=log.level=info
```

#### Способ 2: Из файла

**config.properties:**
```properties
database.host=postgres
database.port=5432
log.level=info
cache.enabled=true
```

```bash
kubectl create configmap app-config --from-file=config.properties
```

#### Способ 3: Из YAML

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: default
data:
  # Key-value пары
  database.host: "postgres"
  database.port: "5432"
  log.level: "info"

  # Многострочные значения
  app.yaml: |
    server:
      port: 8080
      timeout: 30s
    database:
      host: postgres
      port: 5432
      pool: 10
```

```bash
kubectl apply -f configmap.yaml

# Просмотр
kubectl get configmaps
kubectl describe configmap app-config
kubectl get configmap app-config -o yaml
```

### Использование ConfigMap

#### Как Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
  - name: myapp
    image: myapp:1.0.0
    env:
    # Одна переменная из ConfigMap
    - name: DATABASE_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: database.host

    - name: DATABASE_PORT
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: database.port

    # Все ключи как environment variables
    envFrom:
    - configMapRef:
        name: app-config
```

**Результат:**
```bash
DATABASE_HOST=postgres
DATABASE_PORT=5432
database.host=postgres  # из envFrom
database.port=5432      # из envFrom
log.level=info          # из envFrom
```

**Go пример:**
```go
dbHost := os.Getenv("DATABASE_HOST")
dbPort := os.Getenv("DATABASE_PORT")
logLevel := os.Getenv("log.level")

db, _ := sql.Open("postgres",
    fmt.Sprintf("host=%s port=%s", dbHost, dbPort))
```

#### Как Volume

Монтировать ConfigMap как файлы в контейнер.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
  - name: myapp
    image: myapp:1.0.0
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config  # Директория в контейнере

  volumes:
  - name: config-volume
    configMap:
      name: app-config
```

**Результат в контейнере:**
```
/etc/config/
├── database.host (содержимое: "postgres")
├── database.port (содержимое: "5432")
├── log.level (содержимое: "info")
└── app.yaml (содержимое: многострочный YAML)
```

**Go пример:**
```go
data, err := os.ReadFile("/etc/config/app.yaml")
if err != nil {
    log.Fatal(err)
}

var config Config
yaml.Unmarshal(data, &config)
```

#### Монтировать конкретные ключи

```yaml
volumes:
- name: config-volume
  configMap:
    name: app-config
    items:
    - key: app.yaml       # Ключ в ConfigMap
      path: config.yaml   # Имя файла в контейнере
```

**Результат:**
```
/etc/config/config.yaml  (содержимое из app.yaml ключа)
```

### Обновление ConfigMap

```bash
# Редактировать ConfigMap
kubectl edit configmap app-config

# Применить новый YAML
kubectl apply -f configmap.yaml
```

**ВАЖНО:** Поды НЕ автоматически перезагружаются при изменении ConfigMap!

**Решения:**
1. **Environment variables** - НЕ обновляются. Нужно пересоздать под.
2. **Volume mount** - обновляется через ~1 минуту (kubelet sync period).
3. **Использовать Reloader** - автоматически рестартует поды при изменении ConfigMap.

### ConfigMap Immutable

Неизменяемый ConfigMap (нельзя редактировать после создания).

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  database.host: "postgres"
immutable: true
```

**Преимущества:**
- Защита от случайных изменений
- Улучшенная производительность kube-apiserver (не нужно watch изменения)

## Secret

**Secret** - key-value хранилище для **секретной** информации (пароли, токены, ключи).

### Отличия от ConfigMap

| ConfigMap | Secret |
|-----------|--------|
| Для несекретной конфигурации | Для секретов |
| Хранится в plaintext | Хранится в base64 (НЕ шифрование!) |
| Можно просмотреть `kubectl get configmap -o yaml` | Требует дополнительных прав |

**ВАЖНО:** Base64 - это **НЕ шифрование**, а просто кодирование. Любой с доступом к API может декодировать.

**Для реального шифрования:**
- Включить encryption at rest в etcd
- Использовать external secret managers (HashiCorp Vault, AWS Secrets Manager)

### Создание Secret

#### Способ 1: Из literals

```bash
kubectl create secret generic db-secret \
  --from-literal=username=postgres \
  --from-literal=password=super-secret-password
```

#### Способ 2: Из файлов

```bash
# Создать файлы
echo -n 'postgres' > username.txt
echo -n 'super-secret-password' > password.txt

kubectl create secret generic db-secret \
  --from-file=username=username.txt \
  --from-file=password=password.txt

# Удалить файлы
rm username.txt password.txt
```

#### Способ 3: Из YAML

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  # base64 encoded значения
  username: cG9zdGdyZXM=         # postgres
  password: c3VwZXItc2VjcmV0LXBhc3N3b3Jk  # super-secret-password
```

**Кодирование в base64:**
```bash
echo -n 'postgres' | base64
# Output: cG9zdGdyZXM=

echo -n 'super-secret-password' | base64
# Output: c3VwZXItc2VjcmV0LXBhc3N3b3Jk
```

**Применить:**
```bash
kubectl apply -f secret.yaml

# Просмотр
kubectl get secrets
kubectl describe secret db-secret  # НЕ показывает значения
kubectl get secret db-secret -o yaml  # Показывает base64
```

#### stringData (без base64)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:  # Kubernetes автоматически закодирует в base64
  username: postgres
  password: super-secret-password
```

Удобно для создания секретов вручную.

### Типы Secrets

```yaml
type: Opaque  # Generic key-value (по умолчанию)
```

Другие типы:
- `kubernetes.io/tls` - TLS сертификаты
- `kubernetes.io/dockerconfigjson` - Docker registry credentials
- `kubernetes.io/basic-auth` - Basic authentication
- `kubernetes.io/ssh-auth` - SSH ключи
- `kubernetes.io/service-account-token` - Service account token

### Использование Secret

#### Как Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
  - name: myapp
    image: myapp:1.0.0
    env:
    # Одна переменная из Secret
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: username

    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password

    # Все ключи как environment variables
    envFrom:
    - secretRef:
        name: db-secret
```

**Go пример:**
```go
dbUser := os.Getenv("DB_USERNAME")
dbPass := os.Getenv("DB_PASSWORD")

db, _ := sql.Open("postgres",
    fmt.Sprintf("user=%s password=%s host=postgres", dbUser, dbPass))
```

#### Как Volume

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
  - name: myapp
    image: myapp:1.0.0
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true

  volumes:
  - name: secret-volume
    secret:
      secretName: db-secret
      defaultMode: 0400  # Права доступа (read-only для owner)
```

**Результат в контейнере:**
```
/etc/secrets/
├── username (содержимое: "postgres")
└── password (содержимое: "super-secret-password")
```

**Go пример:**
```go
username, _ := os.ReadFile("/etc/secrets/username")
password, _ := os.ReadFile("/etc/secrets/password")

db, _ := sql.Open("postgres",
    fmt.Sprintf("user=%s password=%s", username, password))
```

**Рекомендация:** Используйте Volume вместо env для секретов:
- Меньше риск утечки через логи
- Можно установить права доступа
- Обновляются автоматически

### TLS Secret

```bash
kubectl create secret tls myapp-tls \
  --cert=tls.crt \
  --key=tls.key
```

Или YAML:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: myapp-tls
type: kubernetes.io/tls
data:
  tls.crt: <base64 encoded cert>
  tls.key: <base64 encoded key>
```

Использование в Ingress:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
spec:
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls  # TLS Secret
  rules:
  - host: myapp.example.com
    # ...
```

### Docker Registry Secret

Для pull приватных images.

```bash
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=myemail@example.com
```

Использование:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  imagePullSecrets:
  - name: regcred
  containers:
  - name: myapp
    image: myregistry.com/myapp:1.0.0
```

## Production Examples

### Микросервис с PostgreSQL

```yaml
# ========================================
# ConfigMap для несекретной конфигурации
# ========================================
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  LOG_LEVEL: "info"
  CACHE_ENABLED: "true"
  REDIS_HOST: "redis.production.svc.cluster.local"
  REDIS_PORT: "6379"

  # Конфигурационный файл
  app.yaml: |
    server:
      port: 8080
      read_timeout: 30s
      write_timeout: 30s
    database:
      pool_size: 10
      max_idle: 5
      max_lifetime: 1h

---
# ========================================
# Secret для database credentials
# ========================================
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  namespace: production
type: Opaque
stringData:
  username: postgres
  password: super-secret-password-here
  database: myapp_production
  url: postgres://postgres:super-secret-password-here@postgres:5432/myapp_production?sslmode=disable

---
# ========================================
# Secret для API keys
# ========================================
apiVersion: v1
kind: Secret
metadata:
  name: api-keys
  namespace: production
type: Opaque
stringData:
  stripe_api_key: sk_live_51ABC123...
  sendgrid_api_key: SG.ABC123...
  jwt_secret: random-jwt-secret-key-here

---
# ========================================
# Deployment
# ========================================
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:1.0.0

        # Environment variables из ConfigMap
        env:
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: LOG_LEVEL

        - name: CACHE_ENABLED
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: CACHE_ENABLED

        # Environment variables из Secrets
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url

        - name: STRIPE_API_KEY
          valueFrom:
            secretKeyRef:
              name: api-keys
              key: stripe_api_key

        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: api-keys
              key: jwt_secret

        # envFrom для всех переменных из ConfigMap
        envFrom:
        - configMapRef:
            name: app-config

        # Volume mounts
        volumeMounts:
        # Монтировать app.yaml из ConfigMap
        - name: config
          mountPath: /etc/config
          readOnly: true

        # Монтировать database credentials из Secret
        - name: db-credentials
          mountPath: /etc/secrets/db
          readOnly: true

      volumes:
      # ConfigMap volume
      - name: config
        configMap:
          name: app-config
          items:
          - key: app.yaml
            path: app.yaml

      # Secret volume
      - name: db-credentials
        secret:
          secretName: db-secret
          defaultMode: 0400
```

**Go приложение:**
```go
package main

import (
    "database/sql"
    "fmt"
    "log"
    "os"

    _ "github.com/lib/pq"
    "gopkg.in/yaml.v3"
)

type Config struct {
    Server struct {
        Port         int    `yaml:"port"`
        ReadTimeout  string `yaml:"read_timeout"`
        WriteTimeout string `yaml:"write_timeout"`
    } `yaml:"server"`
    Database struct {
        PoolSize    int    `yaml:"pool_size"`
        MaxIdle     int    `yaml:"max_idle"`
        MaxLifetime string `yaml:"max_lifetime"`
    } `yaml:"database"`
}

func main() {
    // Читать конфигурацию из файла (ConfigMap volume)
    data, err := os.ReadFile("/etc/config/app.yaml")
    if err != nil {
        log.Fatal(err)
    }

    var config Config
    if err := yaml.Unmarshal(data, &config); err != nil {
        log.Fatal(err)
    }

    // Читать секреты из environment variables
    databaseURL := os.Getenv("DATABASE_URL")
    jwtSecret := os.Getenv("JWT_SECRET")
    stripeKey := os.Getenv("STRIPE_API_KEY")

    // Или из файлов (Secret volume)
    dbUser, _ := os.ReadFile("/etc/secrets/db/username")
    dbPass, _ := os.ReadFile("/etc/secrets/db/password")

    // Подключиться к БД
    db, err := sql.Open("postgres", databaseURL)
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    db.SetMaxOpenConns(config.Database.PoolSize)
    db.SetMaxIdleConns(config.Database.MaxIdle)

    log.Printf("Server starting on port %d", config.Server.Port)
    // ...
}
```

## External Secrets Operator

Для production: интеграция с внешними secret managers.

### HashiCorp Vault

```yaml
# ========================================
# SecretStore (External Secrets Operator)
# ========================================
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: production
spec:
  provider:
    vault:
      server: "https://vault.example.com"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "myapp-role"

---
# ========================================
# ExternalSecret
# ========================================
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-secret
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: db-secret  # Создаст Kubernetes Secret
    creationPolicy: Owner
  data:
  - secretKey: password
    remoteRef:
      key: database/postgres
      property: password
```

External Secrets Operator автоматически:
1. Читает секрет из Vault
2. Создаёт Kubernetes Secret
3. Обновляет при изменении в Vault

## Best Practices

### 1. Разделяйте конфигурацию по окружениям

```
k8s/
├── base/
│   ├── deployment.yaml
│   └── service.yaml
├── dev/
│   ├── configmap.yaml
│   └── secret.yaml
├── staging/
│   ├── configmap.yaml
│   └── secret.yaml
└── production/
    ├── configmap.yaml
    └── secret.yaml
```

### 2. НЕ коммитьте Secrets в Git

```gitignore
# .gitignore
k8s/**/secret.yaml
*.secret.yaml
```

Используйте:
- Sealed Secrets
- External Secrets Operator
- HashiCorp Vault
- AWS Secrets Manager / GCP Secret Manager

### 3. Используйте Volume вместо env для Secrets

```yaml
# ✅ Хорошо: Volume
volumeMounts:
- name: secrets
  mountPath: /etc/secrets
  readOnly: true

# ❌ Плохо: Environment variables (могут попасть в логи)
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-secret
      key: password
```

### 4. Установите права доступа

```yaml
volumes:
- name: secrets
  secret:
    secretName: db-secret
    defaultMode: 0400  # Read-only для owner
```

### 5. Используйте Immutable ConfigMaps

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-v1  # Версионирование
data:
  # ...
immutable: true
```

При изменении создавайте новую версию (`app-config-v2`).

### 6. Не храните sensitive data в ConfigMap

```yaml
# ❌ Плохо
kind: ConfigMap
data:
  password: "secret123"

# ✅ Хорошо
kind: Secret
stringData:
  password: "secret123"
```

### 7. Ротация секретов

```yaml
# Создать новый Secret
apiVersion: v1
kind: Secret
metadata:
  name: db-secret-v2
# ...

# Обновить Deployment
kubectl set env deployment/myapp --from=secret/db-secret-v2

# Удалить старый Secret
kubectl delete secret db-secret-v1
```

### 8. Encryption at Rest

Включить шифрование etcd для production кластеров.

```yaml
# /etc/kubernetes/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
    - secrets
    providers:
    - aescbc:
        keys:
        - name: key1
          secret: <base64 encoded secret>
    - identity: {}
```

## Связанные темы

- [[Kubernetes - Основы]]
- [[Kubernetes - Pods и Deployments]]
- [[Kubernetes - Services и Ingress]]
- [[Docker - Основы]]
- [[Переменные среды и .env]]
- [[Безопасность API]]
