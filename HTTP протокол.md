# HTTP протокол

HTTP (HyperText Transfer Protocol) — протокол прикладного уровня для передачи данных в сети. Основа веб-коммуникаций и REST API.

## Основы HTTP

### Структура HTTP запроса

```
GET /users/123 HTTP/1.1
Host: api.example.com
User-Agent: Mozilla/5.0
Accept: application/json
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json

{"name": "Alice"}
```

**Компоненты:**
1. **Request Line:** `METHOD /path HTTP/version`
2. **Headers:** Метаданные запроса
3. **Пустая строка** (разделитель)
4. **Body:** Тело запроса (опционально)

### Структура HTTP ответа

```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 58
Date: Thu, 09 Jan 2026 10:00:00 GMT
Server: nginx/1.18.0

{"id": 123, "name": "Alice", "email": "alice@example.com"}
```

**Компоненты:**
1. **Status Line:** `HTTP/version status_code status_text`
2. **Headers:** Метаданные ответа
3. **Пустая строка**
4. **Body:** Тело ответа

## HTTP Методы

### GET - Получение ресурса

```go
// Получить пользователя
resp, err := http.Get("https://api.example.com/users/123")
if err != nil {
    log.Fatal(err)
}
defer resp.Body.Close()

body, _ := io.ReadAll(resp.Body)
fmt.Println(string(body))
```

**Характеристики:**
- ✅ Идемпотентный
- ✅ Безопасный (не изменяет данные)
- ✅ Кэшируемый
- ❌ Нет body в запросе (данные в URL query)

### POST - Создание ресурса

```go
user := User{Name: "Alice", Email: "alice@example.com"}
jsonData, _ := json.Marshal(user)

resp, err := http.Post(
    "https://api.example.com/users",
    "application/json",
    bytes.NewBuffer(jsonData),
)
```

**Характеристики:**
- ❌ НЕ идемпотентный (каждый вызов создает новый ресурс)
- ❌ НЕ безопасный
- ❌ НЕ кэшируемый

### PUT - Полная замена ресурса

```go
user := User{ID: 123, Name: "Alice Updated", Email: "alice@example.com"}
jsonData, _ := json.Marshal(user)

req, _ := http.NewRequest(
    "PUT",
    "https://api.example.com/users/123",
    bytes.NewBuffer(jsonData),
)
req.Header.Set("Content-Type", "application/json")

client := &http.Client{}
resp, err := client.Do(req)
```

**Характеристики:**
- ✅ Идемпотентный
- ❌ НЕ безопасный
- ❌ НЕ кэшируемый

### PATCH - Частичное обновление

```go
updates := map[string]string{"name": "Alice Updated"}
jsonData, _ := json.Marshal(updates)

req, _ := http.NewRequest(
    "PATCH",
    "https://api.example.com/users/123",
    bytes.NewBuffer(jsonData),
)
req.Header.Set("Content-Type", "application/json")

client := &http.Client{}
resp, err := client.Do(req)
```

### DELETE - Удаление ресурса

```go
req, _ := http.NewRequest("DELETE", "https://api.example.com/users/123", nil)

client := &http.Client{}
resp, err := client.Do(req)
```

**Характеристики:**
- ✅ Идемпотентный
- ❌ НЕ безопасный
- ❌ НЕ кэшируемый

### HEAD - Метаданные без body

```go
// Получить только headers (без body)
resp, err := http.Head("https://api.example.com/users/123")

contentLength := resp.Header.Get("Content-Length")
lastModified := resp.Header.Get("Last-Modified")
```

**Использование:**
- Проверка существования ресурса
- Получение метаданных (размер файла, дата изменения)

### OPTIONS - Разрешенные методы

```go
req, _ := http.NewRequest("OPTIONS", "https://api.example.com/users/123", nil)
resp, _ := client.Do(req)

allowedMethods := resp.Header.Get("Allow")
// "GET, POST, PUT, DELETE, OPTIONS"
```

**Использование:**
- CORS preflight requests
- Определение доступных методов

