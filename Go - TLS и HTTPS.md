# Go - TLS и HTTPS

Работа с защищенными соединениями TLS/HTTPS в Go.

## HTTPS Server

### Базовый HTTPS сервер

```go
func main() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Hello, HTTPS!")
    })

    // Запуск HTTPS сервера
    log.Fatal(http.ListenAndServeTLS(
        ":443",
        "cert.pem",    // Сертификат
        "key.pem",     // Приватный ключ
        nil,
    ))
}
```

### Генерация self-signed сертификата

```bash
# Для разработки
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes
```

### Конфигурация TLS

```go
tlsConfig := &tls.Config{
    MinVersion:               tls.VersionTLS12,
    CurvePreferences:         []tls.CurveID{tls.CurveP521, tls.CurveP384, tls.CurveP256},
    PreferServerCipherSuites: true,
    CipherSuites: []uint16{
        tls.TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,
        tls.TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA,
        tls.TLS_RSA_WITH_AES_256_GCM_SHA384,
        tls.TLS_RSA_WITH_AES_256_CBC_SHA,
    },
}

server := &http.Server{
    Addr:      ":443",
    TLSConfig: tlsConfig,
}

log.Fatal(server.ListenAndServeTLS("cert.pem", "key.pem"))
```

## HTTPS Client

### Простой GET запрос

```go
resp, err := http.Get("https://example.com")
if err != nil {
    log.Fatal(err)
}
defer resp.Body.Close()

body, _ := io.ReadAll(resp.Body)
fmt.Println(string(body))
```

### Client с кастомной конфигурацией

```go
// Отключить проверку сертификатов (только для dev!)
tr := &http.Transport{
    TLSClientConfig: &tls.Config{InsecureSkipVerify: true},
}

client := &http.Client{Transport: tr}
resp, err := client.Get("https://self-signed.example.com")
```

### Client с custom CA

```go
// Загрузить CA сертификат
caCert, err := os.ReadFile("ca.pem")
if err != nil {
    log.Fatal(err)
}

caCertPool := x509.NewCertPool()
caCertPool.AppendCertsFromPEM(caCert)

// Конфигурация TLS
tlsConfig := &tls.Config{
    RootCAs: caCertPool,
}

transport := &http.Transport{TLSClientConfig: tlsConfig}
client := &http.Client{Transport: transport}

resp, err := client.Get("https://example.com")
```

### Mutual TLS (mTLS)

```go
// Загрузить клиентский сертификат
cert, err := tls.LoadX509KeyPair("client-cert.pem", "client-key.pem")
if err != nil {
    log.Fatal(err)
}

// Загрузить CA
caCert, _ := os.ReadFile("ca.pem")
caCertPool := x509.NewCertPool()
caCertPool.AppendCertsFromPEM(caCert)

tlsConfig := &tls.Config{
    Certificates: []tls.Certificate{cert},
    RootCAs:      caCertPool,
}

transport := &http.Transport{TLSClientConfig: tlsConfig}
client := &http.Client{Transport: transport}
```

## TLS Server с mTLS

```go
// Загрузить CA для проверки клиентских сертификатов
caCert, _ := os.ReadFile("ca.pem")
caCertPool := x509.NewCertPool()
caCertPool.AppendCertsFromPEM(caCert)

tlsConfig := &tls.Config{
    ClientCAs:  caCertPool,
    ClientAuth: tls.RequireAndVerifyClientCert, // Требуем клиентский сертификат
}

server := &http.Server{
    Addr:      ":443",
    TLSConfig: tlsConfig,
}

log.Fatal(server.ListenAndServeTLS("server-cert.pem", "server-key.pem"))
```

## Let's Encrypt (автоматические сертификаты)

```go
import "golang.org/x/crypto/acme/autocert"

func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/", handler)

    certManager := autocert.Manager{
        Prompt:     autocert.AcceptTOS,
        HostPolicy: autocert.HostWhitelist("example.com", "www.example.com"),
        Cache:      autocert.DirCache("/var/www/.cache"), // Кэш сертификатов
    }

    server := &http.Server{
        Addr:    ":443",
        Handler: mux,
        TLSConfig: &tls.Config{
            GetCertificate: certManager.GetCertificate,
        },
    }

    // HTTP->HTTPS redirect
    go http.ListenAndServe(":80", certManager.HTTPHandler(nil))

    log.Fatal(server.ListenAndServeTLS("", ""))
}
```

## Проверка сертификата

```go
func verifyCert(host string) error {
    conn, err := tls.Dial("tcp", host+":443", &tls.Config{})
    if err != nil {
        return err
    }
    defer conn.Close()

    certs := conn.ConnectionState().PeerCertificates
    for _, cert := range certs {
        fmt.Printf("Issuer: %s\n", cert.Issuer)
        fmt.Printf("Subject: %s\n", cert.Subject)
        fmt.Printf("NotAfter: %s\n", cert.NotAfter)
    }

    return nil
}
```

