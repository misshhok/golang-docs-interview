# Kubernetes - Pods и Deployments

Подробное руководство по Pods и Deployments - основным workload ресурсам Kubernetes.

## Pod

**Pod** - минимальная deployable единица в Kubernetes. Один или несколько тесно связанных контейнеров, которые:
- Запускаются на одной ноде
- Шарят network namespace (localhost между контейнерами)
- Могут шарить volumes
- Имеют общий IP адрес

### Простой Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
    tier: backend
spec:
  containers:
  - name: myapp
    image: myapp:1.0.0
    ports:
    - containerPort: 8080
```

Создать:
```bash
kubectl apply -f pod.yaml
kubectl get pods
kubectl describe pod myapp-pod
```

### Multi-container Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  containers:
  # Основной контейнер
  - name: nginx
    image: nginx:alpine
    ports:
    - containerPort: 80
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html

  # Sidecar контейнер для логов
  - name: log-shipper
    image: fluent/fluentd:latest
    volumeMounts:
    - name: shared-data
      mountPath: /logs

  volumes:
  - name: shared-data
    emptyDir: {}
```

**Паттерны multi-container pods:**

1. **Sidecar** - дополнительный контейнер (логирование, мониторинг)
2. **Ambassador** - proxy к внешним сервисам
3. **Adapter** - нормализация данных

### Environment Variables

```yaml
spec:
  containers:
  - name: myapp
    image: myapp:1.0.0
    env:
    # Простое значение
    - name: PORT
      value: "8080"

    # Из ConfigMap
    - name: APP_CONFIG
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: config.yaml

    # Из Secret
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password

    # Pod metadata
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name

    - name: POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP

    # envFrom: все ключи из ConfigMap
    envFrom:
    - configMapRef:
        name: app-config
```

### Resources и Limits

```yaml
spec:
  containers:
  - name: myapp
    image: myapp:1.0.0
    resources:
      requests:
        cpu: "100m"      # 0.1 CPU core
        memory: "128Mi"
      limits:
        cpu: "500m"      # 0.5 CPU core
        memory: "512Mi"
```

**CPU:**
- `1` = 1 CPU core
- `100m` = 0.1 CPU (100 millicores)
- `500m` = 0.5 CPU

**Memory:**
- `128Mi` = 128 mebibytes (134 MB)
- `1Gi` = 1 gibibyte (1.074 GB)

**QoS классы:**

| Requests | Limits | QoS Class | Описание |
|----------|--------|-----------|----------|
| = Limits | Установлены | Guaranteed | Высший приоритет |
| < Limits | Установлены | Burstable | Средний приоритет |
| Не установлены | Не установлены | BestEffort | Низший приоритет |

При нехватке ресурсов Kubernetes убивает поды с низким приоритетом первыми.

### Probes (Health Checks)

#### Liveness Probe

Проверяет, жив ли контейнер. При fail - kubelet перезапускает контейнер.

```yaml
spec:
  containers:
  - name: myapp
    image: myapp:1.0.0
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 15  # Задержка перед первой проверкой
      periodSeconds: 10        # Интервал проверок
      timeoutSeconds: 3        # Timeout запроса
      failureThreshold: 3      # Кол-во fails перед рестартом
      successThreshold: 1      # Кол-во success после fail
```

**Go endpoint для liveness:**
```go
http.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
    // Простая проверка что приложение живо
    w.WriteHeader(http.StatusOK)
    fmt.Fprint(w, "OK")
})
```

#### Readiness Probe

Проверяет, готов ли контейнер принимать трафик. При fail - под удаляется из Service endpoints.

```yaml
spec:
  containers:
  - name: myapp
    image: myapp:1.0.0
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
```