## Коды состояния (Status Codes)

### 1xx - Информационные

```
100 Continue      - Сервер получил заголовки, клиент может продолжить
101 Switching Protocols - Переключение на WebSocket
```

### 2xx - Успешные

```go
200 OK            // Успех
201 Created       // Ресурс создан
202 Accepted      // Запрос принят (асинхронная обработка)
204 No Content    // Успех без тела ответа
206 Partial Content // Частичный контент (Range requests)
```

### 3xx - Перенаправление

```go
301 Moved Permanently   // Постоянное перенаправление
302 Found               // Временное перенаправление
304 Not Modified        // Кэшированная версия актуальна
307 Temporary Redirect  // Временное перенаправление (сохраняет метод)
308 Permanent Redirect  // Постоянное перенаправление (сохраняет метод)
```

**Обработка редиректов в Go:**

```go
client := &http.Client{
    CheckRedirect: func(req *http.Request, via []*http.Request) error {
        if len(via) >= 10 {
            return errors.New("too many redirects")
        }
        return nil
    },
}

resp, err := client.Get("https://example.com/redirect")
```

### 4xx - Клиентские ошибки

```go
400 Bad Request          // Невалидный запрос
401 Unauthorized         // Требуется аутентификация
403 Forbidden            // Доступ запрещен
404 Not Found            // Ресурс не найден
405 Method Not Allowed   // Метод не разрешен
408 Request Timeout      // Таймаут запроса
409 Conflict             // Конфликт (например, дубликат email)
410 Gone                 // Ресурс удален навсегда
422 Unprocessable Entity // Невалидные данные
429 Too Many Requests    // Rate limit exceeded
```

### 5xx - Серверные ошибки

```go
500 Internal Server Error  // Общая ошибка сервера
502 Bad Gateway            // Proxy получил невалидный ответ
503 Service Unavailable    // Сервис временно недоступен
504 Gateway Timeout        // Proxy timeout
```

## HTTP Headers

### Request Headers

```go
req, _ := http.NewRequest("GET", "https://api.example.com/users", nil)

// Content negotiation
req.Header.Set("Accept", "application/json")
req.Header.Set("Accept-Language", "ru-RU,en;q=0.9")
req.Header.Set("Accept-Encoding", "gzip, deflate")

// Authentication
req.Header.Set("Authorization", "Bearer "+token)

// Content type (для POST/PUT)
req.Header.Set("Content-Type", "application/json")

// Custom headers
req.Header.Set("X-Request-ID", uuid.New().String())
req.Header.Set("User-Agent", "MyApp/1.0")

// Idempotency
req.Header.Set("Idempotency-Key", "abc123")

// Cache control
req.Header.Set("Cache-Control", "no-cache")
req.Header.Set("If-None-Match", `"etag-value"`)
```

### Response Headers

```go
func handler(w http.ResponseWriter, r *http.Request) {
    // Content type
    w.Header().Set("Content-Type", "application/json")

    // Cache control
    w.Header().Set("Cache-Control", "max-age=3600")
    w.Header().Set("ETag", `"abc123"`)
    w.Header().Set("Last-Modified", time.Now().Format(http.TimeFormat))

    // CORS
    w.Header().Set("Access-Control-Allow-Origin", "*")
    w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE")
    w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")

    // Security headers
    w.Header().Set("X-Content-Type-Options", "nosniff")
    w.Header().Set("X-Frame-Options", "DENY")
    w.Header().Set("Strict-Transport-Security", "max-age=31536000")

    // Rate limiting
    w.Header().Set("X-RateLimit-Limit", "100")
    w.Header().Set("X-RateLimit-Remaining", "95")
    w.Header().Set("X-RateLimit-Reset", "1609459200")

    // Location (для 201 Created)
    w.Header().Set("Location", "/users/123")

    // Retry-After (для 429, 503)
    w.Header().Set("Retry-After", "60")

    w.WriteHeader(http.StatusOK)
    json.NewEncoder(w).Encode(data)
}
```

