# GraphQL

GraphQL — язык запросов для API, разработанный Facebook. Альтернатива REST, позволяющая клиенту запрашивать только нужные данные.

## GraphQL vs REST

### REST

```bash
# Получить пользователя
GET /users/123
Response: { "id": 123, "name": "Alice", "email": "alice@example.com", "age": 30, ... }

# Получить заказы пользователя
GET /users/123/orders
Response: [{ "id": 1, "amount": 100 }, ...]

# Получить детали заказа
GET /orders/1
Response: { "id": 1, "amount": 100, "status": "paid", ... }

Проблемы:
- Over-fetching - получаем лишние данные
- Under-fetching - нужны множественные запросы (N+1 problem)
- 3 HTTP запроса для получения всех данных
```

### GraphQL

```graphql
# Один запрос для всех данных
query {
  user(id: 123) {
    id
    name
    email
    orders {
      id
      amount
      status
    }
  }
}

# Response - только запрошенные поля
{
  "data": {
    "user": {
      "id": 123,
      "name": "Alice",
      "email": "alice@example.com",
      "orders": [
        { "id": 1, "amount": 100, "status": "paid" },
        { "id": 2, "amount": 200, "status": "pending" }
      ]
    }
  }
}

Преимущества:
- Один HTTP запрос
- Только нужные данные
- Нет over-fetching
```

## Основы GraphQL

### Schema (схема)

```graphql
# Типы данных
type User {
  id: ID!           # ! = обязательное поле
  name: String!
  email: String!
  age: Int
  tags: [String!]!  # Массив строк (не может быть null)
  address: Address
  orders: [Order!]!
}

type Address {
  city: String!
  street: String!
  zipCode: String
}

type Order {
  id: ID!
  user: User!
  amount: Float!
  status: OrderStatus!
  createdAt: String!
}

enum OrderStatus {
  PENDING
  PAID
  SHIPPED
  DELIVERED
  CANCELLED
}

# Queries (чтение)
type Query {
  user(id: ID!): User
  users(limit: Int, offset: Int): [User!]!
  order(id: ID!): Order
  searchUsers(query: String!): [User!]!
}

# Mutations (изменение)
type Mutation {
  createUser(input: CreateUserInput!): User!
  updateUser(id: ID!, input: UpdateUserInput!): User!
  deleteUser(id: ID!): Boolean!
  createOrder(input: CreateOrderInput!): Order!
}

# Subscriptions (real-time)
type Subscription {
  orderCreated: Order!
  userUpdated(userId: ID!): User!
}

# Input types (для mutations)
input CreateUserInput {
  name: String!
  email: String!
  age: Int
}

input UpdateUserInput {
  name: String
  email: String
  age: Int
}

input CreateOrderInput {
  userId: ID!
  amount: Float!
}
```

### Scalar Types

```graphql
Int       # 32-bit signed integer
Float     # Floating point
String    # UTF-8 string
Boolean   # true/false
ID        # Unique identifier (string)

# Custom scalars
scalar DateTime
scalar Email
scalar URL
```

## GraphQL в Go (gqlgen)

### 1. Установка gqlgen

```bash
# Инициализация проекта
go mod init github.com/myapp

# Установка gqlgen
go get github.com/99designs/gqlgen
go run github.com/99designs/gqlgen init

# Создаст структуру:
# - graph/schema.graphqls    (GraphQL схема)
# - graph/resolver.go        (Resolvers)
# - graph/generated.go       (Сгенерированный код)
# - server.go                (HTTP сервер)
```

### 2. Определение схемы

```graphql
# graph/schema.graphqls
type User {
  id: ID!
  name: String!
  email: String!
  age: Int
  orders: [Order!]!
}

type Order {
  id: ID!
  userId: ID!
  user: User!
  amount: Float!
  status: OrderStatus!
}

enum OrderStatus {
  PENDING
  PAID
  SHIPPED
}

type Query {
  user(id: ID!): User
  users: [User!]!
  order(id: ID!): Order
}

type Mutation {
  createUser(name: String!, email: String!): User!
  createOrder(userId: ID!, amount: Float!): Order!
}

input CreateUserInput {
  name: String!
  email: String!
  age: Int
}
```

### 3. Генерация кода

```bash
# Генерирует resolvers
go run github.com/99designs/gqlgen generate
```

### 4. Реализация Resolvers