**Go endpoint для readiness:**
```go
var db *sql.DB

http.HandleFunc("/ready", func(w http.ResponseWriter, r *http.Request) {
    // Проверить подключение к БД
    if err := db.Ping(); err != nil {
        http.Error(w, "Database unavailable", http.StatusServiceUnavailable)
        return
    }

    // Проверить другие зависимости (Redis, Kafka, etc)

    w.WriteHeader(http.StatusOK)
    fmt.Fprint(w, "Ready")
})
```

#### Startup Probe

Для медленно стартующих приложений. Отключает liveness/readiness пока приложение не поднимется.

```yaml
spec:
  containers:
  - name: myapp
    image: myapp:1.0.0
    startupProbe:
      httpGet:
        path: /health
        port: 8080
      failureThreshold: 30  # 30 * 10 = 300 секунд макс.
      periodSeconds: 10
```

**Типы проверок:**

```yaml
# HTTP GET
livenessProbe:
  httpGet:
    path: /health
    port: 8080
    httpHeaders:
    - name: X-Custom-Header
      value: Awesome

# TCP Socket
livenessProbe:
  tcpSocket:
    port: 8080

# Exec команда
livenessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
```

### Restart Policy

```yaml
spec:
  restartPolicy: Always  # Always, OnFailure, Never
```

- `Always` - перезапускать всегда (по умолчанию)
- `OnFailure` - только при ошибке
- `Never` - не перезапускать

### Init Containers

Выполняются ПЕРЕД основными контейнерами. Используются для инициализации.

```yaml
spec:
  initContainers:
  # 1. Ждать доступности БД
  - name: wait-for-db
    image: busybox:1.35
    command:
    - sh
    - -c
    - |
      until nc -z postgres 5432; do
        echo "Waiting for postgres..."
        sleep 2
      done
      echo "PostgreSQL is up!"

  # 2. Запустить миграции
  - name: migrate
    image: migrate/migrate:latest
    command:
    - migrate
    - -path=/migrations
    - -database=$(DATABASE_URL)
    - up
    env:
    - name: DATABASE_URL
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: url

  containers:
  - name: myapp
    image: myapp:1.0.0
    # Запустится только после успешного выполнения init containers
```

## ReplicaSet

Гарантирует, что N подов всегда запущено.

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
        ports:
        - containerPort: 8080
```

**ReplicaSet автоматически:**
- Создаёт новые поды если их меньше `replicas`
- Удаляет лишние поды если их больше `replicas`
- Перезапускает упавшие поды

**Обычно вы НЕ создаёте ReplicaSet напрямую** - используйте Deployment!

## Deployment

High-level абстракция для управления ReplicaSets. Добавляет rolling updates, rollbacks, версионирование.

### Базовый Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
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

Применить:
```bash
kubectl apply -f deployment.yaml

# Проверить
kubectl get deployments
kubectl get replicasets
kubectl get pods
```

### Стратегии обновления

#### RollingUpdate (по умолчанию)

Постепенная замена старых подов новыми.

```yaml
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2         # Макс. +2 пода сверх replicas во время обновления
      maxUnavailable: 1   # Макс. 1 unavailable под
```

**Процесс:**
1. Создать 2 новых пода (maxSurge=2)
2. Дождаться их готовности (readiness probe)
3. Удалить 1 старый под (maxUnavailable=1)
4. Повторять пока все поды не обновятся

**Пример с 10 репликами:**
```
Было:       [v1][v1][v1][v1][v1][v1][v1][v1][v1][v1]  (10 подов v1.0)
maxSurge=2: [v1][v1][v1][v1][v1][v1][v1][v1][v1][v1][v2][v2]  (12 подов, +2 новых)
После ready:[v2][v2][v1][v1][v1][v1][v1][v1][v1][v1]  (10 подов, убрали 2 старых)
...продолжается до:
Стало:      [v2][v2][v2][v2][v2][v2][v2][v2][v2][v2]  (10 подов v2.0)
```

#### Recreate

Удалить все старые поды, затем создать новые. **Downtime!**

```yaml
spec:
  strategy:
    type: Recreate