## Кэширование

### Cache-Control

```go
// Кэшировать на 1 час
w.Header().Set("Cache-Control", "public, max-age=3600")

// Не кэшировать
w.Header().Set("Cache-Control", "no-store")

// Перепроверять перед использованием кэша
w.Header().Set("Cache-Control", "no-cache")

// Частный кэш (только в браузере)
w.Header().Set("Cache-Control", "private, max-age=300")
```

### ETag (Entity Tag)

```go
// Сервер генерирует ETag
func handler(w http.ResponseWriter, r *http.Request) {
    data := getData()

    // Вычисляем ETag (хэш данных)
    etag := fmt.Sprintf(`"%x"`, md5.Sum([]byte(data)))

    // Проверяем If-None-Match
    if r.Header.Get("If-None-Match") == etag {
        // Данные не изменились
        w.WriteHeader(http.StatusNotModified)
        return
    }

    // Возвращаем данные с ETag
    w.Header().Set("ETag", etag)
    w.Header().Set("Cache-Control", "max-age=3600")
    w.Write([]byte(data))
}

// Клиент при повторном запросе отправляет ETag
req.Header.Set("If-None-Match", etag)
```

### Last-Modified

```go
func handler(w http.ResponseWriter, r *http.Request) {
    data, modifiedTime := getData()

    // Проверяем If-Modified-Since
    ifModifiedSince := r.Header.Get("If-Modified-Since")
    if ifModifiedSince != "" {
        clientTime, _ := time.Parse(http.TimeFormat, ifModifiedSince)
        if modifiedTime.Before(clientTime) || modifiedTime.Equal(clientTime) {
            w.WriteHeader(http.StatusNotModified)
            return
        }
    }

    // Возвращаем данные с Last-Modified
    w.Header().Set("Last-Modified", modifiedTime.Format(http.TimeFormat))
    w.Header().Set("Cache-Control", "max-age=3600")
    w.Write([]byte(data))
}
```

## Сжатие (Compression)

### Gzip Compression

```go
import "compress/gzip"

func GzipMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Проверяем, поддерживает ли клиент gzip
        if !strings.Contains(r.Header.Get("Accept-Encoding"), "gzip") {
            next.ServeHTTP(w, r)
            return
        }

        // Оборачиваем ResponseWriter
        w.Header().Set("Content-Encoding", "gzip")
        gz := gzip.NewWriter(w)
        defer gz.Close()

        gzipWriter := &gzipResponseWriter{Writer: gz, ResponseWriter: w}
        next.ServeHTTP(gzipWriter, r)
    })
}

type gzipResponseWriter struct {
    io.Writer
    http.ResponseWriter
}

func (w *gzipResponseWriter) Write(b []byte) (int, error) {
    return w.Writer.Write(b)
}
```

## Range Requests (частичная загрузка)

```go
// Клиент запрашивает байты 0-999 из файла
req.Header.Set("Range", "bytes=0-999")

// Сервер отвечает
w.Header().Set("Content-Range", "bytes 0-999/5000") // 0-999 из 5000
w.Header().Set("Content-Length", "1000")
w.WriteHeader(http.StatusPartialContent) // 206
```

**Использование:**
- Докачка файлов
- Стриминг видео
- Параллельная загрузка частей файла

## Chunked Transfer Encoding

```go
// Сервер отправляет данные частями (без знания полного размера)
func handler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Transfer-Encoding", "chunked")

    flusher, ok := w.(http.Flusher)
    if !ok {
        http.Error(w, "Streaming not supported", http.StatusInternalServerError)
        return
    }

    for i := 0; i < 10; i++ {
        fmt.Fprintf(w, "Chunk %d\n", i)
        flusher.Flush() // Отправляем данные немедленно
        time.Sleep(1 * time.Second)
    }
}
```

## Keep-Alive (Connection Reuse)

