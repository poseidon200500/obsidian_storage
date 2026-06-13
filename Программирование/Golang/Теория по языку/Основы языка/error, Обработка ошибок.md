---
tags:
  - Программирование
  - Основы_языка_Golang
---
# **Ошибки (`error`) и их обработка в Go**

## **1. Основы ошибок в Go**

**Тип `error`** — это встроенный интерфейс:
```go
type error interface {
    Error() string
}
```
Любая структура, реализующая метод `Error() string`, является ошибкой.

### **Создание ошибок**
1. **Простые ошибки** (пакет `errors`):
```go
err := errors.New("файл не найден")
```

2. **Форматированные ошибки** (пакет `fmt`):
```go
err := fmt.Errorf("ошибка в строке %d", line)
```

3. **Кастомные ошибки** (реализация интерфейса):
```go
type FileNotFoundError struct {
    Filename string
}

func (e *FileNotFoundError) Error() string {
    return fmt.Sprintf("файл %s не найден", e.Filename)
}

// Использование
err := &FileNotFoundError{"config.json"}
```

---

## **2. Обработка ошибок**

### **Проверка ошибок (явная обработка)**
```go
file, err := os.Open("data.txt")
if err != nil {
    log.Printf("Ошибка открытия файла: %v", err)
    return err
}
defer file.Close()
```

### **Проверка типа ошибки**
```go
if _, err := os.Open("nonexistent.txt"); err != nil {
    if errors.Is(err, os.ErrNotExist) {
        fmt.Println("Файл не существует!")
    } else {
        fmt.Println("Другая ошибка:", err)
    }
}
```

### **Обертки ошибок (`%w`)**
```go
if err := readConfig(); err != nil {
    return fmt.Errorf("не удалось прочитать конфиг: %w", err)
}
```

### **Извлечение оригинальной ошибки (`errors.As`, `errors.Is`)**
```go
var pathError *os.PathError
if errors.As(err, &pathError) {
    fmt.Println("Ошибка в пути:", pathError.Path)
}
```

---

## **3. Практические паттерны**

### **1. Централизованная обработка**
```go
func handleHTTPRequest(w http.ResponseWriter, r *http.Request) {
    defer func() {
        if err := recover(); err != nil {
            log.Printf("PANIC: %v", err)
            http.Error(w, "Internal Server Error", 500)
        }
    }()

    data, err := fetchData(r.URL.Query())
    if err != nil {
        log.Printf("Ошибка запроса: %v", err)
        http.Error(w, err.Error(), 400)
        return
    }

    w.Write(data)
}
```

### **2. Кастомные коды ошибок**
```go
var (
    ErrUserNotFound = errors.New("пользователь не найден")
    ErrInvalidInput = errors.New("неверные входные данные")
)

func getUser(id int) (*User, error) {
    if id <= 0 {
        return nil, ErrInvalidInput
    }
    // ...
}
```

### **3. Логирование с контекстом**
```go
func processJob(data []byte) error {
    if err := validate(data); err != nil {
        return fmt.Errorf("валидация данных: %w", err)
    }
    // ...
}
```

---

## **4. Связь с другими концепциями**

### **1. `defer` + ошибки**
```go
func readFile(filename string) (err error) {
    file, err := os.Open(filename)
    if err != nil {
        return fmt.Errorf("ошибка открытия: %w", err)
    }
    defer func() {
        if closeErr := file.Close(); closeErr != nil {
            err = fmt.Errorf("ошибка закрытия: %v (исходная ошибка: %w)", closeErr, err)
        }
    }()

    // Чтение файла...
    return nil
}
```

### **2. Горутины и ошибки**
```go
func worker(results chan<- Result, errors chan<- error) {
    defer func() {
        if r := recover(); r != nil {
            errors <- fmt.Errorf("паника в горутине: %v", r)
        }
    }()

    // Работа...
    if err := doTask(); err != nil {
        errors <- err
        return
    }

    results <- result
}
```

### **3. `context` и отмена операций**
```go
func fetchData(ctx context.Context, url string) ([]byte, error) {
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return nil, fmt.Errorf("ошибка создания запроса: %w", err)
    }

    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return nil, fmt.Errorf("ошибка HTTP-запроса: %w", err)
    }
    defer resp.Body.Close()

    // ...
}
```

---

## **5. Лучшие практики**

✅ **Явная обработка** — всегда проверяйте `err != nil`  
✅ **Добавляйте контекст** — `fmt.Errorf("шаг 1: %w", err)`  
✅ **Логируйте ошибки** — но не дублируйте в коде  
✅ **Используйте `errors.Is`/`errors.As`** вместо прямых сравнений  
❌ **Не игнорируйте ошибки** — `_, _ = fmt.Println()` это плохо  
❌ **Не паникуйте** на ожидаемых ошибках  

```go
// Плохо
_ = os.Remove("temp.txt")

// Хорошо
if err := os.Remove("temp.txt"); err != nil && !errors.Is(err, os.ErrNotExist) {
    log.Printf("Ошибка удаления файла: %v", err)
}
```

---

## **Заключение**
Ошибки в Go — это **явная часть API**, а не исключения.  
**Правила хорошего тона:**  
1. **Документируйте** возможные ошибки в godoc  
2. **Тестируйте** обработку ошибок  
3. **Используйте обертки** для добавления контекста  

Подробнее: [[Тестирование в Go]], [[Работа с файлами в Go]].