```go
// graph/resolver.go
package graph

import (
    "context"
    "fmt"
    "strconv"

    "github.com/myapp/graph/model"
)

type Resolver struct {
    users  map[string]*model.User
    orders map[string]*model.Order
}

func NewResolver() *Resolver {
    return &Resolver{
        users:  make(map[string]*model.User),
        orders: make(map[string]*model.Order),
    }
}

// Query Resolvers
func (r *queryResolver) User(ctx context.Context, id string) (*model.User, error) {
    user, exists := r.users[id]
    if !exists {
        return nil, fmt.Errorf("user %s not found", id)
    }
    return user, nil
}

func (r *queryResolver) Users(ctx context.Context) ([]*model.User, error) {
    users := make([]*model.User, 0, len(r.users))
    for _, user := range r.users {
        users = append(users, user)
    }
    return users, nil
}

func (r *queryResolver) Order(ctx context.Context, id string) (*model.Order, error) {
    order, exists := r.orders[id]
    if !exists {
        return nil, fmt.Errorf("order %s not found", id)
    }
    return order, nil
}

// Mutation Resolvers
func (r *mutationResolver) CreateUser(ctx context.Context, name string, email string) (*model.User, error) {
    id := strconv.Itoa(len(r.users) + 1)

    user := &model.User{
        ID:    id,
        Name:  name,
        Email: email,
    }

    r.users[id] = user
    return user, nil
}

func (r *mutationResolver) CreateOrder(ctx context.Context, userID string, amount float64) (*model.Order, error) {
    // Проверяем существование пользователя
    _, exists := r.users[userID]
    if !exists {
        return nil, fmt.Errorf("user %s not found", userID)
    }

    id := strconv.Itoa(len(r.orders) + 1)

    order := &model.Order{
        ID:     id,
        UserID: userID,
        Amount: amount,
        Status: model.OrderStatusPending,
    }

    r.orders[id] = order
    return order, nil
}

// Field Resolvers (для вложенных данных)
func (r *orderResolver) User(ctx context.Context, obj *model.Order) (*model.User, error) {
    // Загружаем User для Order
    user, exists := r.users[obj.UserID]
    if !exists {
        return nil, fmt.Errorf("user %s not found", obj.UserID)
    }
    return user, nil
}

func (r *userResolver) Orders(ctx context.Context, obj *model.User) ([]*model.Order, error) {
    // Загружаем Orders для User
    orders := make([]*model.Order, 0)
    for _, order := range r.orders {
        if order.UserID == obj.ID {
            orders = append(orders, order)
        }
    }
    return orders, nil
}

// Методы для gqlgen
func (r *Resolver) Query() QueryResolver       { return &queryResolver{r} }
func (r *Resolver) Mutation() MutationResolver { return &mutationResolver{r} }
func (r *Resolver) Order() OrderResolver       { return &orderResolver{r} }
func (r *Resolver) User() UserResolver         { return &userResolver{r} }

type queryResolver struct{ *Resolver }
type mutationResolver struct{ *Resolver }
type orderResolver struct{ *Resolver }
type userResolver struct{ *Resolver }
```

### 5. HTTP Server

```go
// server.go
package main

import (
    "log"
    "net/http"

    "github.com/99designs/gqlgen/graphql/handler"
    "github.com/99designs/gqlgen/graphql/playground"
    "github.com/myapp/graph"
)

func main() {
    resolver := graph.NewResolver()

    // GraphQL сервер
    srv := handler.NewDefaultServer(graph.NewExecutableSchema(graph.Config{
        Resolvers: resolver,
    }))

    // GraphQL Playground (UI для тестирования)
    http.Handle("/", playground.Handler("GraphQL Playground", "/query"))
    http.Handle("/query", srv)

    log.Println("Server is running on http://localhost:8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

### 6. Запуск и тестирование

```bash
# Запуск сервера
go run server.go

# Открываем браузер
# http://localhost:8080
# GraphQL Playground UI
```

**Примеры запросов:**

```graphql
# Query - получить пользователя
query {
  user(id: "1") {
    id
    name
    email
    orders {
      id
      amount
      status
    }
  }
}

# Mutation - создать пользователя
mutation {
  createUser(name: "Alice", email: "alice@example.com") {
    id
    name
    email
  }
}

# Mutation - создать заказ
mutation {
  createOrder(userId: "1", amount: 100.50) {
    id
    amount
    status
    user {
      name
    }
  }
}
```

## Запросы (Queries)

### Базовый запрос

```graphql
query {
  user(id: "123") {
    id
    name
    email
  }
}
```

### Запрос с алиасами

```graphql
query {
  alice: user(id: "1") {
    name
    email
  }
  bob: user(id: "2") {
    name
    email
  }
}
```

### Запрос с переменными

```graphql
query GetUser($userId: ID!) {
  user(id: $userId) {
    id
    name
    email
    orders {
      id
      amount
    }
  }
}

# Variables (JSON)
{
  "userId": "123"
}
```

### Fragments (переиспользуемые части)

```graphql
fragment UserFields on User {
  id
  name
  email
}