```go
// HTTP/1.1 Keep-Alive включен по умолчанию
client := &http.Client{
    Transport: &http.Transport{
        MaxIdleConns:        100,              // Максимум idle соединений
        MaxIdleConnsPerHost: 10,               // Максимум idle соединений на хост
        IdleConnTimeout:     90 * time.Second, // Таймаут idle соединения
    },
}

// Несколько запросов используют одно TCP соединение
for i := 0; i < 10; i++ {
    resp, _ := client.Get("https://api.example.com/users")
    resp.Body.Close()
}
```

## Timeouts

```go
client := &http.Client{
    Timeout: 10 * time.Second, // Общий таймаут (dial + request + response)

    Transport: &http.Transport{
        DialContext: (&net.Dialer{
            Timeout:   5 * time.Second,  // Таймаут установки соединения
            KeepAlive: 30 * time.Second,
        }).DialContext,

        TLSHandshakeTimeout:   5 * time.Second,  // Таймаут TLS handshake
        ResponseHeaderTimeout: 5 * time.Second,  // Таймаут получения заголовков
        ExpectContinueTimeout: 1 * time.Second,
    },
}

// Таймаут через context
ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
defer cancel()

req, _ := http.NewRequestWithContext(ctx, "GET", "https://api.example.com/users", nil)
resp, err := client.Do(req)
```

## CORS (Cross-Origin Resource Sharing)

```go
func CORSMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Разрешаем запросы с любого origin
        w.Header().Set("Access-Control-Allow-Origin", "*")

        // Разрешенные методы
        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")

        // Разрешенные headers
        w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")

        // Credentials (cookies, HTTP auth)
        w.Header().Set("Access-Control-Allow-Credentials", "true")

        // Max age для preflight cache
        w.Header().Set("Access-Control-Max-Age", "3600")

        // Preflight request
        if r.Method == "OPTIONS" {
            w.WriteHeader(http.StatusNoContent)
            return
        }

        next.ServeHTTP(w, r)
    })
}
```

**Preflight request:**

```
OPTIONS /users HTTP/1.1
Origin: https://example.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Content-Type

→ Response:
Access-Control-Allow-Origin: https://example.com
Access-Control-Allow-Methods: POST, GET
Access-Control-Allow-Headers: Content-Type
Access-Control-Max-Age: 3600
```

## Content Types

### Common Content Types

```go
// JSON
w.Header().Set("Content-Type", "application/json")
json.NewEncoder(w).Encode(data)

// XML
w.Header().Set("Content-Type", "application/xml")
xml.NewEncoder(w).Encode(data)

// HTML
w.Header().Set("Content-Type", "text/html; charset=utf-8")
w.Write([]byte("<html><body>Hello</body></html>"))

// Plain text
w.Header().Set("Content-Type", "text/plain")
w.Write([]byte("Hello, World!"))

// Binary data
w.Header().Set("Content-Type", "application/octet-stream")
w.Write(binaryData)

// File download
w.Header().Set("Content-Type", "application/pdf")
w.Header().Set("Content-Disposition", "attachment; filename=report.pdf")
w.Write(pdfData)

// Multipart form data (file upload)
w.Header().Set("Content-Type", "multipart/form-data; boundary=----FormBoundary")
```

## Multipart File Upload

```go
// Сервер принимает файл
func UploadFile(w http.ResponseWriter, r *http.Request) {
    // Лимит размера (10MB)
    r.ParseMultipartForm(10 << 20)

    file, handler, err := r.FormFile("file")
    if err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }
    defer file.Close()

    // Создаем файл на диске
    dst, err := os.Create("/uploads/" + handler.Filename)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    defer dst.Close()

    // Копируем данные
    io.Copy(dst, file)

    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(map[string]string{
        "message": "File uploaded successfully",
        "filename": handler.Filename,
    })
}

// Клиент отправляет файл
func uploadFile() {
    file, _ := os.Open("document.pdf")
    defer file.Close()

    body := &bytes.Buffer{}
    writer := multipart.NewWriter(body)

    part, _ := writer.CreateFormFile("file", "document.pdf")
    io.Copy(part, file)

    writer.Close()

    req, _ := http.NewRequest("POST", "https://api.example.com/upload", body)
    req.Header.Set("Content-Type", writer.FormDataContentType())

    client := &http.Client{}
    resp, _ := client.Do(req)
}
```

