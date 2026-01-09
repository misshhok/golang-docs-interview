# HTTPS и TLS

HTTPS (HTTP Secure) — это расширение HTTP протокола с шифрованием через TLS/SSL, обеспечивающее конфиденциальность и целостность данных при передаче между клиентом и сервером.

## Основные концепции

TLS (Transport Layer Security) — криптографический протокол, который обеспечивает безопасную передачу данных в интернете. HTTPS = HTTP + TLS.

### Зачем нужен HTTPS

1. **Конфиденциальность** — шифрование данных (пароли, токены, личная информация)
2. **Целостность** — защита от подмены данных
3. **Аутентификация** — подтверждение подлинности сервера
4. **SEO** — Google приоритизирует HTTPS сайты
5. **Trust** — браузеры показывают замочек, пользователи доверяют больше

## TLS Handshake (Рукопожатие)

Процесс установления защищённого соединения между клиентом и сервером.

### Шаги TLS 1.2 Handshake

```
1. Client Hello
   Client -> Server
   - Версия TLS
   - Список поддерживаемых cipher suites
   - Random bytes

2. Server Hello
   Server -> Client
   - Выбранная версия TLS
   - Выбранный cipher suite
   - Random bytes
   - Сертификат сервера

3. Key Exchange
   Client -> Server
   - Клиент проверяет сертификат
   - Генерирует pre-master secret
   - Шифрует его публичным ключом сервера
   - Отправляет на сервер

4. Finished
   Client & Server
   - Оба вычисляют master secret
   - Оба отправляют "Finished" сообщения (зашифрованные)
   - Handshake завершён

5. Application Data
   - Защищённая передача данных
```

### TLS 1.3 (улучшенный)

TLS 1.3 быстрее — всего 1 round-trip вместо 2.

```
Client Hello + Key Share -> Server
Server Hello + Key Share + Finished <- Server
[Application Data] -> Server (сразу после!)
```

## Сертификаты

### Структура сертификата

```
X.509 Certificate:
- Subject (кому выдан): example.com
- Issuer (кто выдал): Let's Encrypt Authority
- Validity Period: 2024-01-01 до 2024-12-31
- Public Key: RSA 2048 бит
- Signature: подпись CA
```

### Типы сертификатов