query {
  user1: user(id: "1") {
    ...UserFields
  }
  user2: user(id: "2") {
    ...UserFields
  }
}
```

### Inline Fragments

```graphql
query {
  search(query: "Alice") {
    __typename
    ... on User {
      id
      name
      email
    }
    ... on Order {
      id
      amount
    }
  }
}
```

## Mutations

```graphql
# Создание пользователя
mutation {
  createUser(name: "Alice", email: "alice@example.com") {
    id
    name
    email
  }
}

# Обновление пользователя
mutation {
  updateUser(id: "123", input: {name: "Alice Updated"}) {
    id
    name
  }
}

# Удаление пользователя
mutation {
  deleteUser(id: "123")
}

# Множественные mutations
mutation {
  user1: createUser(name: "Alice", email: "alice@example.com") {
    id
  }
  user2: createUser(name: "Bob", email: "bob@example.com") {
    id
  }
}
```

## Subscriptions (Real-time)

```go
// Добавляем Subscription resolver
func (r *subscriptionResolver) OrderCreated(ctx context.Context) (<-chan *model.Order, error) {
    ch := make(chan *model.Order, 1)

    go func() {
        // Симуляция событий
        for {
            select {
            case <-ctx.Done():
                close(ch)
                return
            case order := <-r.orderChannel:
                ch <- order
            }
        }
    }()

    return ch, nil
}
```

**Клиент (WebSocket):**

```graphql
subscription {
  orderCreated {
    id
    amount
    status
    user {
      name
    }
  }
}
```

## DataLoader (N+1 Problem)

Проблема N+1:

```go
// Плохо - N+1 queries
func (r *orderResolver) User(ctx context.Context, obj *model.Order) (*model.User, error) {
    // Вызывается для каждого Order
    // 1 query для orders + N queries для users
    return r.db.GetUser(obj.UserID)
}
```

Решение с DataLoader:

```go
import "github.com/graph-gophers/dataloader"

// Batch функция - загружает много users за раз
func batchGetUsers(ctx context.Context, keys []string) []*dataloader.Result {
    users, err := db.GetUsersByIDs(keys) // Один запрос для всех IDs

    results := make([]*dataloader.Result, len(keys))
    for i, key := range keys {
        if err != nil {
            results[i] = &dataloader.Result{Error: err}
        } else {
            results[i] = &dataloader.Result{Data: users[key]}
        }
    }
    return results
}

// Создаем DataLoader
userLoader := dataloader.NewBatchedLoader(batchGetUsers)

// Используем в resolver
func (r *orderResolver) User(ctx context.Context, obj *model.Order) (*model.User, error) {
    // DataLoader автоматически батчит запросы
    thunk := userLoader.Load(ctx, dataloader.StringKey(obj.UserID))
    result, err := thunk()
    if err != nil {
        return nil, err
    }
    return result.(*model.User), nil
}
```

## Аутентификация и авторизация

```go
// Middleware для аутентификации
func authMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")

        user, err := validateToken(token)
        if err != nil {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }

        // Добавляем user в context
        ctx := context.WithValue(r.Context(), "user", user)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// Использование в resolver
func (r *mutationResolver) CreateOrder(ctx context.Context, input model.CreateOrderInput) (*model.Order, error) {
    // Извлекаем user из context
    user, ok := ctx.Value("user").(*model.User)
    if !ok {
        return nil, fmt.Errorf("unauthorized")
    }

    // Проверяем права
    if !user.CanCreateOrder() {
        return nil, fmt.Errorf("permission denied")
    }

    // Создаем заказ
    return r.orderService.CreateOrder(input)
}

// Регистрация middleware
http.Handle("/query", authMiddleware(srv))
```

## Directives (директивы)

```graphql
# Схема с директивами
directive @auth(requires: Role = USER) on FIELD_DEFINITION

enum Role {
  USER
  ADMIN
}

type Query {
  user(id: ID!): User
  users: [User!]! @auth(requires: ADMIN)
  adminPanel: AdminPanel! @auth(requires: ADMIN)
}

type Mutation {
  deleteUser(id: ID!): Boolean! @auth(requires: ADMIN)
}
```

**Реализация:**

```go
// Проверка директивы @auth
func authDirective(ctx context.Context, obj interface{}, next graphql.Resolver, requires model.Role) (interface{}, error) {
    user, ok := ctx.Value("user").(*model.User)
    if !ok {
        return nil, fmt.Errorf("unauthorized")
    }

    if user.Role < requires {
        return nil, fmt.Errorf("insufficient permissions")
    }

    return next(ctx)
}
```

## Error Handling

```go
import (
    "github.com/vektah/gqlparser/v2/gqlerror"
)