## HTTP/1.1 vs HTTP/2 vs HTTP/3

### HTTP/1.1

```
- Один запрос на одно TCP соединение
- Keep-Alive для переиспользования соединения
- Head-of-line blocking
- Text-based protocol
```

### HTTP/2

```go
// HTTP/2 включен автоматически в Go для HTTPS
server := &http.Server{
    Addr: ":443",
    TLSConfig: &tls.Config{
        NextProtos: []string{"h2", "http/1.1"}, // HTTP/2 + fallback
    },
}

server.ListenAndServeTLS("cert.pem", "key.pem")
```

**Преимущества HTTP/2:**
- ✅ Multiplexing - несколько запросов на одно соединение
- ✅ Server Push - сервер отправляет ресурсы до запроса
- ✅ Header compression (HPACK)
- ✅ Binary protocol (быстрее парсинг)

### HTTP/3 (QUIC)

- Работает через UDP вместо TCP
- Еще быстрее установка соединения
- Лучше работает на нестабильных сетях

## Server-Sent Events (SSE)

```go
// Сервер отправляет события клиенту
func EventsHandler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "text/event-stream")
    w.Header().Set("Cache-Control", "no-cache")
    w.Header().Set("Connection", "keep-alive")

    flusher, _ := w.(http.Flusher)

    for i := 0; i < 10; i++ {
        fmt.Fprintf(w, "data: Event %d\n\n", i)
        flusher.Flush()
        time.Sleep(1 * time.Second)
    }
}

// Клиент (JavaScript)
const eventSource = new EventSource('/events');
eventSource.onmessage = function(event) {
    console.log('New event:', event.data);
};
```

## WebSocket (upgrade from HTTP)

```go
import "github.com/gorilla/websocket"

var upgrader = websocket.Upgrader{
    CheckOrigin: func(r *http.Request) bool {
        return true // Allow all origins
    },
}

func WebSocketHandler(w http.ResponseWriter, r *http.Request) {
    // Upgrade HTTP → WebSocket
    conn, err := upgrader.Upgrade(w, r, nil)
    if err != nil {
        log.Println(err)
        return
    }
    defer conn.Close()

    // Двусторонняя коммуникация
    for {
        messageType, message, err := conn.ReadMessage()
        if err != nil {
            break
        }

        // Echo message back
        err = conn.WriteMessage(messageType, message)
        if err != nil {
            break
        }
    }
}
```

## Best Practices

### ✅ DO (Делайте)

1. **Используйте правильные HTTP методы**
2. **Возвращайте правильные коды ответов**
3. **Используйте HTTPS в продакшене**
4. **Включайте кэширование** (Cache-Control, ETag)
5. **Используйте сжатие** (gzip)
6. **Настраивайте timeouts**
7. **Включайте CORS headers** для публичных API
8. **Логируйте запросы** (метод, URL, status, latency)
9. **Переиспользуйте соединения** (Keep-Alive)
10. **Используйте HTTP/2** для HTTPS

### ❌ DON'T (Не делайте)

1. ❌ **Не используйте GET для изменения данных**
2. ❌ **Не игнорируйте status codes** (всегда 200)
3. ❌ **Не передавайте чувствительные данные в URL**
4. ❌ **Не используйте HTTP для production** (только HTTPS)
5. ❌ **Не забывайте закрывать body** (resp.Body.Close())
6. ❌ **Не блокируйте горутины на чтение body**
7. ❌ **Не игнорируйте Content-Type**

## Связанные темы

- [[REST API - Основы]]
- [[REST API - Дизайн и best practices]]
- [[gRPC]]
- [[HTTPS и TLS]]
- [[Go - Пакет net-http]]
- [[OAuth 2.0]]
- [[Безопасность API]]
