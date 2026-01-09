# gRPC

gRPC (gRPC Remote Procedure Call) — современный высокопроизводительный RPC фреймворк от Google. Использует Protocol Buffers для сериализации и HTTP/2 для транспорта.

## Зачем gRPC вместо REST?

### REST API

```
Client → HTTP/1.1 → JSON → Server
         (text)    (text)

- Медленная сериализация JSON
- HTTP/1.1 - один запрос на соединение
- Нет типизации (нужны доп. проверки)
- Больше трафика (текст)
```

### gRPC

```
Client → HTTP/2 → Protobuf → Server
         (binary) (binary)

- Быстрая сериализация Protobuf (в 5-10 раз быстрее JSON)
- HTTP/2 multiplexing - несколько запросов параллельно
- Строгая типизация из .proto файлов
- Меньше трафика (бинарный формат)
```

## Основы gRPC

### 1. Определение сервиса (.proto)

```protobuf
// user.proto
syntax = "proto3";

package user;

option go_package = "github.com/myapp/pb/user";

// Сервис User
service UserService {
    // Unary RPC - простой запрос-ответ
    rpc GetUser(GetUserRequest) returns (User);

    // Server streaming - сервер возвращает поток данных
    rpc ListUsers(ListUsersRequest) returns (stream User);

    // Client streaming - клиент отправляет поток данных
    rpc CreateUsers(stream CreateUserRequest) returns (CreateUsersResponse);

    // Bidirectional streaming - двусторонний поток
    rpc Chat(stream ChatMessage) returns (stream ChatMessage);
}

// Сообщения (типы данных)
message GetUserRequest {
    int32 id = 1;
}

message User {
    int32 id = 1;
    string name = 2;
    string email = 3;
    int32 age = 4;
    repeated string tags = 5;  // массив
}

message ListUsersRequest {
    int32 page = 1;
    int32 limit = 2;
}

message CreateUserRequest {
    string name = 1;
    string email = 2;
}

message CreateUsersResponse {
    int32 created_count = 1;
}

message ChatMessage {
    string user_id = 1;
    string text = 2;
    int64 timestamp = 3;
}
```

### 2. Генерация Go кода

```bash
# Установка protoc компилятора
# macOS:
brew install protobuf

# Linux:
apt install -y protobuf-compiler

# Установка Go плагинов
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest

# Генерация кода
protoc --go_out=. --go_opt=paths=source_relative \
    --go-grpc_out=. --go-grpc_opt=paths=source_relative \
    user.proto

# Создаст файлы:
# - user.pb.go        (типы данных)
# - user_grpc.pb.go   (сервер и клиент)
```

### 3. Реализация сервера

```go
package main

import (
    "context"
    "log"
    "net"

    pb "github.com/myapp/pb/user"
    "google.golang.org/grpc"
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
)

// Реализация UserService
type userServer struct {
    pb.UnimplementedUserServiceServer
    users map[int32]*pb.User
}

// Unary RPC - простой запрос-ответ
func (s *userServer) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.User, error) {
    user, exists := s.users[req.Id]
    if !exists {
        return nil, status.Errorf(codes.NotFound, "User %d not found", req.Id)
    }
    return user, nil
}

// Server streaming - сервер возвращает поток
func (s *userServer) ListUsers(req *pb.ListUsersRequest, stream pb.UserService_ListUsersServer) error {
    for _, user := range s.users {
        if err := stream.Send(user); err != nil {
            return err
        }
    }
    return nil
}

// Client streaming - клиент отправляет поток
func (s *userServer) CreateUsers(stream pb.UserService_CreateUsersServer) error {
    count := int32(0)

    for {
        req, err := stream.Recv()
        if err == io.EOF {
            // Клиент закончил отправку
            return stream.SendAndClose(&pb.CreateUsersResponse{
                CreatedCount: count,
            })
        }
        if err != nil {
            return err
        }

        // Создаем пользователя
        user := &pb.User{
            Id:    count + 1,
            Name:  req.Name,
            Email: req.Email,
        }
        s.users[user.Id] = user
        count++
    }
}

// Bidirectional streaming - двусторонний поток
func (s *userServer) Chat(stream pb.UserService_ChatServer) error {
    for {
        msg, err := stream.Recv()
        if err == io.EOF {
            return nil
        }
        if err != nil {
            return err
        }

        // Echo message back
        if err := stream.Send(msg); err != nil {
            return err
        }
    }
}

func main() {
    // Создаем TCP listener
    lis, err := net.Listen("tcp", ":50051")
    if err != nil {
        log.Fatalf("Failed to listen: %v", err)
    }

    // Создаем gRPC сервер
    grpcServer := grpc.NewServer()

    // Регистрируем сервис
    pb.RegisterUserServiceServer(grpcServer, &userServer{
        users: make(map[int32]*pb.User),
    })

    log.Println("gRPC server listening on :50051")
    if err := grpcServer.Serve(lis); err != nil {
        log.Fatalf("Failed to serve: %v", err)
    }
}
```

