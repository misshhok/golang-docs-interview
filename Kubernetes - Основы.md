# Kubernetes - Основы

Kubernetes (K8s) - платформа для автоматизации развёртывания, масштабирования и управления контейнеризированными приложениями.

## Что такое Kubernetes?

Kubernetes оркестрирует Docker контейнеры:
- **Автоматическое размещение** контейнеров на нодах
- **Масштабирование** приложений (horizontal scaling)
- **Self-healing** - автоматический перезапуск упавших контейнеров
- **Load balancing** - распределение трафика
- **Rolling updates** - обновление без даунтайма
- **Rollback** - откат к предыдущей версии
- **Управление секретами** и конфигурациями

## Зачем Kubernetes?

### Без Kubernetes (Docker Compose)

```yaml
# docker-compose.yml
services:
  app:
    image: myapp:1.0.0
    deploy:
      replicas: 3
    ports:
      - "8080:8080"
```

**Проблемы:**
- Работает только на одном хосте
- Нет автоматического failover между хостами
- Сложно масштабировать на несколько серверов
- Нет встроенного service discovery
- Ручное управление обновлениями

### С Kubernetes

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
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
        ports:
        - containerPort: 8080
```

Kubernetes автоматически:
- Распределяет поды по нодам
- Перезапускает упавшие поды
- Балансирует нагрузку
- Обновляет версии без даунтайма

## Архитектура Kubernetes

### Кластер

**Kubernetes кластер** = Control Plane + Worker Nodes

```
┌─────────────────────────────────────────┐
│         Control Plane (Master)          │
│  ┌──────────┐  ┌────────┐  ┌─────────┐ │
│  │API Server│  │Scheduler│  │Controller│ │
│  └──────────┘  └────────┘  └─────────┘ │
│       ┌──────┐                          │
│       │ etcd │                          │
│       └──────┘                          │
└─────────────────────────────────────────┘
           │
    ┌──────┴──────┐
    │             │
┌───▼────┐   ┌────▼───┐
│ Node 1 │   │ Node 2 │ ... Worker Nodes
└────────┘   └────────┘
```

### Control Plane компоненты

**1. API Server (`kube-apiserver`)**
- Центральный компонент Kubernetes
- REST API для всех операций
- Все команды `kubectl` идут через API Server

**2. etcd**
- Distributed key-value хранилище
- Хранит всё состояние кластера
- Конфигурации, метаданные, состояние подов

**3. Scheduler (`kube-scheduler`)**
- Выбирает ноду для запуска подов
- Учитывает ресурсы (CPU, RAM), affinity, taints

**4. Controller Manager (`kube-controller-manager`)**
- Запускает контроллеры (Deployment, ReplicaSet, etc)
- Следит за состоянием кластера
- Приводит текущее состояние к желаемому

**5. Cloud Controller Manager** (опционально)
- Интеграция с облачными провайдерами (AWS, GCP, Azure)
- Управление LoadBalancers, Volumes

### Worker Node компоненты

**1. kubelet**
- Агент на каждой ноде
- Запускает и мониторит поды
- Взаимодействует с API Server

**2. kube-proxy**
- Сетевой прокси на каждой ноде
- Реализует Service abstraction
- Балансировка трафика между подами

**3. Container Runtime**
- Docker, containerd, CRI-O
- Запускает контейнеры

## Основные объекты Kubernetes

### Pod

Минимальная единица деплоя. Pod = один или несколько контейнеров, которые всегда запускаются вместе.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: myapp:1.0.0
    ports:
    - containerPort: 8080
    env:
    - name: DATABASE_URL
      value: "postgres://..."
```

**Характеристики Pod:**
- Поды эфемерны (могут быть удалены/пересозданы в любой момент)
- Каждый под имеет уникальный IP в кластере
- Контейнеры внутри пода share:
  - Network namespace (localhost между контейнерами)
  - Volumes
  - IP адрес

### ReplicaSet

Гарантирует, что заданное количество подов всегда запущено.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: myapp-replicaset
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
```

Если под упадёт, ReplicaSet автоматически создаст новый.

### Deployment

High-level абстракция над ReplicaSet. Управляет rolling updates и rollbacks.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
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
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
```

**Deployment автоматически:**
- Создаёт ReplicaSet
- Обновляет поды без даунтайма
- Откатывается при ошибках

### Service