1. **Domain Validation (DV)** — проверка владения доменом (бесплатно, Let's Encrypt)
2. **Organization Validation (OV)** — проверка организации
3. **Extended Validation (EV)** — расширенная проверка (зелёная строка в браузере)

## Реализация HTTPS в Go

### Простой HTTPS сервер

```go
package main

import (
    "fmt"
    "log"
    "net/http"
)

func main() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Secure connection!")
    })

    // HTTPS сервер с сертификатами
    log.Println("Starting HTTPS server on :443")
    err := http.ListenAndServeTLS(
        ":443",
        "server.crt", // Путь к сертификату
        "server.key", // Путь к приватному ключу
        nil,
    )

    if err != nil {
        log.Fatal(err)
    }
}
```

### Генерация self-signed сертификата для разработки

```bash
# Генерация приватного ключа и сертификата
openssl req -x509 -newkey rsa:4096 -keyout server.key -out server.crt \
  -days 365 -nodes -subj "/CN=localhost"

# Или использовать Go:
go run $(go env GOROOT)/src/crypto/tls/generate_cert.go --host localhost
```

```go
package main

import (
    "crypto/rand"
    "crypto/rsa"
    "crypto/x509"
    "crypto/x509/pkix"
    "encoding/pem"
    "math/big"
    "os"
    "time"
)

// Генерация self-signed сертификата в Go
func generateSelfSignedCert() error {
    // Генерируем приватный ключ
    privateKey, err := rsa.GenerateKey(rand.Reader, 2048)
    if err != nil {
        return err
    }

    // Шаблон сертификата
    template := x509.Certificate{
        SerialNumber: big.NewInt(1),
        Subject: pkix.Name{
            Organization: []string{"My Company"},
            CommonName:   "localhost",
        },
        NotBefore:             time.Now(),
        NotAfter:              time.Now().Add(365 * 24 * time.Hour),
        KeyUsage:              x509.KeyUsageKeyEncipherment | x509.KeyUsageDigitalSignature,
        ExtKeyUsage:           []x509.ExtKeyUsage{x509.ExtKeyUsageServerAuth},
        BasicConstraintsValid: true,
        DNSNames:              []string{"localhost", "127.0.0.1"},
    }

    // Создаём сертификат
    certDER, err := x509.CreateCertificate(rand.Reader, &template, &template, &privateKey.PublicKey, privateKey)
    if err != nil {
        return err
    }

    // Сохраняем сертификат
    certOut, err := os.Create("server.crt")
    if err != nil {
        return err
    }
    defer certOut.Close()
    pem.Encode(certOut, &pem.Block{Type: "CERTIFICATE", Bytes: certDER})

    // Сохраняем приватный ключ
    keyOut, err := os.Create("server.key")
    if err != nil {
        return err
    }
    defer keyOut.Close()
    pem.Encode(keyOut, &pem.Block{
        Type:  "RSA PRIVATE KEY",
        Bytes: x509.MarshalPKCS1PrivateKey(privateKey),
    })

    return nil
}
```

### Настройка TLS Config

```go
package main

import (
    "crypto/tls"
    "log"
    "net/http"
)

func main() {
    // Кастомная TLS конфигурация
    tlsConfig := &tls.Config{
        MinVersion: tls.VersionTLS12, // Минимум TLS 1.2
        MaxVersion: tls.VersionTLS13, // Максимум TLS 1.3
        CipherSuites: []uint16{
            // Рекомендуемые cipher suites
            tls.TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,
            tls.TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,
            tls.TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,
            tls.TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,
        },
        PreferServerCipherSuites: true,
        // Не использовать сжатие (против CRIME attack)
        // SessionTicketsDisabled: false, // Можно отключить для большей безопасности
    }

    server := &http.Server{
        Addr:      ":443",
        TLSConfig: tlsConfig,
        Handler:   http.DefaultServeMux,
    }

    log.Println("Starting secure server")
    log.Fatal(server.ListenAndServeTLS("server.crt", "server.key"))
}
```

### Автоматическое получение сертификатов Let's Encrypt

```go
package main

import (
    "crypto/tls"
    "log"
    "net/http"

    "golang.org/x/crypto/acme/autocert"
)

func main() {
    // Менеджер автоматических сертификатов
    certManager := autocert.Manager{
        Prompt:     autocert.AcceptTOS,
        HostPolicy: autocert.HostWhitelist("example.com", "www.example.com"),
        Cache:      autocert.DirCache("certs"), // Кеш сертификатов
    }

    server := &http.Server{
        Addr: ":443",
        TLSConfig: &tls.Config{
            GetCertificate: certManager.GetCertificate,
            MinVersion:     tls.VersionTLS12,
        },
        Handler: http.DefaultServeMux,
    }

    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        w.Write([]byte("Hello, HTTPS!"))
    })

    // HTTP сервер для ACME challenge (Let's Encrypt проверка)
    go http.ListenAndServe(":80", certManager.HTTPHandler(nil))

    log.Println("Starting server with Let's Encrypt")
    log.Fatal(server.ListenAndServeTLS("", ""))
}
```

## HTTPS Клиент в Go

### Базовый HTTPS запрос

```go
package main

import (
    "fmt"
    "io"
    "net/http"
)

func main() {
    // Стандартный HTTPS клиент
    resp, err := http.Get("https://example.com")
    if err != nil {
        log.Fatal(err)
    }
    defer resp.Body.Close()

    body, _ := io.ReadAll(resp.Body)
    fmt.Println(string(body))
}
```

### Клиент с кастомным TLS Config

```go
package main

import (
    "crypto/tls"
    "crypto/x509"
    "io/ioutil"
    "net/http"
)

func createSecureClient() *http.Client {
    // Загрузка корневых сертификатов
    caCert, err := ioutil.ReadFile("ca.crt")
    if err != nil {
        log.Fatal(err)
    }

    caCertPool := x509.NewCertPool()
    caCertPool.AppendCertsFromPEM(caCert)

    tlsConfig := &tls.Config{
        RootCAs:    caCertPool,
        MinVersion: tls.VersionTLS12,
    }

    transport := &http.Transport{
        TLSClientConfig: tlsConfig,
    }

    return &http.Client{
        Transport: transport,
    }
}

func main() {
    client := createSecureClient()
    resp, err := client.Get("https://secure-api.example.com")
    if err != nil {
        log.Fatal(err)
    }
    defer resp.Body.Close()
}
```

### Пропуск проверки сертификата (только для development!)

```go
package main

import (
    "crypto/tls"
    "net/http"
)

func insecureClient() *http.Client {
    // ❌ ТОЛЬКО ДЛЯ РАЗРАБОТКИ!
    transport := &http.Transport{
        TLSClientConfig: &tls.Config{
            InsecureSkipVerify: true, // Не проверять сертификат
        },
    }

    return &http.Client{
        Transport: transport,
    }
}
```

## mTLS (Mutual TLS)

Двусторонняя аутентификация — не только сервер предоставляет сертификат, но и клиент.

### Сервер с mTLS

```go
package main

import (
    "crypto/tls"
    "crypto/x509"
    "io/ioutil"
    "log"
    "net/http"
)

func main() {
    // Загружаем CA для проверки клиентских сертификатов
    caCert, err := ioutil.ReadFile("ca.crt")
    if err != nil {
        log.Fatal(err)
    }

    caCertPool := x509.NewCertPool()
    caCertPool.AppendCertsFromPEM(caCert)

    tlsConfig := &tls.Config{
        ClientAuth: tls.RequireAndVerifyClientCert, // Требуем клиентский сертификат
        ClientCAs:  caCertPool,
        MinVersion: tls.VersionTLS12,
    }

    server := &http.Server{
        Addr:      ":443",
        TLSConfig: tlsConfig,
    }

    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        // Получаем информацию о клиентском сертификате
        if len(r.TLS.PeerCertificates) > 0 {
            cert := r.TLS.PeerCertificates[0]
            w.Write([]byte("Authenticated as: " + cert.Subject.CommonName))
        }
    })

    log.Fatal(server.ListenAndServeTLS("server.crt", "server.key"))
}
```

### Клиент с mTLS

```go
package main

import (
    "crypto/tls"
    "net/http"
)

func createMTLSClient() *http.Client {
    // Загружаем клиентский сертификат
    cert, err := tls.LoadX509KeyPair("client.crt", "client.key")
    if err != nil {
        log.Fatal(err)
    }

    tlsConfig := &tls.Config{
        Certificates: []tls.Certificate{cert},
        MinVersion:   tls.VersionTLS12,
    }

    transport := &http.Transport{
        TLSClientConfig: tlsConfig,
    }

    return &http.Client{
        Transport: transport,
    }
}

func main() {
    client := createMTLSClient()
    resp, err := client.Get("https://mtls-server.example.com")
    if err != nil {
        log.Fatal(err)
    }
    defer resp.Body.Close()
}
```

## Редирект с HTTP на HTTPS

```go
package main

import (
    "log"
    "net/http"
)

func redirectToHTTPS(w http.ResponseWriter, r *http.Request) {
    httpsURL := "https://" + r.Host + r.RequestURI
    http.Redirect(w, r, httpsURL, http.StatusMovedPermanently)
}

func main() {
    // HTTP сервер для редиректа
    go func() {
        log.Println("HTTP redirect server on :80")
        http.ListenAndServe(":80", http.HandlerFunc(redirectToHTTPS))
    }()

    // HTTPS сервер
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        w.Write([]byte("Secure!"))
    })

    log.Println("HTTPS server on :443")
    log.Fatal(http.ListenAndServeTLS(":443", "server.crt", "server.key", nil))
}
```

## HSTS (HTTP Strict Transport Security)

Заголовок, который указывает браузеру всегда использовать HTTPS.

```go
func securityHeadersMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // HSTS: заставляет браузер использовать только HTTPS
        w.Header().Set("Strict-Transport-Security", "max-age=31536000; includeSubDomains; preload")

        // Запрет отображения в iframe (против clickjacking)
        w.Header().Set("X-Frame-Options", "DENY")

        // XSS Protection
        w.Header().Set("X-Content-Type-Options", "nosniff")
        w.Header().Set("X-XSS-Protection", "1; mode=block")

        next.ServeHTTP(w, r)
    })
}
```

## Проверка качества TLS

### Онлайн инструменты

1. **SSL Labs** — https://www.ssllabs.com/ssltest/
2. **Mozilla Observatory** — https://observatory.mozilla.org/

### Команда openssl

```bash
# Проверка сертификата сервера
openssl s_client -connect example.com:443 -showcerts

# Проверка версии TLS
openssl s_client -connect example.com:443 -tls1_2

# Проверка cipher suites
nmap --script ssl-enum-ciphers -p 443 example.com
```

## Типичные ошибки

### ❌ Ошибка 1: Использование устаревших версий TLS

```go
// ❌ TLS 1.0 и 1.1 устарели
tlsConfig := &tls.Config{
    MinVersion: tls.VersionTLS10,
}
```

```go
// ✅ Минимум TLS 1.2
tlsConfig := &tls.Config{
    MinVersion: tls.VersionTLS12,
}
```

### ❌ Ошибка 2: Слабые cipher suites

```go
// ✅ Использовать только сильные cipher suites
CipherSuites: []uint16{
    tls.TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,
    tls.TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,
}
```

### ❌ Ошибка 3: InsecureSkipVerify в продакшене

```go
// ❌ НИКОГДА в продакшене!
TLSClientConfig: &tls.Config{
    InsecureSkipVerify: true,
}
```

### ❌ Ошибка 4: Истёкший сертификат

```bash
# Проверка срока действия сертификата
openssl x509 -in server.crt -noout -dates

# Автоматизация с Let's Encrypt
# Используйте autocert или certbot для автообновления
```

## Best Practices

1. ✅ **Используйте TLS 1.2 или 1.3** (отключите TLS 1.0/1.1)
2. ✅ **Используйте сильные cipher suites** (ECDHE, AES-GCM)
3. ✅ **Включите HSTS** для принудительного HTTPS
4. ✅ **Используйте Let's Encrypt** для бесплатных сертификатов
5. ✅ **Настройте автообновление** сертификатов
6. ✅ **Редиректьте HTTP -> HTTPS** автоматически
7. ✅ **Используйте mTLS** для inter-service коммуникации
8. ✅ **Тестируйте конфигурацию** на SSL Labs
9. ✅ **Храните приватные ключи безопасно** (не в git!)
10. ✅ **Мониторьте срок действия** сертификатов

## Вопросы с собеседований

**Вопрос 1:** В чём разница между SSL и TLS?

**Ответ:** SSL (Secure Sockets Layer) — устаревший протокол (SSL 3.0 deprecated с 2015). TLS (Transport Layer Security) — современная версия SSL. TLS 1.0 = SSL 3.1. Сейчас используют TLS 1.2 и TLS 1.3. Термины часто используют взаимозаменяемо, но правильно говорить TLS.

**Вопрос 2:** Как работает TLS handshake?

**Ответ:**
1. Client Hello (клиент отправляет версии TLS, cipher suites)
2. Server Hello (сервер выбирает параметры, отправляет сертификат)
3. Key Exchange (клиент проверяет сертификат, обменивается ключами)
4. Finished (обе стороны подтверждают установку соединения)
5. Application Data (шифрованная передача данных)

**Вопрос 3:** Что такое Perfect Forward Secrecy (PFS)?

**Ответ:** PFS — свойство, при котором компрометация долгосрочного ключа не позволяет расшифровать прошлые сессии. Достигается использованием эфемерных (временных) ключей для каждой сессии (ECDHE, DHE). Если кто-то украдёт приватный ключ сервера, он не сможет расшифровать старый трафик.

**Вопрос 4:** Что такое Certificate Pinning?

**Ответ:** Certificate Pinning — техника, при которой клиент "прикрепляет" (pin) конкретный сертификат или публичный ключ сервера. Клиент будет принимать только этот сертификат, игнорируя даже валидные сертификаты от доверенных CA. Защита от MITM атак через скомпрометированные CA.

## Связанные темы

- [[HTTP протокол]]
- [[OAuth 2.0]]
- [[JWT (JSON Web Tokens)]]
- [[Basic Authentication]]
- [[Session-based аутентификация]]
- [[Безопасность API]]
- [[Go - TLS и HTTPS]]
- [[Go - Пакет net-http]]