### 4. Реализация клиента

```go
package main

import (
    "context"
    "io"
    "log"
    "time"

    pb "github.com/myapp/pb/user"
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"
)

func main() {
    // Подключаемся к серверу
    conn, err := grpc.Dial("localhost:50051",
        grpc.WithTransportCredentials(insecure.NewCredentials()),
    )
    if err != nil {
        log.Fatalf("Failed to connect: %v", err)
    }
    defer conn.Close()

    // Создаем клиент
    client := pb.NewUserServiceClient(conn)

    // 1. Unary RPC
    ctx := context.Background()
    user, err := client.GetUser(ctx, &pb.GetUserRequest{Id: 1})
    if err != nil {
        log.Println("GetUser error:", err)
    } else {
        log.Printf("User: %+v\n", user)
    }

    // 2. Server streaming
    stream, err := client.ListUsers(ctx, &pb.ListUsersRequest{})
    if err != nil {
        log.Fatal(err)
    }

    for {
        user, err := stream.Recv()
        if err == io.EOF {
            break
        }
        if err != nil {
            log.Fatal(err)
        }
        log.Printf("User: %+v\n", user)
    }

    // 3. Client streaming
    createStream, err := client.CreateUsers(ctx)
    if err != nil {
        log.Fatal(err)
    }

    users := []*pb.CreateUserRequest{
        {Name: "Alice", Email: "alice@example.com"},
        {Name: "Bob", Email: "bob@example.com"},
    }

    for _, u := range users {
        if err := createStream.Send(u); err != nil {
            log.Fatal(err)
        }
    }

    resp, err := createStream.CloseAndRecv()
    if err != nil {
        log.Fatal(err)
    }
    log.Printf("Created %d users\n", resp.CreatedCount)

    // 4. Bidirectional streaming
    chatStream, err := client.Chat(ctx)
    if err != nil {
        log.Fatal(err)
    }

    // Горутина для отправки сообщений
    go func() {
        for i := 0; i < 5; i++ {
            msg := &pb.ChatMessage{
                UserId:    "user123",
                Text:      fmt.Sprintf("Message %d", i),
                Timestamp: time.Now().Unix(),
            }
            if err := chatStream.Send(msg); err != nil {
                log.Fatal(err)
            }
            time.Sleep(1 * time.Second)
        }
        chatStream.CloseSend()
    }()

    // Получение сообщений
    for {
        msg, err := chatStream.Recv()
        if err == io.EOF {
            break
        }
        if err != nil {
            log.Fatal(err)
        }
        log.Printf("Received: %s\n", msg.Text)
    }
}
```

## gRPC Error Handling

### Коды ошибок