Абстракция для доступа к подам. Service имеет стабильный IP и DNS имя.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  type: ClusterIP
  selector:
    app: myapp
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
```

**Типы Service:**
1. **ClusterIP** (по умолчанию) - внутри кластера
2. **NodePort** - доступ через порт на каждой ноде
3. **LoadBalancer** - внешний load balancer (AWS ELB, GCP LB)
4. **ExternalName** - CNAME для внешнего сервиса

### Namespace

Логическое разделение кластера.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
```

```bash
# Создать namespace
kubectl create namespace staging

# Деплой в namespace
kubectl apply -f deployment.yaml -n staging

# Список всех namespaces
kubectl get namespaces
```

**Дефолтные namespaces:**
- `default` - по умолчанию
- `kube-system` - системные компоненты Kubernetes
- `kube-public` - публичные ресурсы
- `kube-node-lease` - health check нод

## Установка Kubernetes

### Локальная разработка

**1. Minikube** (рекомендуется для обучения)

```bash
# Установка (macOS)
brew install minikube

# Запустить кластер
minikube start

# Проверка
kubectl cluster-info
kubectl get nodes
```

**2. Kind (Kubernetes in Docker)**

```bash
# Установка
brew install kind

# Создать кластер
kind create cluster

# Удалить кластер
kind delete cluster
```

**3. Docker Desktop**

В настройках Docker Desktop → включить Kubernetes.

### kubectl - CLI для Kubernetes

```bash
# Установка (macOS)
brew install kubectl

# Проверка
kubectl version --client

# Конфигурация
kubectl config view
kubectl config get-contexts
kubectl config use-context minikube
```

## Основные команды kubectl

### Управление ресурсами

```bash
# Применить конфигурацию
kubectl apply -f deployment.yaml
kubectl apply -f https://example.com/manifest.yaml

# Создать ресурс
kubectl create deployment myapp --image=myapp:1.0.0

# Удалить ресурс
kubectl delete deployment myapp
kubectl delete -f deployment.yaml

# Получить ресурсы
kubectl get pods
kubectl get deployments
kubectl get services
kubectl get all

# С деталями
kubectl get pods -o wide
kubectl get pods -o yaml
kubectl get pods -o json

# Описание ресурса
kubectl describe pod myapp-pod-12345
kubectl describe deployment myapp

# Логи
kubectl logs myapp-pod-12345
kubectl logs -f myapp-pod-12345  # Follow
kubectl logs myapp-pod-12345 -c container-name  # Конкретный контейнер

# Выполнить команду в поде
kubectl exec -it myapp-pod-12345 -- sh
kubectl exec -it myapp-pod-12345 -- /bin/bash

# Port forwarding
kubectl port-forward pod/myapp-pod-12345 8080:8080

# Масштабирование
kubectl scale deployment myapp --replicas=5

# Обновление image
kubectl set image deployment/myapp myapp=myapp:2.0.0

# Rollout
kubectl rollout status deployment/myapp
kubectl rollout history deployment/myapp
kubectl rollout undo deployment/myapp
```

### Работа с namespaces

```bash
# Создать namespace
kubectl create namespace staging

# Список namespaces
kubectl get namespaces

# Все поды в namespace
kubectl get pods -n staging

# Все поды во всех namespaces
kubectl get pods --all-namespaces
kubectl get pods -A

# Установить namespace по умолчанию
kubectl config set-context --current --namespace=staging
```

### Debug

```bash
# События
kubectl get events
kubectl get events --sort-by='.lastTimestamp'

# Состояние нод
kubectl get nodes
kubectl describe node node-name

# Top (требует metrics-server)
kubectl top nodes
kubectl top pods

# Proxy к API server
kubectl proxy

# Проверить доступность сервиса
kubectl run curl --image=curlimages/curl -i --tty --rm -- sh
```

## Пример: Go приложение в Kubernetes

### Структура проекта

```
myapp/
├── cmd/
│   └── server/
│       └── main.go
├── Dockerfile
├── k8s/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── namespace.yaml
├── go.mod
└── go.sum
```

### main.go

```go
package main

import (
    "fmt"
    "log"
    "net/http"
    "os"
)

func main() {
    port := os.Getenv("PORT")
    if port == "" {
        port = "8080"
    }

    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        hostname, _ := os.Hostname()
        fmt.Fprintf(w, "Hello from pod: %s\n", hostname)
    })

    http.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        fmt.Fprint(w, "OK")
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
RUN CGO_ENABLED=0 go build -o main ./cmd/server

FROM alpine:latest

RUN apk --no-cache add ca-certificates
RUN adduser -D -u 1000 appuser

WORKDIR /home/appuser
COPY --from=builder /app/main .

USER appuser

EXPOSE 8080

CMD ["./main"]
```