## HTTP/2

HTTP/2 включен по умолчанию для HTTPS:

```go
server := &http.Server{
    Addr:    ":443",
    Handler: handler,
}

// HTTP/2 автоматически включается
log.Fatal(server.ListenAndServeTLS("cert.pem", "key.pem"))
```

### Отключить HTTP/2

```go
server := &http.Server{
    Addr:    ":443",
    Handler: handler,
    TLSNextProto: make(map[string]func(*http.Server, *tls.Conn, http.Handler)),
}
```

## Redirect HTTP -> HTTPS

```go
func redirectToHTTPS(w http.ResponseWriter, r *http.Request) {
    target := "https://" + r.Host + r.URL.Path
    if len(r.URL.RawQuery) > 0 {
        target += "?" + r.URL.RawQuery
    }
    http.Redirect(w, r, target, http.StatusMovedPermanently)
}

func main() {
    // HTTPS сервер
    go func() {
        mux := http.NewServeMux()
        mux.HandleFunc("/", handler)
        log.Fatal(http.ListenAndServeTLS(":443", "cert.pem", "key.pem", mux))
    }()

    // HTTP сервер (редирект на HTTPS)
    http.HandleFunc("/", redirectToHTTPS)
    log.Fatal(http.ListenAndServe(":80", nil))
}
```

## HSTS Header

```go
func securityHeadersMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // HSTS - заставляет браузер использовать HTTPS
        w.Header().Set("Strict-Transport-Security", "max-age=31536000; includeSubDomains")

        // Другие security headers
        w.Header().Set("X-Content-Type-Options", "nosniff")
        w.Header().Set("X-Frame-Options", "DENY")
        w.Header().Set("X-XSS-Protection", "1; mode=block")

        next.ServeHTTP(w, r)
    })
}
```

## Тестирование HTTPS

```go
func TestHTTPSEndpoint(t *testing.T) {
    // Создать test сервер с TLS
    ts := httptest.NewTLSServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintln(w, "Hello, HTTPS!")
    }))
    defer ts.Close()

    // ts.Client() автоматически настроен для работы с test сервером
    resp, err := ts.Client().Get(ts.URL)
    if err != nil {
        t.Fatal(err)
    }
    defer resp.Body.Close()

    body, _ := io.ReadAll(resp.Body)
    if string(body) != "Hello, HTTPS!\n" {
        t.Errorf("Unexpected body: %s", body)
    }
}
```

## Timeouts для TLS

```go
server := &http.Server{
    Addr:              ":443",
    Handler:           handler,
    ReadTimeout:       15 * time.Second,
    WriteTimeout:      15 * time.Second,
    IdleTimeout:       60 * time.Second,
    ReadHeaderTimeout: 5 * time.Second,
    TLSConfig: &tls.Config{
        MinVersion: tls.VersionTLS12,
    },
}
```

## Проверка версии TLS

```go
func handler(w http.ResponseWriter, r *http.Request) {
    if r.TLS != nil {
        version := r.TLS.Version
        switch version {
        case tls.VersionTLS10:
            fmt.Fprintf(w, "TLS 1.0")
        case tls.VersionTLS11:
            fmt.Fprintf(w, "TLS 1.1")
        case tls.VersionTLS12:
            fmt.Fprintf(w, "TLS 1.2")
        case tls.VersionTLS13:
            fmt.Fprintf(w, "TLS 1.3")
        }
    }
}
```

## Best Practices

### Рекомендуемая конфигурация TLS

```go
tlsConfig := &tls.Config{
    // Минимальная версия TLS 1.2
    MinVersion: tls.VersionTLS12,

    // Предпочитать server cipher suites
    PreferServerCipherSuites: true,

    // Современные cipher suites
    CipherSuites: []uint16{
        tls.TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,
        tls.TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,
        tls.TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,
        tls.TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,
    },

    // Предпочитаемые кривые
    CurvePreferences: []tls.CurveID{
        tls.X25519,
        tls.CurveP256,
    },
}
```

## Checklist

1. ✅ Используйте TLS 1.2 или выше
2. ✅ Используйте сильные cipher suites
3. ✅ Включите HSTS header
4. ✅ Redirect HTTP -> HTTPS
5. ✅ Валидные сертификаты (не self-signed в production)
6. ✅ Регулярно обновляйте сертификаты
7. ✅ Используйте Let's Encrypt для автоматизации
8. ✅ Тестируйте с httptest.NewTLSServer
9. ❌ Не используйте InsecureSkipVerify в production
10. ❌ Не используйте устаревшие TLS версии

## Связанные темы

- [[Go - Пакет net-http]]
- [[Go - Простой web-сервер]]
- [[Безопасность API]]