```go
import (
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
)

func (s *userServer) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.User, error) {
    if req.Id <= 0 {
        // InvalidArgument
        return nil, status.Errorf(codes.InvalidArgument, "Invalid user ID: %d", req.Id)
    }

    user, exists := s.users[req.Id]
    if !exists {
        // NotFound
        return nil, status.Errorf(codes.NotFound, "User %d not found", req.Id)
    }

    return user, nil
}

// Другие коды ошибок:
// codes.OK                - Успех
// codes.Canceled          - Запрос отменен
// codes.Unknown           - Неизвестная ошибка
// codes.InvalidArgument   - Невалидные аргументы
// codes.DeadlineExceeded  - Таймаут
// codes.NotFound          - Ресурс не найден
// codes.AlreadyExists     - Ресурс уже существует
// codes.PermissionDenied  - Нет прав
// codes.ResourceExhausted - Rate limit
// codes.Unauthenticated   - Не авторизован
// codes.Unavailable       - Сервис недоступен
// codes.Internal          - Внутренняя ошибка
```

### Клиент обрабатывает ошибки

```go
user, err := client.GetUser(ctx, &pb.GetUserRequest{Id: 999})
if err != nil {
    // Извлекаем gRPC статус
    st, ok := status.FromError(err)
    if ok {
        switch st.Code() {
        case codes.NotFound:
            log.Println("User not found:", st.Message())
        case codes.InvalidArgument:
            log.Println("Invalid argument:", st.Message())
        default:
            log.Println("Error:", st.Code(), st.Message())
        }
    } else {
        log.Println("Unknown error:", err)
    }
}
```

## Metadata (Headers)

### Сервер устанавливает metadata

```go
import "google.golang.org/grpc/metadata"

func (s *userServer) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.User, error) {
    // Устанавливаем metadata (headers)
    header := metadata.Pairs(
        "x-request-id", "abc123",
        "x-timestamp", time.Now().Format(time.RFC3339),
    )
    grpc.SetHeader(ctx, header)

    // Устанавливаем trailer (footer)
    trailer := metadata.Pairs("x-response-time", "15ms")
    grpc.SetTrailer(ctx, trailer)

    // ...
    return user, nil
}
```

### Клиент отправляет metadata

```go
// Создаем metadata
md := metadata.Pairs(
    "authorization", "Bearer "+token,
    "x-client-id", "mobile-app",
)

// Добавляем в context
ctx := metadata.NewOutgoingContext(context.Background(), md)

// Делаем запрос
user, err := client.GetUser(ctx, &pb.GetUserRequest{Id: 1})

// Получаем metadata из ответа
var header, trailer metadata.MD
user, err := client.GetUser(ctx, req,
    grpc.Header(&header),
    grpc.Trailer(&trailer),
)
log.Println("Request ID:", header.Get("x-request-id"))
```

## Interceptors (Middleware)

### Server Interceptor

```go
// Unary interceptor (для unary RPC)
func loggingInterceptor(
    ctx context.Context,
    req interface{},
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler,
) (interface{}, error) {
    start := time.Now()

    // Вызываем handler
    resp, err := handler(ctx, req)

    // Логируем
    duration := time.Since(start)
    log.Printf("Method: %s, Duration: %v, Error: %v",
        info.FullMethod, duration, err)

    return resp, err
}

// Stream interceptor (для streaming RPC)
func streamLoggingInterceptor(
    srv interface{},
    ss grpc.ServerStream,
    info *grpc.StreamServerInfo,
    handler grpc.StreamHandler,
) error {
    log.Printf("Stream method: %s", info.FullMethod)
    return handler(srv, ss)
}

// Регистрация interceptors
grpcServer := grpc.NewServer(
    grpc.UnaryInterceptor(loggingInterceptor),
    grpc.StreamInterceptor(streamLoggingInterceptor),
)
```

### Authentication Interceptor

```go
func authInterceptor(
    ctx context.Context,
    req interface{},
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler,
) (interface{}, error) {
    // Извлекаем metadata
    md, ok := metadata.FromIncomingContext(ctx)
    if !ok {
        return nil, status.Errorf(codes.Unauthenticated, "Missing metadata")
    }

    // Проверяем токен
    tokens := md.Get("authorization")
    if len(tokens) == 0 {
        return nil, status.Errorf(codes.Unauthenticated, "Missing authorization token")
    }

    token := tokens[0]
    if !validateToken(token) {
        return nil, status.Errorf(codes.Unauthenticated, "Invalid token")
    }

    // Продолжаем
    return handler(ctx, req)
}
```