### k8s/namespace.yaml

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
```

### k8s/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: myapp
  labels:
    app: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
        version: "1.0.0"
    spec:
      containers:
      - name: myapp
        image: myapp:1.0.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
          name: http
        env:
        - name: PORT
          value: "8080"
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
```

### k8s/service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
  namespace: myapp
spec:
  type: LoadBalancer
  selector:
    app: myapp
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
```

### Деплой

```bash
# 1. Построить image
docker build -t myapp:1.0.0 .

# 2. Загрузить image в minikube (для локальной разработки)
minikube image load myapp:1.0.0

# 3. Создать namespace
kubectl apply -f k8s/namespace.yaml

# 4. Деплой приложения
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml

# 5. Проверить
kubectl get pods -n myapp
kubectl get services -n myapp

# 6. Получить URL сервиса (minikube)
minikube service myapp-service -n myapp --url

# 7. Тестировать
curl http://<service-url>/
```

## Labels и Selectors

Labels - key-value метаданные для организации ресурсов.

```yaml
metadata:
  labels:
    app: myapp
    environment: production
    tier: backend
    version: "1.0.0"
```

**Selectors:**

```yaml
# Equality-based
selector:
  matchLabels:
    app: myapp
    environment: production

# Set-based
selector:
  matchExpressions:
  - key: environment
    operator: In
    values:
    - production
    - staging
  - key: tier
    operator: NotIn
    values:
    - frontend
```

**kubectl с селекторами:**

```bash
# Получить поды с label
kubectl get pods -l app=myapp
kubectl get pods -l environment=production,tier=backend

# Удалить по label
kubectl delete pods -l app=myapp
```

## Resources и Limits

```yaml
containers:
- name: myapp
  resources:
    requests:
      cpu: 100m       # 0.1 CPU
      memory: 128Mi
    limits:
      cpu: 500m       # 0.5 CPU
      memory: 512Mi
```

- **requests** - минимальные ресурсы, гарантированные контейнеру
- **limits** - максимальные ресурсы, которые может использовать контейнер

**QoS Classes:**
1. **Guaranteed** - requests == limits
2. **Burstable** - requests < limits
3. **BestEffort** - нет requests/limits

## Health Checks

### Liveness Probe

Проверяет, жив ли контейнер. Если проверка fails, kubelet перезапускает контейнер.

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 10
  timeoutSeconds: 3
  failureThreshold: 3
```

### Readiness Probe

Проверяет, готов ли контейнер принимать трафик. Если fails, под удаляется из Service endpoints.

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

### Startup Probe

Для медленно стартующих приложений.

```yaml
startupProbe:
  httpGet:
    path: /health
    port: 8080
  failureThreshold: 30
  periodSeconds: 10
```

**Типы проверок:**
- `httpGet` - HTTP GET запрос
- `tcpSocket` - TCP соединение
- `exec` - выполнение команды

## Rolling Updates

```bash
# Обновить image
kubectl set image deployment/myapp myapp=myapp:2.0.0

# Следить за rollout
kubectl rollout status deployment/myapp

# История
kubectl rollout history deployment/myapp

# Откатиться
kubectl rollout undo deployment/myapp

# Откатиться к конкретной версии
kubectl rollout undo deployment/myapp --to-revision=2
```

**Стратегии обновления:**

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1         # Макс. кол-во подов сверх replicas
      maxUnavailable: 0   # Макс. кол-во unavailable подов
```

## Best Practices

1. ✅ **Всегда используйте Deployments** вместо голых Pods
2. ✅ **Указывайте конкретные версии images** (не :latest)
3. ✅ **Настраивайте resource requests/limits**
4. ✅ **Добавляйте liveness и readiness probes**
5. ✅ **Используйте namespaces** для разделения окружений
6. ✅ **Добавляйте labels** для организации ресурсов
7. ✅ **Не запускайте от root** в контейнерах
8. ❌ **Не храните state в контейнерах** - используйте Volumes

## Связанные темы

- [[Kubernetes - Pods и Deployments]]
- [[Kubernetes - Services и Ingress]]
- [[Kubernetes - ConfigMaps и Secrets]]
- [[Docker - Основы]]
- [[Docker - Docker Compose]]
- [[Микросервисная архитектура]]
- [[Prometheus]]
