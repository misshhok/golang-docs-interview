# Go - Пакет io и io/ioutil

Работа с потоками ввода-вывода.

## io.Reader

Базовый интерфейс для чтения:

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}
```

### Примеры Reader

```go
// Файл
file, _ := os.Open("file.txt")
defer file.Close()

// HTTP response
resp, _ := http.Get("https://example.com")
defer resp.Body.Close()

// Строка
reader := strings.NewReader("Hello, World!")

// Bytes
reader := bytes.NewReader([]byte{1, 2, 3})

// Stdin
reader := os.Stdin
```

### Чтение из Reader

```go
// Чтение блоков
buf := make([]byte, 1024)
n, err := reader.Read(buf)
if err != nil {
    if err == io.EOF {
        // Конец файла
    }
}

// Прочитать все
data, err := io.ReadAll(reader)

// Копировать в Writer
n, err := io.Copy(dst, src)

// Копировать N байт
n, err := io.CopyN(dst, src, 1024)
```

## io.Writer

Базовый интерфейс для записи:

```go
type Writer interface {
    Write(p []byte) (n int, err error)
}
```

### Примеры Writer

```go
// Файл
file, _ := os.Create("output.txt")
defer file.Close()

// HTTP response
func handler(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("Hello"))
}

// Buffer
var buf bytes.Buffer

// Stdout
writer := os.Stdout
```

### Запись в Writer

```go
// Запись байт
n, err := writer.Write([]byte("Hello"))

// Запись строки
io.WriteString(writer, "Hello, World!")

// Копирование из Reader
io.Copy(writer, reader)
```

## io.Closer

```go
type Closer interface {
    Close() error
}

// Всегда закрывайте!
file, _ := os.Open("file.txt")
defer file.Close()
```

## Композитные интерфейсы

```go
// ReadWriter
type ReadWriter interface {
    Reader
    Writer
}

// ReadCloser
type ReadCloser interface {
    Reader
    Closer
}

// WriteCloser
type WriteCloser interface {
    Writer
    Closer
}

// ReadWriteCloser
type ReadWriteCloser interface {
    Reader
    Writer
    Closer
}
```

## io util функции

### ReadAll

```go
// Прочитать все содержимое
data, err := io.ReadAll(reader)
fmt.Println(string(data))
```

### ReadFile (os)

```go
// Прочитать весь файл
data, err := os.ReadFile("file.txt")
fmt.Println(string(data))
```

### WriteFile (os)

```go
// Записать в файл
data := []byte("Hello, World!")
err := os.WriteFile("file.txt", data, 0644)
```

### Copy

```go
// Копировать из src в dst
src, _ := os.Open("source.txt")
defer src.Close()

dst, _ := os.Create("dest.txt")
defer dst.Close()

n, err := io.Copy(dst, src)
fmt.Printf("Copied %d bytes\n", n)
```

### CopyN

```go
// Копировать N байт
n, err := io.CopyN(dst, src, 1024)
```

### CopyBuffer

```go
// Копировать с кастомным буфером
buf := make([]byte, 32*1024) // 32 KB buffer
n, err := io.CopyBuffer(dst, src, buf)
```

## Буферизация

### bufio.Reader

```go
import "bufio"

file, _ := os.Open("file.txt")
defer file.Close()

reader := bufio.NewReader(file)

// Читать построчно
for {
    line, err := reader.ReadString('\n')
    if err == io.EOF {
        break
    }
    fmt.Print(line)
}

// Или
scanner := bufio.NewScanner(file)
for scanner.Scan() {
    fmt.Println(scanner.Text())
}
```

### bufio.Writer

```go
file, _ := os.Create("output.txt")
defer file.Close()

writer := bufio.NewWriter(file)
defer writer.Flush() // Важно!

writer.WriteString("Hello\n")
writer.WriteString("World\n")
```

## Pipes

```go
// Создать pipe
pr, pw := io.Pipe()

// Запись в горутине
go func() {
    defer pw.Close()
    pw.Write([]byte("Hello from pipe!"))
}()

