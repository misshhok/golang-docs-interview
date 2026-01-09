# Kubernetes - Services и Ingress

Service и Ingress - механизмы для доступа к подам и организации сетевой связности в Kubernetes.

## Проблема: поды эфемерны

Поды имеют динамические IP адреса и могут быть пересозданы в любой момент:
- Автоскейлинг создаёт/удаляет поды
- Rolling update пересоздаёт поды
- Node failure приводит к перезапуску подов на других нодах

**Невозможно** обращаться к подам напрямую по IP.

## Service

**Service** - абстракция для доступа к группе подов. Предоставляет:
- **Стабильный IP адрес** (Cluster IP)
- **DNS имя** (`service-name.namespace.svc.cluster.local`)
- **Load balancing** между подами
- **Service discovery**

### ClusterIP (по умолчанию)

Доступен **только внутри кластера**.

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
    port: 80        # Порт сервиса
    targetPort: 8080  # Порт контейнера
```

**Как это работает:**
1. Service получает статический Cluster IP (например `10.96.0.1`)
2. Service отслеживает поды с label `app: myapp`
3. Создаёт endpoints - список IP подов
4. kube-proxy настраивает iptables rules для балансировки

```bash
kubectl apply -f service.yaml

# Получить Service
kubectl get services
kubectl get svc

# Endpoints (IP подов)
kubectl get endpoints myapp-service

# Describe
kubectl describe service myapp-service
```

**Доступ изнутри кластера:**
```bash
# DNS имя
curl http://myapp-service
curl http://myapp-service.default.svc.cluster.local

# По IP
curl http://10.96.0.1
```

**Port mapping:**
```yaml
ports:
- name: http
  protocol: TCP
  port: 80        # Клиент обращается к сервису на порт 80
  targetPort: 8080  # Трафик перенаправляется на порт 8080 контейнера
```

Можно указать имя порта:
```yaml
# В Deployment
spec:
  containers:
  - name: myapp
    ports:
    - name: http
      containerPort: 8080

# В Service
ports:
- protocol: TCP
  port: 80
  targetPort: http  # Ссылка на имя порта
```

### NodePort

Открывает порт на **каждой ноде** кластера.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-nodeport
spec:
  type: NodePort
  selector:
    app: myapp
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
    nodePort: 30080  # Порт на нодах (30000-32767)
```

**Доступ:**
```bash
# С любой ноды кластера
curl http://<node-ip>:30080

# Minikube
minikube service myapp-nodeport --url
```

**Kubernetes автоматически:**
1. Создаёт ClusterIP Service
2. Открывает `nodePort` на каждой ноде
3. Балансирует трафик с nodePort на поды

**Когда использовать:**
- Разработка/тестирование
- Когда нет LoadBalancer
- On-premise кластеры

### LoadBalancer

Создаёт внешний load balancer (AWS ELB, GCP LB, Azure LB).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-lb
spec:
  type: LoadBalancer
  selector:
    app: myapp
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
```

**Работает только в облаках** (AWS, GCP, Azure) или с MetalLB (on-premise).

```bash
kubectl apply -f service.yaml

# Получить внешний IP
kubectl get service myapp-lb

# Output:
# NAME       TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)
# myapp-lb   LoadBalancer   10.96.0.1      35.123.45.67     80:31234/TCP
```

Доступ:
```bash
curl http://35.123.45.67
```

### ExternalName

CNAME для внешнего сервиса.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: db.example.com
```

Теперь можно обращаться:
```bash
psql -h external-db -U postgres
# → резолвится в db.example.com
```

### Headless Service

Service **без** Cluster IP. Возвращает IP всех подов напрямую.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-headless
spec:
  clusterIP: None  # Headless
  selector:
    app: myapp
  ports:
  - port: 8080
```

**DNS запрос к headless service:**
```
myapp-headless.default.svc.cluster.local → [10.1.1.1, 10.1.1.2, 10.1.1.3]
```

Возвращает все IP подов, а не один Cluster IP.

**Когда использовать:**
- StatefulSet (stable network identity для каждого пода)
- Прямое обращение к конкретным подам
- Custom load balancing logic

## Service Discovery

### DNS

Kubernetes автоматически создаёт DNS записи для Service.

**Формат:**
```
<service-name>.<namespace>.svc.cluster.local
```

**Примеры:**
```bash
# В том же namespace
curl http://myapp-service

# В другом namespace
curl http://myapp-service.production