### Client Interceptor

```go
func clientLoggingInterceptor(
    ctx context.Context,
    method string,
    req, reply interface{},
    cc *grpc.ClientConn,
    invoker grpc.UnaryInvoker,
    opts ...grpc.CallOption,
) error {
    start := time.Now()

    // Вызываем метод
    err := invoker(ctx, method, req, reply, cc, opts...)

    log.Printf("Method: %s, Duration: %v, Error: %v",
        method, time.Since(start), err)

    return err
}

// Регистрация
conn, err := grpc.Dial("localhost:50051",
    grpc.WithUnaryInterceptor(clientLoggingInterceptor),
    grpc.WithTransportCredentials(insecure.NewCredentials()),
)
```

## Timeouts и Deadlines

```go
// Клиент устанавливает deadline
ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
defer cancel()

user, err := client.GetUser(ctx, &pb.GetUserRequest{Id: 1})
if err != nil {
    st, ok := status.FromError(err)
    if ok && st.Code() == codes.DeadlineExceeded {
        log.Println("Request timed out")
    }
}

// Сервер проверяет deadline
func (s *userServer) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.User, error) {
    // Проверяем, не истек ли deadline
    if ctx.Err() == context.DeadlineExceeded {
        return nil, status.Errorf(codes.DeadlineExceeded, "Request timeout")
    }

    // Долгая операция с проверкой context
    select {
    case <-ctx.Done():
        return nil, status.Errorf(codes.Canceled, "Request canceled")
    case result := <-longOperation():
        return result, nil
    }
}
```

## TLS/SSL (Безопасное соединение)

### Сервер с TLS

```go
import "google.golang.org/grpc/credentials"

// Загружаем сертификаты
creds, err := credentials.NewServerTLSFromFile("cert.pem", "key.pem")
if err != nil {
    log.Fatal(err)
}

// Создаем сервер с TLS
grpcServer := grpc.NewServer(grpc.Creds(creds))
```

### Клиент с TLS

```go
// Загружаем CA certificate
creds, err := credentials.NewClientTLSFromFile("ca-cert.pem", "")
if err != nil {
    log.Fatal(err)
}

// Подключаемся с TLS
conn, err := grpc.Dial("localhost:50051", grpc.WithTransportCredentials(creds))
```

## Load Balancing

```go
// Client-side load balancing
conn, err := grpc.Dial(
    "dns:///my-service.example.com",  // DNS resolver
    grpc.WithDefaultServiceConfig(`{"loadBalancingPolicy":"round_robin"}`),
    grpc.WithTransportCredentials(insecure.NewCredentials()),
)

// Поддерживаемые политики:
// - round_robin  - по кругу
// - pick_first   - первый доступный
```

## Health Checking

```go
import "google.golang.org/grpc/health"
import healthpb "google.golang.org/grpc/health/grpc_health_v1"

// Регистрируем health service
healthServer := health.NewServer()
healthpb.RegisterHealthServer(grpcServer, healthServer)

// Устанавливаем статус
healthServer.SetServingStatus("user.UserService", healthpb.HealthCheckResponse_SERVING)

// Клиент проверяет health
healthClient := healthpb.NewHealthClient(conn)
resp, err := healthClient.Check(ctx, &healthpb.HealthCheckRequest{
    Service: "user.UserService",
})
if resp.Status == healthpb.HealthCheckResponse_SERVING {
    log.Println("Service is healthy")
}
```

## Reflection (для grpcurl, grpcui)

```go
import "google.golang.org/grpc/reflection"

// Включаем reflection (полезно для dev/testing)
reflection.Register(grpcServer)
```

**Использование grpcurl:**

```bash
# Установка
go install github.com/fullstorydev/grpcurl/cmd/grpcurl@latest

# Список сервисов
grpcurl -plaintext localhost:50051 list

# Список методов
grpcurl -plaintext localhost:50051 list user.UserService

# Вызов метода
grpcurl -plaintext -d '{"id": 1}' localhost:50051 user.UserService/GetUser
```