func (r *queryResolver) User(ctx context.Context, id string) (*model.User, error) {
    user, err := r.db.GetUser(id)
    if err != nil {
        // Возвращаем GraphQL ошибку
        return nil, &gqlerror.Error{
            Message: "User not found",
            Extensions: map[string]interface{}{
                "code":   "USER_NOT_FOUND",
                "userId": id,
            },
        }
    }
    return user, nil
}
```

**Response с ошибкой:**

```json
{
  "data": {
    "user": null
  },
  "errors": [
    {
      "message": "User not found",
      "path": ["user"],
      "extensions": {
        "code": "USER_NOT_FOUND",
        "userId": "123"
      }
    }
  ]
}
```

## Pagination

### Offset-based

```graphql
type Query {
  users(offset: Int, limit: Int): [User!]!
}

query {
  users(offset: 0, limit: 10) {
    id
    name
  }
}
```

### Cursor-based (Relay style)

```graphql
type UserConnection {
  edges: [UserEdge!]!
  pageInfo: PageInfo!
}

type UserEdge {
  node: User!
  cursor: String!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}

type Query {
  users(first: Int, after: String): UserConnection!
}

query {
  users(first: 10, after: "cursor123") {
    edges {
      node {
        id
        name
      }
      cursor
    }
    pageInfo {
      hasNextPage
      endCursor
    }
  }
}
```

## GraphQL Client (в Go)

```go
import "github.com/machinebox/graphql"

// Создаем клиент
client := graphql.NewClient("http://localhost:8080/query")

// Query
req := graphql.NewRequest(`
    query ($userId: ID!) {
        user(id: $userId) {
            id
            name
            email
        }
    }
`)

req.Var("userId", "123")
req.Header.Set("Authorization", "Bearer "+token)

var respData struct {
    User struct {
        ID    string
        Name  string
        Email string
    }
}

if err := client.Run(context.Background(), req, &respData); err != nil {
    log.Fatal(err)
}

log.Printf("User: %+v", respData.User)

// Mutation
mutReq := graphql.NewRequest(`
    mutation ($name: String!, $email: String!) {
        createUser(name: $name, email: $email) {
            id
            name
        }
    }
`)

mutReq.Var("name", "Alice")
mutReq.Var("email", "alice@example.com")

var mutResp struct {
    CreateUser struct {
        ID   string
        Name string
    }
}

client.Run(context.Background(), mutReq, &mutResp)
```

## GraphQL vs REST vs gRPC

| Критерий | GraphQL | REST | gRPC |
|----------|---------|------|------|
| **Запросы** | Гибкие | Фиксированные endpoints | Фиксированные methods |
| **Over-fetching** | ✅ Нет | ❌ Да | ❌ Да |
| **Under-fetching** | ✅ Нет (один запрос) | ❌ Да (N+1) | ❌ Да (N+1) |
| **Формат** | JSON | JSON/XML | Protobuf (binary) |
| **Типизация** | ✅ Строгая (schema) | ❌ Нет | ✅ Строгая (.proto) |
| **Скорость** | Средняя | Медленная | ⚡ Быстрая |
| **Размер данных** | Средний | Большой | Маленький |
| **Кэширование** | Сложное | ✅ Легкое (HTTP cache) | Сложное |
| **Real-time** | ✅ Subscriptions | SSE | ✅ Streaming |
| **Learning curve** | Высокая | Низкая | Средняя |
| **Tooling** | Хорошее | Отличное | Хорошее |

## Когда использовать GraphQL

**✅ Используйте GraphQL когда:**

1. **Разнообразные клиенты**
   - Mobile, Web, Desktop
   - Разные требования к данным

2. **Over-fetching проблема**
   - REST возвращает слишком много данных
   - Нужны только определенные поля

3. **Under-fetching проблема**
   - Нужно много связанных данных
   - N+1 запросов в REST

4. **Частое изменение требований**
   - Клиенты хотят разные данные
   - Гибкость запросов

5. **Real-time updates**
   - WebSocket subscriptions

**❌ НЕ используйте GraphQL когда:**

1. **Простые CRUD операции**
   - REST достаточно

2. **Публичные API**
   - REST проще для внешних разработчиков
   - HTTP кэширование важно

3. **Микросервисы (internal)**
   - gRPC быстрее

4. **File uploads/downloads**
   - REST/HTTP проще

## Best Practices

1. ✅ **Используйте DataLoader** - решает N+1 problem
2. ✅ **Пагинация обязательна** для списков
3. ✅ **Аутентификация через middleware**
4. ✅ **Авторизация через директивы**
5. ✅ **Версионирование не нужно** (просто добавляйте поля)
6. ✅ **Используйте nullable с умом**
7. ✅ **Документируйте schema**
8. ❌ **Не злоупотребляйте вложенностью** (глубина > 3-4)
9. ❌ **Не забывайте про rate limiting**
10. ❌ **Не возвращайте все данные сразу** (pagination!)

## Связанные темы

- [[REST API - Основы]]
- [[REST API - Дизайн и best practices]]
- [[HTTP протокол]]
- [[gRPC]]
- [[Микросервисная архитектура]]
- [[Go - Пакет net-http]]