```

**Когда использовать:**
- Нельзя запускать старую и новую версии одновременно
- Приложение использует Persistent Volume (RWO mode)

### Обновление Deployment

```bash
# Метод 1: Изменить image
kubectl set image deployment/myapp myapp=myapp:2.0.0

# Метод 2: Редактировать напрямую
kubectl edit deployment myapp

# Метод 3: Изменить YAML и применить
kubectl apply -f deployment.yaml

# Следить за rollout
kubectl rollout status deployment/myapp

# Пауза rollout
kubectl rollout pause deployment/myapp

# Возобновить
kubectl rollout resume deployment/myapp
```

### Rollback

```bash
# История ревизий
kubectl rollout history deployment/myapp

# Детали конкретной ревизии
kubectl rollout history deployment/myapp --revision=2

# Откатиться к предыдущей версии
kubectl rollout undo deployment/myapp

# Откатиться к конкретной ревизии
kubectl rollout undo deployment/myapp --to-revision=2
```

**Сохранить причину изменения:**
```bash
kubectl apply -f deployment.yaml --record
```

Тогда в history будет видно команду изменения.

### Масштабирование

```bash
# Изменить количество реплик
kubectl scale deployment myapp --replicas=5

# Автоматическое масштабирование (HPA)
kubectl autoscale deployment myapp --min=3 --max=10 --cpu-percent=80
```

**HorizontalPodAutoscaler (HPA):**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 80
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

HPA автоматически увеличивает/уменьшает replicas на основе CPU/memory.

## Production-ready Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
  labels:
    app: myapp
    version: "1.0.0"
spec:
  replicas: 3
  revisionHistoryLimit: 10  # Сколько хранить старых ReplicaSets

  # Стратегия обновления
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0  # Zero downtime

  selector:
    matchLabels:
      app: myapp

  template:
    metadata:
      labels:
        app: myapp
        version: "1.0.0"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"

    spec:
      # Affinity: распределять поды по разным нодам
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - myapp
              topologyKey: kubernetes.io/hostname

      # Init контейнеры для миграций
      initContainers:
      - name: migrate
        image: migrate/migrate:latest
        command:
        - migrate
        - -path=/migrations
        - -database=$(DATABASE_URL)
        - up
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url

      containers:
      - name: myapp
        image: myapp:1.0.0
        imagePullPolicy: IfNotPresent

        ports:
        - name: http
          containerPort: 8080
          protocol: TCP

        # Environment variables
        env:
        - name: PORT
          value: "8080"
        - name: LOG_LEVEL
          value: "info"
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url
        - name: REDIS_URL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: redis_url

        # Resources
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi

        # Liveness probe
        livenessProbe:
          httpGet:
            path: /health
            port: http
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3

        # Readiness probe
        readinessProbe:
          httpGet:
            path: /ready
            port: http
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3

        # Startup probe для медленного старта
        startupProbe:
          httpGet:
            path: /health
            port: http
          failureThreshold: 30
          periodSeconds: 10

        # Security context
        securityContext:
          runAsNonRoot: true
          runAsUser: 1000
          readOnlyRootFilesystem: true
          allowPrivilegeEscalation: false

        # Volume mounts
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /app/cache

      # Volumes
      volumes:
      - name: tmp
        emptyDir: {}
      - name: cache
        emptyDir: {}

      # Security
      securityContext:
        fsGroup: 1000

      # Graceful shutdown
      terminationGracePeriodSeconds: 30
```

## Pod Lifecycle

```
Pending → ContainerCreating → Running → Succeeded/Failed
```

**Фазы:**
1. **Pending** - под создан, но контейнеры ещё не запущены
2. **ContainerCreating** - pull image, start container
3. **Running** - как минимум один контейнер запущен
4. **Succeeded** - все контейнеры завершились успешно (exit code 0)
5. **Failed** - хотя бы один контейнер завершился с ошибкой
6. **Unknown** - состояние не удалось определить

## Pod Conditions

```bash
kubectl describe pod myapp-pod-12345
```

