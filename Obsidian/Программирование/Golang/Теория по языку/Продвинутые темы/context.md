---
tags:
  - Программирование
  - Теория_по_языку_Golang
  - Продвинутые_темы
Связанные темы:
---
# **Пакет context в Go**  
*Управление жизненным циклом горутин и распределенных операций*

---

## **1. Отмена операций через cancel()**  
### **Базовое использование**  
```go
ctx, cancel := context.WithCancel(context.Background())
defer cancel() // Освобождает ресурсы при завершении

go func() {
    select {
    case <-ctx.Done():
        fmt.Println("Операция отменена:", ctx.Err())
    case <-time.After(2 * time.Second):
        fmt.Println("Операция завершена")
    }
}()

// Принудительная отмена через 1 сек
time.Sleep(1 * time.Second)
cancel()
```

### **Особенности:**
- `cancel()` можно вызывать многократно
- Отмена распространяется на все дочерние контексты
- После отмены `ctx.Done()` всегда возвращает закрытый канал

---

## **2. Таймауты и дедлайны**  
### **WithTimeout**  
```go
ctx, cancel := context.WithTimeout(context.Background(), 100*time.Millisecond)
defer cancel()

select {
case <-time.After(200 * time.Millisecond):
    fmt.Println("Операция выполнена")
case <-ctx.Done():
    fmt.Println("Превышен таймаут:", ctx.Err()) // Сработает первым
}
```

### **WithDeadline**  
```go
deadline := time.Now().Add(50 * time.Millisecond)
ctx, cancel := context.WithDeadline(context.Background(), deadline)
defer cancel()

// Использование в HTTP-запросе
req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
client.Do(req)
```

**Разница между ними:**  
| WithTimeout               | WithDeadline              |
|---------------------------|---------------------------|
| Указывает длительность    | Указывает точное время    |
| `ctx.WithTimeout(100ms)`  | `ctx.WithDeadline(t)`     |

---

## **3. Передача метаданных**  
### **WithValue**  
```go
type ctxKey string

func middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        ctx := context.WithValue(r.Context(), ctxKey("user"), "admin")
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

func handler(w http.ResponseWriter, r *http.Request) {
    user := r.Context().Value(ctxKey("user")).(string)
    fmt.Fprintf(w, "Привет, %s!", user)
}
```

### **Правила использования значений:**
1. Ключи должны быть конкретными типами (не строками)
2. Значения должны быть потокобезопасными
3. Не использовать для передачи параметров функций

---

## **4. Композиция контекстов**  
### **Объединение нескольких контекстов**  
```go
// Контекст с таймаутом + метаданные
timeoutCtx, cancel := context.WithTimeout(context.Background(), time.Second)
defer cancel()

valueCtx := context.WithValue(timeoutCtx, ctxKey("requestID"), "12345")

// Приоритеты:
// 1. Отмена родительского контекста
// 2. Таймаут/дедлайн
// 3. Метаданные
```

### **Лучшие практики:**
- Всегда передавайте `context.Context` как первый аргумент
- Проверяйте `ctx.Done()` в долгих операциях
- Не храните контексты в структурах - передавайте явно

---

## **5. Реальные примеры**  

### **HTTP-сервер с таймаутами**  
```go
func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        ctx := r.Context()
        
        select {
        case <-time.After(5 * time.Second):
            w.Write([]byte("Ответ"))
        case <-ctx.Done():
            fmt.Println("Запрос отменен:", ctx.Err())
        }
    })
    
    srv := &http.Server{
        Handler:      mux,
        ReadTimeout:  3 * time.Second,
        WriteTimeout: 3 * time.Second,
    }
    srv.ListenAndServe()
}
```

### **Параллельные задачи с отменой**  
```go
func worker(ctx context.Context, id int, ch chan<- int) {
    select {
    case <-ctx.Done():
        return
    case <-time.After(time.Duration(id) * time.Second):
        ch <- id
    }
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel()
    
    ch := make(chan int)
    for i := 1; i <= 5; i++ {
        go worker(ctx, i, ch)
    }
    
    for i := 1; i <= 5; i++ {
        select {
        case <-ctx.Done():
            fmt.Println("Таймаут")
            return
        case result := <-ch:
            fmt.Println("Завершен:", result)
        }
    }
}
```

---

## **6. Ошибки и отладка**  

### **Типичные проблемы**  
1. **Утечки горутин**  
   - Не проверяете `ctx.Done()` в бесконечных циклах  
   ```go
   for {
       select {
       case <-ctx.Done():  // Обязательно!
           return
       default:
           // работа
       }
   }
   ```

2. **Неправильная вложенность**  
   ```go
   // Плохо:
   ctx := context.WithValue(ctx, key, value)
   ctx, cancel := context.WithTimeout(ctx, time.Second) // cancel не будет видеть value
   
   // Хорошо:
   ctx, cancel := context.WithTimeout(ctx, time.Second)
   ctx = context.WithValue(ctx, key, value)
   ```

3. **Использование фонового контекста**  
   ```go
   // Плохо:
   func SaveData(data Data) error {
       ctx := context.Background() // Нет контроля над жизненным циклом
       // ...
   }
   ```

---

**Дополнительно:**  
- [Официальная документация](https://golang.org/pkg/context/)  
- [Паттерны использования context](https://blog.golang.org/context)  
- [Context и gRPC](https://grpc.io/docs/guides/concepts/#context)  

Пакет `context` — это стандартный способ управления жизненным циклом операций в Go, особенно критичный для:  
- HTTP-серверов  
- Распределенных систем  
- Долгих вычислений  
- Параллельной обработки данных