// Чтение
data, _ := io.ReadAll(pr)
fmt.Println(string(data))
```

## MultiReader

```go
// Объединить несколько Reader
r1 := strings.NewReader("Hello ")
r2 := strings.NewReader("World!")

mr := io.MultiReader(r1, r2)
data, _ := io.ReadAll(mr)
fmt.Println(string(data)) // "Hello World!"
```

## MultiWriter

```go
// Писать в несколько Writer одновременно
file, _ := os.Create("output.txt")
defer file.Close()

mw := io.MultiWriter(file, os.Stdout)
mw.Write([]byte("Hello\n")) // Пишет и в файл, и в stdout
```

## TeeReader

```go
// Читать и писать одновременно
src := strings.NewReader("Hello, World!")
var buf bytes.Buffer

tee := io.TeeReader(src, &buf)

// Читаем из tee
data, _ := io.ReadAll(tee)

// buf теперь содержит копию
fmt.Println(buf.String()) // "Hello, World!"
```

## LimitReader

```go
// Ограничить чтение N байтами
src := strings.NewReader("Hello, World!")
limited := io.LimitReader(src, 5)

data, _ := io.ReadAll(limited)
fmt.Println(string(data)) // "Hello"
```

## Примеры использования

### Чтение файла построчно

```go
file, err := os.Open("file.txt")
if err != nil {
    log.Fatal(err)
}
defer file.Close()

scanner := bufio.NewScanner(file)
for scanner.Scan() {
    line := scanner.Text()
    fmt.Println(line)
}

if err := scanner.Err(); err != nil {
    log.Fatal(err)
}
```

### Копирование файла

```go
func copyFile(src, dst string) error {
    source, err := os.Open(src)
    if err != nil {
        return err
    }
    defer source.Close()

    destination, err := os.Create(dst)
    if err != nil {
        return err
    }
    defer destination.Close()

    _, err = io.Copy(destination, source)
    return err
}
```

### Запись в файл с буферизацией

```go
func writeLines(filename string, lines []string) error {
    file, err := os.Create(filename)
    if err != nil {
        return err
    }
    defer file.Close()

    writer := bufio.NewWriter(file)
    defer writer.Flush()

    for _, line := range lines {
        fmt.Fprintln(writer, line)
    }

    return nil
}
```

### Чтение HTTP response

```go
resp, err := http.Get("https://example.com")
if err != nil {
    log.Fatal(err)
}
defer resp.Body.Close()

// Прочитать все
body, err := io.ReadAll(resp.Body)
fmt.Println(string(body))

// Или построчно
scanner := bufio.NewScanner(resp.Body)
for scanner.Scan() {
    fmt.Println(scanner.Text())
}
```

### Progress reader

```go
type ProgressReader struct {
    reader io.Reader
    total  int64
    read   int64
}

func (pr *ProgressReader) Read(p []byte) (int, error) {
    n, err := pr.reader.Read(p)
    pr.read += int64(n)

    progress := float64(pr.read) / float64(pr.total) * 100
    fmt.Printf("\rProgress: %.2f%%", progress)

    return n, err
}
```

## Временные файлы/директории

```go
// Временный файл
tmpfile, err := os.CreateTemp("", "example")
if err != nil {
    log.Fatal(err)
}
defer os.Remove(tmpfile.Name())
defer tmpfile.Close()

tmpfile.Write([]byte("temporary data"))

// Временная директория
tmpdir, err := os.MkdirTemp("", "example")
if err != nil {
    log.Fatal(err)
}
defer os.RemoveAll(tmpdir)
```

## Best Practices

1. ✅ Всегда defer Close() для файлов и connections
2. ✅ Проверяйте ошибки (особенно Close)
3. ✅ Используйте bufio для построчного чтения
4. ✅ Используйте io.Copy для копирования
5. ✅ Flush() буферизованные Writers
6. ✅ Обрабатывайте io.EOF правильно
7. ❌ Не читайте огромные файлы полностью в память
8. ❌ Не забывайте Flush для bufio.Writer

## Связанные темы

- [[Go - Пакет net-http]]
- [[Go - Обработка ошибок]]
- [[Go - Defer, Panic, Recover]]
