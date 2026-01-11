# OpenAPI Specification Example

Пример OpenAPI 3.0 спецификации для генерации Go-кода.

## Файл спецификации

Расположение: `examples/openapi.yaml`

## Описание API

Демонстрационное REST API для управления пользователями:

### Endpoints

| Метод | Путь | Описание |
|-------|------|----------|
| POST | `/auth/login` | Авторизация |
| POST | `/auth/refresh` | Обновление токена |
| GET | `/users` | Список пользователей (пагинация, фильтры) |
| POST | `/users` | Создание пользователя |
| GET | `/users/{id}` | Получение пользователя |
| PUT | `/users/{id}` | Обновление пользователя |
| DELETE | `/users/{id}` | Удаление пользователя |
| POST | `/users/{id}/avatar` | Загрузка аватара |

### Модели данных

- `User` - пользователь с полями id, email, name, status
- `UserStatus` - enum статусов (active, inactive, pending, banned)
- `LoginRequest` / `LoginResponse` - авторизация
- `Error` / `ValidationError` - ошибки

### Особенности примера

1. **JWT аутентификация** - Bearer token в заголовке
2. **Пагинация** - limit, offset параметры
3. **Фильтрация и поиск** - query параметры
4. **Валидация** - minLength, maxLength, format
5. **Загрузка файлов** - multipart/form-data
6. **Переиспользуемые компоненты** - responses, schemas

## Генерация кода

```bash
# Установка
go install github.com/oapi-codegen/oapi-codegen/v2/cmd/oapi-codegen@latest

# Генерация сервера (Chi)
oapi-codegen -generate types,chi-server,strict-server \
  -package api \
  -o internal/api/api.gen.go \
  examples/openapi.yaml

# Генерация клиента
oapi-codegen -generate types,client \
  -package client \
  -o internal/client/client.gen.go \
  examples/openapi.yaml
```

## Связанные заметки

- [[Go - Генерация кода (go generate)]] - подробное руководство по генерации
- [[REST API - Основы]]
- [[REST API - Дизайн и best practices]]
- [[JWT (JSON Web Tokens)]]
