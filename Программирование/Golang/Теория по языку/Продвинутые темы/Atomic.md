---
tags:
  - Программирование
  - Теория_по_языку_Golang
  - Продвинутые_темы
  - Atomic
Связанные темы:
---
# **Атомарные операции в Go (пакет sync/atomic)**  
*Безблокировочные операции для работы с разделяемыми данными*

---

## **1. Базовые атомарные операции**  
### **Атомарные счетчики**  
```go
var counter int32

func increment() {
    atomic.AddInt32(&counter, 1)  // Атомарное увеличение
}

func getValue() int32 {
    return atomic.LoadInt32(&counter)  // Безопасное чтение
}
```
**Поддерживаемые типы:**  
- `int32`, `int64`, `uint32`, `uint64`, `uintptr`  
- `unsafe.Pointer` (для произвольных указателей)  

### **Compare-And-Swap (CAS)**  
```go
var config atomic.Value  // Атомарное значение

func updateConfig(newCfg Config) {
    for {
        old := config.Load().(Config)
        if atomic.CompareAndSwapPointer(
            (*unsafe.Pointer)(unsafe.Pointer(&config)),
            unsafe.Pointer(&old),
            unsafe.Pointer(&newCfg)) {
            break  // Успешное обновление
        }
    }
}
```
**Принцип работы CAS:**  
1. Сравнивает текущее значение с ожидаемым  
2. Если совпадает — устанавливает новое значение  
3. Возвращает `true` при успехе  

---

## **2. Memory Ordering гарантии**  
### **Модель памяти Go**  
- **Атомарные операции** обеспечивают последовательную согласованность  
- **Нормальные** чтения/записи могут переупорядочиваться  

**Пример проблемы:**  
```go
var (
    data int
    ready bool  // Без атомарности!
)

func writer() {
    data = 42
    ready = true  // Может быть переупорядочено!
}

func reader() {
    if ready {  // Может увидеть true ДО записи data=42!
        fmt.Println(data)
    }
}
```
**Решение:**  
```go
var ready atomic.Bool  // Атомарный флаг
```

---

## **3. Паттерны использования**  

### **a) Lock-free структуры данных**  
```go
type LockFreeStack struct {
    top unsafe.Pointer  // Атомарный указатель
}

func (s *LockFreeStack) Push(value interface{}) {
    newHead := &Node{value: value}
    for {
        oldHead := atomic.LoadPointer(&s.top)
        newHead.next = oldHead
        if atomic.CompareAndSwapPointer(&s.top, oldHead, unsafe.Pointer(newHead)) {
            return
        }
    }
}
```

### **b) Статистика в конкурентной среде**  
```go
type Metrics struct {
    requests atomic.Int64
    errors   atomic.Int32
}

func (m *Metrics) IncRequests() {
    m.requests.Add(1)
}

func (m *Metrics) Get() (int64, int32) {
    return m.requests.Load(), m.errors.Load()
}
```

### **c) Однократная инициализация (без Once)**  
```go
var initialized atomic.Bool

func initService() {
    if initialized.CompareAndSwap(false, true) {
        // Выполнится только один раз
        setup()
    }
}
```

---

## **4. Ограничения и подводные камни**  

### **a) ABA проблема**  
```go
// CAS может "успешно" сработать, даже если значение
// временно менялось между чтением и записью
```

**Решение:**  
- Использовать `atomic.Value` для сложных структур  
- Добавлять версии/метки изменений  

### **b) Нет атомарных операций для float**  
```go
// Не работает:
atomic.AddFloat64(&f, 1.0)  
```
**Обходное решение:**  
```go
var floatBits atomic.Uint64
bits := math.Float64bits(f)
newBits := math.Float64bits(f + 1.0)
floatBits.CompareAndSwap(bits, newBits)
```

### **c) Сложность отладки**  
- Атомарные ошибки могут проявляться только под нагрузкой  
- Требуют тщательного тестирования  

---

## **5. Сравнение с Mutex**  
| Критерий       | Atomic | Mutex |
|----------------|--------|-------|
| Скорость       | ~5ns   | ~25ns |
| Блокировки     | Нет    | Да    |
| Сложность      | Высокая| Низкая|
| Применимость   | Простые операции | Сложная логика |

**Правило выбора:**  
- Используйте `atomic` для:  
  - Счетчиков  
  - Флагов состояния  
  - Указателей  
- Используйте `Mutex` для:  
  - Сложных структур данных  
  - Операций, требующих нескольких изменений  

---

**Дополнительно:**  
- [Официальная документация atomic](https://pkg.go.dev/sync/atomic)  
- [Исходный код atomic](https://github.com/golang/go/blob/master/src/sync/atomic/doc.go)  
- [Модель памяти Go](https://golang.org/ref/mem)