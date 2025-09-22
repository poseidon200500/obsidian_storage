---
tags:
  - Программирование
  - Теория_по_языку_Golang
  - Продвинутые_темы
Связанные темы:
---
# **Примитивы синхронизации в Go (пакет sync)**  
*Базовые механизмы для безопасного доступа к общим ресурсам*

---

## **1. Mutex и RWMutex**  
### **sync.Mutex**  
**Эксклюзивная блокировка:**  
```go
var mu sync.Mutex
var counter int

func increment() {
    mu.Lock()         // Блокировка
    defer mu.Unlock() // Гарантированное освобождение
    counter++
}
```
- Гарантирует, что только **одна горутина** может владеть блокировкой
- `Lock()` блокирует, если мьютекс уже занят
- Всегда используйте `defer Unlock()` для предотвращения deadlock

### **sync.RWMutex**  
**Оптимизированная блокировка для чтения/записи:**  
```go
var rwMu sync.RWMutex
var cache map[string]string

func get(key string) string {
    rwMu.RLock()       // Блокировка для чтения
    defer rwMu.RUnlock()
    return cache[key]  // Множество горутин может читать одновременно
}

func set(key, value string) {
    rwMu.Lock()        // Эксклюзивная блокировка
    defer rwMu.Unlock()
    cache[key] = value // Только одна горутина пишет
}
```
- **RLock()**: множественные читатели (shared lock)
- **Lock()**: единственный писатель (exclusive lock)

---

## **2. WaitGroup**  
**Ожидание завершения группы горутин:**  
```go
var wg sync.WaitGroup

func worker(id int) {
    defer wg.Done()  // Уменьшаем счетчик при завершении
    fmt.Printf("Worker %d started\n", id)
    time.Sleep(time.Second)
}

func main() {
    for i := 1; i <= 5; i++ {
        wg.Add(1)    // Увеличиваем счетчик перед запуском
        go worker(i)
    }
    wg.Wait()       // Ожидаем завершения всех
    fmt.Println("All workers done")
}
```
- **Add(n)**: увеличивает счетчик на `n` (вызывается в **основном потоке**)
- **Done()**: уменьшает счетчик на 1 (аналог `Add(-1)`)
- **Wait()**: блокируется, пока счетчик ≠ 0

---

## **3. Once**  
**Гарантированное однократное выполнение:**  
```go
var (
    once sync.Once
    config map[string]string
)

func loadConfig() {
    once.Do(func() {  // Выполнится только один раз
        config = readConfigFile()  // Инициализация
    })
}
```
- Полезен для:
  - Ленивой инициализации
  - Создания синглтонов
  - Однократной настройки

---

## **4. Другие примитивы**  
### **sync.Pool**  
**Пул временных объектов:**  
```go
var bufPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

func getBuffer() *bytes.Buffer {
    return bufPool.Get().(*bytes.Buffer)
}

func putBuffer(buf *bytes.Buffer) {
    buf.Reset()
    bufPool.Put(buf)
}
```
- Уменьшает нагрузку на GC
- Не гарантирует сохранность объектов

### **sync.Map**  
**Потокобезопасная мапа:**  
```go
var sm sync.Map

sm.Store("key", "value")  // Безопасная запись
val, ok := sm.Load("key") // Безопасное чтение
```
- Оптимизирована для случаев:
  - Много чтений / редкие записи
  - Конкурентный доступ без блокировок

---

## **5. Антипаттерны**  
### **a) Забытый Unlock**  
```go
mu.Lock()
result := process()  // Если panic → deadlock!
mu.Unlock()
```
**Решение:** Всегда использовать `defer`

### **b) Глобальные мьютексы**  
```go
var dbMu sync.Mutex  // Слишком широкий scope

func save(data string) {
    dbMu.Lock()      // Блокирует ВСЕ операции
    defer dbMu.Unlock()
    // ...
}
```
**Решение:** Использовать мьютексы на уровне конкретных структур

### **c) Копирование примитивов**  
```go
var mu sync.Mutex
mu2 := mu  // Копирование → НОВЫЙ мьютекс!
```
**Решение:** Всегда передавать по указателю

---

## **6. Производительность**  
| Примитив       | Время операции (ns) | Использование CPU |
|----------------|---------------------|-------------------|
| Mutex          | 18-25               | Высокое           |
| RWMutex (Read) | 8-12                | Среднее           |
| Atomic         | 2-5                 | Низкое           |

**Совет:** Для простых счетчиков используйте `atomic`:
```go
var counter int64
atomic.AddInt64(&counter, 1)  // Без блокировок
```

---

**Дополнительно:**  
- [Официальная документация sync](https://pkg.go.dev/sync)  
- [Исходный код mutex.go](https://github.com/golang/go/blob/master/src/sync/mutex.go)