# Полное имя
curl http://myapp-service.production.svc.cluster.local
```

**Для headless service (StatefulSet):**
```
<pod-name>.<service-name>.<namespace>.svc.cluster.local
```

Пример:
```bash
# StatefulSet pod
curl http://web-0.nginx-headless.default.svc.cluster.local
```

### Environment Variables

Kubernetes автоматически создаёт environment variables для Services.

```bash
# Для Service "myapp-service" на порту 80
MYAPP_SERVICE_SERVICE_HOST=10.96.0.1
MYAPP_SERVICE_SERVICE_PORT=80
MYAPP_SERVICE_PORT=tcp://10.96.0.1:80
MYAPP_SERVICE_PORT_80_TCP=tcp://10.96.0.1:80
MYAPP_SERVICE_PORT_80_TCP_PROTO=tcp
MYAPP_SERVICE_PORT_80_TCP_PORT=80
MYAPP_SERVICE_PORT_80_TCP_ADDR=10.96.0.1
```

**Go пример:**
```go
dbHost := os.Getenv("POSTGRES_SERVICE_SERVICE_HOST")
dbPort := os.Getenv("POSTGRES_SERVICE_SERVICE_PORT")

db, err := sql.Open("postgres",
    fmt.Sprintf("host=%s port=%s user=postgres", dbHost, dbPort))
```

**Проблема:** Environment variables создаются только для Services, созданных **до** пода.

**Решение:** Используйте DNS вместо environment variables.

## Endpoints

Endpoints - список IP:Port подов, которые match селектору Service.

```bash
kubectl get endpoints myapp-service
```

**Пример:**
```
NAME            ENDPOINTS
myapp-service   10.1.1.1:8080,10.1.1.2:8080,10.1.1.3:8080
```

Kubernetes автоматически управляет endpoints:
- Добавляет поды когда они становятся Ready
- Удаляет поды когда readiness probe fails

### Ручные Endpoints

Service **без selector** для внешних ресурсов.

```yaml
# Service без selector
apiVersion: v1
kind: Service
metadata:
  name: external-service
spec:
  ports:
  - protocol: TCP
    port: 5432
    targetPort: 5432

---
# Ручные Endpoints
apiVersion: v1
kind: Endpoints
metadata:
  name: external-service  # Должно совпадать с именем Service
subsets:
- addresses:
  - ip: 192.168.1.100  # Внешний PostgreSQL
  ports:
  - port: 5432
```

Теперь можно обращаться:
```bash
psql -h external-service -U postgres
```

## Session Affinity

По умолчанию трафик балансируется случайно. Session affinity направляет запросы от одного клиента на один и тот же под.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  type: ClusterIP
  selector:
    app: myapp
  sessionAffinity: ClientIP  # None (default) или ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800  # 3 часа
  ports:
  - port: 80
    targetPort: 8080
```

**Когда использовать:**
- WebSockets
- Sticky sessions

## Ingress

**Ingress** - управляет внешним HTTP/HTTPS доступом к Services.

**Проблема с LoadBalancer Service:**
- Каждый Service создаёт отдельный cloud load balancer ($$$)
- Нет URL-based routing
- Нет TLS termination

**Ingress решает:**
- Один load balancer для многих Services
- Host-based routing (`api.example.com`, `web.example.com`)
- Path-based routing (`/api`, `/admin`)
- TLS/HTTPS termination

### Ingress Controller

**Ingress resource** - просто конфигурация. Нужен **Ingress Controller** для реализации.

**Популярные Ingress Controllers:**
- **NGINX Ingress Controller** (самый популярный)
- Traefik
- HAProxy
- AWS ALB Ingress Controller
- Istio Gateway

**Установка NGINX Ingress Controller:**
```bash
# Minikube
minikube addons enable ingress

# Kubernetes (Helm)
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install nginx-ingress ingress-nginx/ingress-nginx

# Проверка
kubectl get pods -n ingress-nginx
```

### Простой Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-service
            port:
              number: 80
```

**Доступ:**
```bash
curl http://myapp.example.com
# → перенаправляется на myapp-service:80
```

### Path-based routing

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
spec:
  rules:
  - host: example.com
    http:
      paths:
      # /api → api-service
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080

      # /admin → admin-service
      - path: /admin
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 3000

      # / → frontend-service
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

**PathType:**
- `Prefix` - prefix match (`/api` matches `/api/users`)
- `Exact` - точное совпадение (`/api` НЕ matches `/api/users`)
- `ImplementationSpecific` - зависит от Ingress Controller

### Host-based routing

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-host-ingress
spec:
  rules:
  # api.example.com → api-service
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080

  # web.example.com → web-service
  - host: web.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

### TLS/HTTPS

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls-secret  # TLS сертификат
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-service
            port:
              number: 80
```

**Создать TLS Secret:**
```bash
kubectl create secret tls myapp-tls-secret \
  --cert=cert.pem \
  --key=key.pem
```

Или из YAML:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: myapp-tls-secret
type: kubernetes.io/tls
data:
  tls.crt: <base64 encoded cert>
  tls.key: <base64 encoded key>
