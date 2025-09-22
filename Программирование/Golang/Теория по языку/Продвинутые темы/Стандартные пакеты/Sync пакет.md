---
tags:
  - Программирование
  - Теория_по_языку_Golang
  - Продвинутые_темы
  - sync
---
# **Пакет `sync` в Go**

## **1. Основное назначение**
Пакет `sync` предоставляет примитивы синхронизации для безопасной работы с общими ресурсами в конкурентных программах.  
**Подробнее:** [[Горутины]], [[Конкурентность в Go]].

---

## **2. Мьютексы**
### **`sync.Mutex` (взаимное исключение)**
Базовый примитив для защиты общих данных:
```go
var mu sync.Mutex
var counter int

func increment() {
    mu.Lock()         // Блокировка
    defer mu.Unlock() // Гарантированное освобождение
    counter++
}
```
**Особенности:**
- Только одна горутина может владеть мьютексом
- Повторная блокировка в одной горутине приводит к deadlock (`fatal error: all goroutines are asleep - deadlock!`)

### **`sync.RWMutex` (чтение-запись)**
Оптимизирован для сценариев с частым чтением:
```go
var rwMu sync.RWMutex
var cache map[string]string

func get(key string) string {
    rwMu.RLock() // Блокировка для чтения
    defer rwMu.RUnlock()
    return cache[key]
}

func set(key, value string) {
    rwMu.Lock() // Эксклюзивная блокировка
    defer rwMu.Unlock()
    cache[key] = value
}
```
**Правила:**
- Множество `RLock()` разрешены одновременно
- `Lock()` блокирует и читателей, и других писателей

---

## **3. `sync.WaitGroup`**
Счетчик для ожидания завершения группы горутин:
```go
var wg sync.WaitGroup

for i := 0; i < 5; i++ {
    wg.Add(1) // Увеличиваем счетчик
    go func(id int) {
        defer wg.Done() // Уменьшаем при завершении
        worker(id)
    }(i)
}

wg.Wait() // Ожидаем завершения всех
```
**Важно:**
- `Add()` должен вызываться до запуска горутины
- Нулевое значение `WaitGroup` готово к использованию

---

## **4. `sync.Once`**
Гарантирует однократное выполнение кода:
```go
var (
    once sync.Once
    config map[string]string
)

func loadConfig() {
    once.Do(func() { // Выполнится только один раз
        config = readConfigFile()
    })
}
```
**Применение:**
- Инициализация синглтонов
- Ленивая загрузка ресурсов
- Защита от повторной инициализации

---

## **5. Другие примитивы**
### **`sync.Pool`**
Пул временных объектов для уменьшения нагрузки на GC:
```go
var bufferPool = sync.Pool{
    New: func() interface{} {
        return bytes.NewBuffer(make([]byte, 0, 1024))
    },
}

// Получение
buf := bufferPool.Get().(*bytes.Buffer)
defer bufferPool.Put(buf) // Возврат
```

### **`sync.Map`**
Оптимизированная потокобезопасная мапа:
```go
var sm sync.Map
sm.Store("key", "value") // Безопасная запись
if val, ok := sm.Load("key"); ok {
    fmt.Println(val)
}
```
**Используйте когда:**
- Частое чтение при редкой записи
- Нужна атомарность операций

---

## **6. Опасные ситуации**
### **Deadlock**
```go
var mu sync.Mutex
mu.Lock()
mu.Lock() // Блокировка навсегда
```

### **Race condition**
```go
var data int
go func() { data++ }() // Гонка данных!
go func() { fmt.Println(data) }()
```
**Решение:** всегда использовать `go run -race` при тестировании.

### **Неправильное использование Once**
```go
var once sync.Once
var conn net.Conn

func getConn() net.Conn {
    once.Do(func() {
        conn = connect() // Если connect() вернет ошибку?
    })
    return conn // Всегда будет возвращать ошибочное соединение
}
```

---

## **7. Производительность**
### **Бенчмарк Mutex vs RWMutex**
```go
// Чтение с RWMutex: ~15 ns/op
// Чтение с Mutex: ~50 ns/op
// Запись: одинаково ~50 ns/op
```
**Вывод:** `RWMutex` выгоден только при >90% операций чтения.

### **Pool vs New**
```go
// Pool: 8 allocs/op
// New: 800 allocs/op (на 100 итераций)
```

---

**Дополнительно:**  
- [Официальная документация](https://pkg.go.dev/sync)  
- [Паттерны синхронизации](https://github.com/golang/go/wiki/MutexOrChannel)  

---

### **Когда что использовать?**
| Примитив    | Сценарий применения               |
| ----------- | --------------------------------- |
| `Mutex`     | Защита любых общих данных         |
| `RWMutex`   | Частое чтение, редкая запись      |
| `WaitGroup` | Ожидание группы горутин           |
| `Once`      | Однократная инициализация         |
| `Pool`      | Частое создание/удаление объектов |
| `sync.Map`  | Специфичные конкурентные сценарии |