## REST Gateway (HTTP → gRPC)

```protobuf
// user.proto с HTTP mapping
import "google/api/annotations.proto";

service UserService {
    rpc GetUser(GetUserRequest) returns (User) {
        option (google.api.http) = {
            get: "/v1/users/{id}"
        };
    }

    rpc CreateUser(CreateUserRequest) returns (User) {
        option (google.api.http) = {
            post: "/v1/users"
            body: "*"
        };
    }
}
```

```bash
# Генерация REST gateway
protoc --grpc-gateway_out=. \
    --grpc-gateway_opt=paths=source_relative \
    user.proto
```

```go
// Запуск REST gateway
import "github.com/grpc-ecosystem/grpc-gateway/v2/runtime"

func runGateway() {
    ctx := context.Background()
    mux := runtime.NewServeMux()

    // Регистрируем gateway
    err := pb.RegisterUserServiceHandlerFromEndpoint(
        ctx, mux,
        "localhost:50051",
        []grpc.DialOption{grpc.WithInsecure()},
    )
    if err != nil {
        log.Fatal(err)
    }

    // HTTP сервер
    http.ListenAndServe(":8080", mux)
}

// Теперь можно делать HTTP запросы:
// curl http://localhost:8080/v1/users/1
```

## gRPC vs REST

| Критерий | gRPC | REST |
|----------|------|------|
| **Протокол** | HTTP/2 | HTTP/1.1 |
| **Формат** | Protobuf (binary) | JSON (text) |
| **Скорость** | ⚡ Быстрее (5-10x) | Медленнее |
| **Размер данных** | Меньше | Больше |
| **Типизация** | Строгая | Нет |
| **Streaming** | ✅ Да (bi-directional) | ❌ Нет (только SSE) |
| **Browser support** | ❌ Ограничено (нужен gRPC-web) | ✅ Да |
| **Human-readable** | ❌ Бинарный | ✅ Текстовый |
| **Генерация кода** | ✅ Да (из .proto) | ❌ Нет (вручную) |
| **Debugging** | Сложнее | Легче |
| **Кэширование** | Сложнее | Легче (HTTP cache) |
| **Использование** | Микросервисы, internal API | Публичные API, веб |

## Когда использовать gRPC

**✅ Используйте gRPC когда:**

1. **Микросервисы (internal API)**
   - Быстрая коммуникация между сервисами
   - Не нужна совместимость с браузером

2. **Высокая нагрузка**
   - Нужна максимальная производительность
   - Критична латентность

3. **Streaming**
   - Реальное время (чаты, уведомления)
   - Двусторонний поток данных

4. **Polyglot systems**
   - Разные языки программирования
   - Автоматическая генерация клиентов

5. **Строгая типизация**
   - Нужен контракт API (schema)
   - Compile-time проверки

**❌ НЕ используйте gRPC когда:**

1. **Публичные API для браузеров**
   - REST проще для браузеров
   - gRPC-web сложнее в настройке

2. **Простые CRUD операции**
   - REST достаточно
   - Overhead от gRPC не оправдан

3. **Human-readable API**
   - Нужен легкий debugging (curl, Postman)
   - JSON понятнее

## Best Practices

1. ✅ **Используйте TLS в продакшене**
2. ✅ **Добавляйте timeouts/deadlines**
3. ✅ **Логируйте через interceptors**
4. ✅ **Включайте health checks**
5. ✅ **Версионируйте .proto файлы**
6. ✅ **Используйте правильные error codes**
7. ✅ **Keep-alive для long-lived connections**
8. ✅ **Reflection для dev/testing**
9. ❌ **Не забывайте про circuit breakers**
10. ❌ **Не блокируйте streaming handlers**

## Связанные темы

- [[Protocol Buffers (protobuf)]]
- [[HTTP протокол]]
- [[REST API - Основы]]
- [[Микросервисная архитектура]]
- [[Go - Пакет net-http]]
- [[Service Mesh]]