```

### Annotations

Ingress annotations для дополнительной конфигурации (зависят от Ingress Controller).

**NGINX Ingress Controller примеры:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    # Rewrite URL
    nginx.ingress.kubernetes.io/rewrite-target: /$2

    # SSL Redirect
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://example.com"

    # Rate Limiting
    nginx.ingress.kubernetes.io/limit-rps: "10"

    # Client Body Size
    nginx.ingress.kubernetes.io/proxy-body-size: "8m"

    # Timeouts
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "30"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"

    # Whitelist IPs
    nginx.ingress.kubernetes.io/whitelist-source-range: "10.0.0.0/8,192.168.0.0/16"

    # Basic Auth
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: "Authentication Required"
spec:
  # ...
```

**Rewrite example:**
```yaml
annotations:
  nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /api(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

Запрос: `example.com/api/users` → `api-service:8080/users`

### Default Backend

Fallback для unmatch запросов (404).

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
spec:
  defaultBackend:
    service:
      name: default-backend
      port:
        number: 80
  rules:
  - host: myapp.example.com
    # ...
```

## Production Example

```yaml
# ========================================
# Backend Service
# ========================================
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: api
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 8080

---
# ========================================
# Frontend Service
# ========================================
apiVersion: v1
kind: Service
metadata:
  name: web-service
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: web
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 3000

---
# ========================================
# TLS Secret
# ========================================
apiVersion: v1
kind: Secret
metadata:
  name: example-tls
  namespace: production
type: kubernetes.io/tls
data:
  tls.crt: LS0tLS1CRUdJTi... # base64 encoded
  tls.key: LS0tLS1CRUdJTi... # base64 encoded

---
# ========================================
# Ingress
# ========================================
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: production
  annotations:
    # Force HTTPS
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"

    # CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, DELETE, OPTIONS"
    nginx.ingress.kubernetes.io/cors-allow-credentials: "true"

    # Rate Limiting
    nginx.ingress.kubernetes.io/limit-rps: "100"
    nginx.ingress.kubernetes.io/limit-burst-multiplier: "5"

    # Timeouts
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "300"

    # Client max body size (file uploads)
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"

    # Health check
    nginx.ingress.kubernetes.io/backend-protocol: "HTTP"

spec:
  ingressClassName: nginx

  tls:
  - hosts:
    - example.com
    - api.example.com
    secretName: example-tls

  rules:
  # API subdomain
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80

  # Main domain - frontend + API
  - host: example.com
    http:
      paths:
      # API endpoints
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80

      # Frontend
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

## cert-manager для автоматического TLS

**cert-manager** автоматически получает и обновляет TLS сертификаты от Let's Encrypt.

### Установка cert-manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# Проверка
kubectl get pods -n cert-manager
```

### ClusterIssuer (Let's Encrypt)

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
```

### Ingress с автоматическим TLS

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls-auto  # cert-manager создаст автоматически
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-service
            port:
              number: 80
```

cert-manager автоматически:
1. Получит сертификат от Let's Encrypt
2. Создаст Secret `myapp-tls-auto`
3. Будет обновлять сертификат перед истечением

## Best Practices

### 1. Используйте ClusterIP по умолчанию

```yaml
# ✅ По умолчанию для внутренних сервисов
spec:
  type: ClusterIP

# ❌ Избегайте NodePort в production
spec:
  type: NodePort
```

### 2. Именованные порты

```yaml
# ✅ Хорошо
ports:
- name: http
  port: 80
  targetPort: http  # Ссылка на имя в Deployment

# ❌ Плохо
ports:
- port: 80
  targetPort: 8080
```

### 3. Используйте Ingress вместо LoadBalancer

```yaml
# ✅ Один Ingress для всех сервисов
kind: Ingress

# ❌ Отдельный LoadBalancer для каждого сервиса (дорого!)
kind: Service
spec:
  type: LoadBalancer
```

### 4. TLS/HTTPS везде

```yaml
# ✅ Обязательно в production
annotations:
  nginx.ingress.kubernetes.io/ssl-redirect: "true"
  cert-manager.io/cluster-issuer: "letsencrypt-prod"
```

### 5. Rate Limiting

```yaml
annotations:
  nginx.ingress.kubernetes.io/limit-rps: "100"
```

### 6. Health checks в Ingress

```yaml
annotations:
  nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
  nginx.ingress.kubernetes.io/health-check-path: "/health"
```

## Связанные темы

- [[Kubernetes - Основы]]
- [[Kubernetes - Pods и Deployments]]
- [[Kubernetes - ConfigMaps и Secrets]]
- [[Микросервисная архитектура]]
- [[REST API - Дизайн и best practices]]
- [[Prometheus]]