```
Conditions:
  Type              Status
  Initialized       True    # Init containers завершены
  Ready             True    # Pod ready для приёма трафика
  ContainersReady   True    # Все контейнеры ready
  PodScheduled      True    # Pod назначен на ноду
```

## Debugging

### Логи

```bash
# Логи пода
kubectl logs myapp-pod-12345

# Follow (tail -f)
kubectl logs -f myapp-pod-12345

# Предыдущий crashed контейнер
kubectl logs myapp-pod-12345 --previous

# Multi-container pod: указать контейнер
kubectl logs myapp-pod-12345 -c container-name

# Все поды deployment
kubectl logs -l app=myapp --all-containers=true
```

### Exec в контейнер

```bash
# Bash shell
kubectl exec -it myapp-pod-12345 -- /bin/bash

# Sh (для alpine)
kubectl exec -it myapp-pod-12345 -- /bin/sh

# Выполнить команду
kubectl exec myapp-pod-12345 -- env
kubectl exec myapp-pod-12345 -- cat /etc/config/app.yaml
```

### Describe

```bash
# Детальная информация о поде
kubectl describe pod myapp-pod-12345

# События (Events)
kubectl get events --sort-by='.lastTimestamp'
kubectl get events --field-selector involvedObject.name=myapp-pod-12345
```

### Port Forward

```bash
# Локальный порт 8080 → pod порт 8080
kubectl port-forward pod/myapp-pod-12345 8080:8080

# Deployment
kubectl port-forward deployment/myapp 8080:8080

# Service
kubectl port-forward service/myapp 8080:80
```

Теперь можно обратиться: `curl http://localhost:8080`

## Best Practices

### 1. Всегда используйте Deployments

```bash
# ❌ Плохо: создавать голые Pods
kubectl run myapp --image=myapp:1.0.0

# ✅ Хорошо: использовать Deployments
kubectl create deployment myapp --image=myapp:1.0.0
```

### 2. Указывайте конкретные версии images

```yaml
# ❌ Плохо
image: myapp:latest

# ✅ Хорошо
image: myapp:1.0.0

# ✅ Ещё лучше
image: myapp:1.0.0@sha256:abcd1234...
```

### 3. Настраивайте resource requests/limits

```yaml
# ✅ Обязательно
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

### 4. Добавляйте health checks

```yaml
# ✅ Минимум liveness + readiness
livenessProbe:
  httpGet:
    path: /health
    port: 8080

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
```

### 5. Labels для организации

```yaml
metadata:
  labels:
    app: myapp
    version: "1.0.0"
    environment: production
    tier: backend
    team: platform
```

### 6. Security

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
```

### 7. Graceful shutdown

**Go пример:**
```go
func main() {
    srv := &http.Server{Addr: ":8080"}

    // Handle SIGTERM для graceful shutdown
    stop := make(chan os.Signal, 1)
    signal.Notify(stop, os.Interrupt, syscall.SIGTERM)

    go func() {
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatal(err)
        }
    }()

    <-stop
    log.Println("Shutting down gracefully...")

    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := srv.Shutdown(ctx); err != nil {
        log.Fatal(err)
    }
    log.Println("Server stopped")
}
```

```yaml
spec:
  terminationGracePeriodSeconds: 30  # Время на graceful shutdown
```

### 8. Не запускайте несколько приложений в одном поде

```yaml
# ❌ Плохо: несколько приложений
containers:
- name: frontend
  image: frontend:1.0
- name: backend
  image: backend:1.0

# ✅ Хорошо: создать отдельные Deployments
```

**Исключение:** Sidecar паттерн (логирование, мониторинг, proxy).

## Связанные темы

- [[Kubernetes - Основы]]
- [[Kubernetes - Services и Ingress]]
- [[Kubernetes - ConfigMaps и Secrets]]
- [[Docker - Основы]]
- [[Микросервисная архитектура]]
- [[Prometheus]]
