---
tags:
  - Шаблоны
  - Программирование
  - Теория_по_языку_Golang
  - Sync
  - SyncMap
  - map
Связанные темы:
---
# **Внутреннее устройство sync.Map в Go**

## **1. Базовая структура и принцип работы**

sync.Map - это потокобезопасная реализация map, оптимизированная для двух основных сценариев: (1) когда записи происходят редко, а чтения часто, и (2) когда разные горутины работают с непересекающимися наборами ключей. Структура использует технику copy-on-write и атомарные указатели для минимизации блокировок.

```go
// Исходный код из src/sync/map.go
type Map struct {
    mu Mutex // Мьютекс для защиты dirty мапы
    
    // read содержит основную часть мапы, доступную для чтения без блокировок
    read atomic.Pointer[readOnly]
    
    // dirty содержит полную копию мапы, используется при записях
    dirty map[any]*entry
    
    // misses подсчитывает промахи в read, для определения когда переносить dirty в read
    misses int
}

// readOnly - неизменяемое представление мапы
type readOnly struct {
    m       map[any]*entry
    amended bool // true если dirty содержит ключи отсутствующие в read
}

// entry представляет значение в мапе
type entry struct {
    p atomic.Pointer[any] // Указатель на значение (может быть expunged маркером)
}
```

**Специальные указатели в entry:**
```go
var (
    nilPtr     = new(any)    // nil значение
    expunged   = new(any)    // маркер что ключ удален из dirty
)
```

**Назначение полей:**

- **mu (Mutex)**: защищает доступ к dirty мапе при записях
- **read (atomic.Pointer)**: атомарный указатель на readOnly структуру для lock-free чтения
- **dirty (map)**: полная версия мапы, защищенная мьютексом
- **misses (int)**: счетчик промахов при чтении для определения момента продвижения dirty в read
- **amended (bool)**: флаг указывающий что dirty содержит ключи отсутствующие в read
- **p (atomic.Pointer)**: атомарный указатель на значение в entry, может содержать специальные маркеры

---

## **2. Алгоритм работы Load()**

Метод `Load()` выполняет поиск значения по ключу, оптимизированный для lock-free чтения из read мапы.

```go
func (m *Map) Load(key any) (value any, ok bool) {
    // Сначала пытаемся найти в read мапе (lock-free)
    read := m.loadReadOnly()
    if e, ok := read.m[key]; ok {
        // Ключ найден в read - возвращаем значение
        return e.load()
    }
    
    // Ключа нет в read, но есть в dirty (требуется блокировка)
    if read.amended {
        m.mu.Lock()
        
        // Двойная проверка под мьютексом
        read = m.loadReadOnly()
        if e, ok := read.m[key]; ok {
            // Ключ появился в read после первой проверки
            m.mu.Unlock()
            return e.load()
        }
        
        // Ищем в dirty мапе
        if read.amended {
            if e, ok := m.dirty[key]; ok {
                // Нашли в dirty - увеличиваем счетчик промахов
                m.missLocked()
                m.mu.Unlock()
                return e.load()
            }
        }
        m.mu.Unlock()
    }
    
    return nil, false
}
```

**Детальный алгоритм entry.load():**

```go
func (e *entry) load() (value any, ok bool) {
    p := e.p.Load()
    if p == nil || p == expunged {
        // Значение удалено или экспанджировано
        return nil, false
    }
    return *p, true
}
```

**Детальный алгоритм missLocked():**

```go
func (m *Map) missLocked() {
    m.misses++
    if m.misses < len(m.dirty) {
        return
    }
    
    // Слишком много промахов - продвигаем dirty в read
    m.read.Store(&readOnly{m: m.dirty})
    m.dirty = nil
    m.misses = 0
}
```

---

## **3. Алгоритм работы Store()**

Метод `Store()` устанавливает значение для ключа, используя оптимистичную блокировку и copy-on-write.

```go
func (m *Map) Store(key, value any) {
    // Сначала проверяем существование в read (lock-free)
    read := m.loadReadOnly()
    if e, ok := read.m[key]; ok && e.tryStore(&value) {
        return // Успешно обновили существующий entry
    }
    
    // Требуется блокировка для работы с dirty
    m.mu.Lock()
    defer m.mu.Unlock()
    
    read = m.loadReadOnly()
    
    // Обновляем существующий entry в read
    if e, ok := read.m[key]; ok {
        if e.unexpungeLocked() {
            // Entry был expunged - добавляем в dirty
            m.dirty[key] = e
        }
        e.storeLocked(&value)
    } else if e, ok := m.dirty[key]; ok {
        // Ключ существует в dirty - обновляем
        e.storeLocked(&value)
    } else {
        // Новый ключ
        if !read.amended {
            // Инициализируем dirty если нужно
            m.dirtyLocked()
            m.read.Store(&readOnly{m: read.m, amended: true})
        }
        m.dirty[key] = newEntry(value)
    }
}
```

**Детальный алгоритм dirtyLocked():**

```go
func (m *Map) dirtyLocked() {
    if m.dirty != nil {
        return // Уже инициализирована
    }
    
    read := m.loadReadOnly()
    m.dirty = make(map[any]*entry, len(read.m))
    
    // Копируем все не-expunged entries из read в dirty
    for k, e := range read.m {
        if !e.tryExpungeLocked() {
            m.dirty[k] = e
        }
    }
}
```

**Детальный алгоритм tryExpungeLocked():**

```go
func (e *entry) tryExpungeLocked() (isExpunged bool) {
    p := e.p.Load()
    for p == nil {
        // Пытаемся пометить как expunged если nil
        if e.p.CompareAndSwap(nil, expunged) {
            return true
        }
        p = e.p.Load()
    }
    return p == expunged
}
```

---

## **4. Алгоритм работы Delete()**

Метод `Delete()` удаляет ключ из мапы, используя мягкое удаление через пометку entry.

```go
func (m *Map) Delete(key any) {
    m.LoadAndDelete(key) // Delete делегирует к LoadAndDelete
}

func (m *Map) LoadAndDelete(key any) (value any, loaded bool) {
    read := m.loadReadOnly()
    
    // Пытаемся удалить из read (lock-free)
    if e, ok := read.m[key]; ok {
        if e.delete() {
            return e.load()
        }
    }
    
    // Удаление из dirty (требуется блокировка)
    if read.amended {
        m.mu.Lock()
        
        // Двойная проверка
        read = m.loadReadOnly()
        if e, ok := read.m[key]; ok {
            if e.delete() {
                loaded = true
                value, _ = e.load()
            }
        } else if read.amended {
            // Удаляем из dirty
            if e, ok := m.dirty[key]; ok {
                loaded = true
                value, _ = e.load()
                delete(m.dirty, key)
                m.missLocked()
            }
        }
        
        m.mu.Unlock()
    }
    
    return value, loaded
}
```

**Детальный алгоритм entry.delete():**

```go
func (e *entry) delete() (hadValue bool) {
    for {
        p := e.p.Load()
        if p == nil || p == expunged {
            return false // Уже удален
        }
        if e.p.CompareAndSwap(p, nil) {
            return p != nil
        }
    }
}
```

---

## **5. Алгоритм работы Range()**

Метод `Range()` итерируется по всем элементам мапы, обеспечивая консистентное представление.

```go
func (m *Map) Range(f func(key, value any) bool) {
    read := m.loadReadOnly()
    
    // Если dirty содержит дополнительные ключи, нужна блокировка
    if read.amended {
        m.mu.Lock()
        
        // Продвигаем dirty в read если нужно
        if m.misses > len(m.dirty) {
            m.read.Store(&readOnly{m: m.dirty})
            m.dirty = nil
            m.misses = 0
        }
        read = m.loadReadOnly()
        m.mu.Unlock()
    }
    
    // Итерируемся по read мапе (lock-free)
    for k, e := range read.m {
        v, ok := e.load()
        if !ok {
            continue // Пропускаем удаленные
        }
        if !f(k, v) {
            break // Пользователь прервал итерацию
        }
    }
}
```

---

## **6. Состояния entry и переходы**

### **Состояния entry.p:**

1. **nil**: значение было удалено, но entry еще в read мапе
2. **expunged**: значение удалено и entry не присутствует в dirty мапе  
3. **указатель на значение**: нормальное состояние

### **Диаграмма переходов состояний:**

```
Новый entry → [указатель] ←→ [nil] → [expunged]
      ↓           ↑            ↓
   Store()     Store()     dirtyLocked()
   Delete()    Delete()    
```

**Правила переходов:**

- **nil → expunged**: при инициализации dirty копируются только не-expunged entries
- **expunged → указатель**: при Store() для expunged entry
- **указатель → nil**: при Delete() операции

---

## **7. Механизм продвижения dirty в read**

### **Условие продвижения:**

```go
if m.misses < len(m.dirty) {
    return // Недостаточно промахов для продвижения
}
```

**Логика продвижения:**

- **misses**: счетчик неудачных поисков в read
- **len(m.dirty)**: количество элементов в dirty
- **Порог**: misses должно достичь размера dirty мапы

### **Процесс продвижения:**

1. **Атомарная замена**: `m.read.Store(&readOnly{m: m.dirty})`
2. **Очистка dirty**: `m.dirty = nil`
3. **Сброс счетчика**: `m.misses = 0`
4. **Сброс флага**: amended автоматически становится false

---

## **8. Атомарные операции и memory model**

### **Используемые атомарные примитивы**

```go
// Для read указателя
m.read.Load()
m.read.Store()

// Для entry значений  
e.p.Load()
e.p.Store()
e.p.CompareAndSwap()
```

### **Гарантии памяти**

- **read pointer**: атомарные операции обеспечивают consistency между горутинами
- **entry pointers**: CAS операции гарантируют atomic updates
- **Memory barriers**: автоматически вставляются атомарными операциями

### **Happens-before отношения**

```
Store() → Load() : запись видна последующим чтениям
Delete() → Load() : удаление видно последующим операциям
dirty promotion → Range() : продвижение видно при итерации
```

---

## **9. Оптимизации производительности**

### **Lock-free чтение**

```go
// 90%+ операций выполняются без блокировок
if e, ok := read.m[key]; ok {
    return e.load() // NO LOCK
}
```

### **Copy-on-write для записей**

- **read мапа**: неизменяемая, разделяемая между горутинами
- **dirty мапа**: приватная копия для модификаций
- **Продвижение**: атомарная замена read когда dirty стабилизируется

### **Lazy инициализация dirty**

```go
func (m *Map) dirtyLocked() {
    if m.dirty != nil { return }
    // Создается только при первой записи после продвижения
}
```

### **Амортизация стоимости копирования**

- **Множественные записи**: используют одну копию dirty
- **Продвижение**: происходит после достаточного количества чтений
- **Экономия памяти**: dirty = nil когда не нужна

---

## **10. Практические примеры работы**

### **Сценарий 1: Read-mostly workload**
```go
var m sync.Map

// Инициализация (редкие записи)
m.Store("config1", "value1")
m.Store("config2", "value2")

// Множественные concurrent чтения (lock-free)
for i := 0; i < 1000; i++ {
    go func() {
        v, _ := m.Load("config1")
        _ = v
    }()
}
```

### **Сценарий 2: Disjoint key sets**
```go
// Разные горутины работают с разными ключами
for i := 0; i < 10; i++ {
    go func(id int) {
        key := fmt.Sprintf("worker%d", id)
        m.Store(key, id)
        v, _ := m.Load(key)
        // Операции только со своим ключом
    }(i)
}
```

### **Сценарий 3: Entry жизненный цикл**
```go
m.Store("key", "value")    // entry: указатель → значение
m.Delete("key")            // entry: указатель → nil
m.Store("key", "value2")   // entry: nil → указатель

m2.Store("key", "value")   // entry: указатель → значение  
m2.Delete("key")           // entry: указатель → nil
// При следующей записи: nil → expunged (если dirty пересоздается)
```

---

## **11. Производительность и характеристики**

### **Временные затраты**

| Операция | Время (нс) | Условия |
|----------|------------|---------|
| Load() (read hit) | 5-15 | Lock-free, в read мапе |
| Load() (read miss) | 50-150 | Блокировка, поиск в dirty |
| Store() (read hit) | 10-25 | Lock-free CAS в entry |
| Store() (new key) | 100-300 | Блокировка, копирование dirty |
| Delete() (read hit) | 10-30 | Lock-free CAS в entry |
| Range() (no amended) | 20-50 * N | Lock-free итерация |

### **Потребление памяти**

- **Базовая структура**: ~40 байт
- **Каждый entry**: 8 байт (atomic.Pointer) + размер ключа/значения
- **Двойное хранение**: read и dirty могут содержать дубликаты entries
- **Временные пики**: при копировании dirty потребление удваивается

### **Scalability характеристика**

- **Чтения**: отлично масштабируются благодаря lock-free доступу
- **Записи**: умеренная масштабируемость из-за мьютекса на dirty
- **Продвижение**: O(N) операция, но амортизированная по многим операциям

---

## **12. Сравнение с альтернативами**

### **sync.Map vs map + Mutex/RWMutex**

```go
// sync.Map
var sm sync.Map
sm.Store("key", "value")
v, _ := sm.Load("key")

// map + RWMutex  
var mu sync.RWMutex
var m map[string]any
mu.Lock()
m["key"] = "value"
mu.Unlock()

mu.RLock()
v := m["key"]
mu.RUnlock()
```

**Преимущества sync.Map:**
- Лучшая производительность при read-heavy workload
- Лучшая масштабируемость для disjoint key access
- Меньше contention при конкурентных чтениях

**Недостатки sync.Map:**
- Хуже при write-heavy workload
- Больше потребление памяти
- Сложнее API

### **sync.Map vs concurrent map sharding**

```go
// Sharded map
type ShardedMap []map[any]any

// sync.Map проще в использовании но менее гибкая
```

---

## **13. Ограничения и рекомендации**

### **Типобезопасность**
```go
var m sync.Map
m.Store("key", 123)
v, _ := m.Load("key")
num := v.(int) // Требуется приведение типов
```

### **Memory leaks**
```go
// Entry с nil значением остается в read мапе
m.Store("key", largeObject)
m.Delete("key") // largeObject освобождается, но entry остается

// Для полного удаления нужно продвижение dirty
m.Store("other", "value") // Может вызвать продвижение
```

### **Оптимальные сценарии использования**

**Хорошо для:**
- Кэширования конфигураций
- Metadata хранилищ
- Read-heavy workload с редкими обновлениями

**Плохо для:**
- Write-heavy workload
- Частых обновлений одних и тех же ключей
- Когда нужны batch операции

---

**Ссылки на исходный код:**
- [sync/map.go](https://github.com/golang/go/blob/master/src/sync/map.go)
- [sync/atomic/type.go](https://github.com/golang/go/blob/master/src/sync/atomic/type.go)
- [Тесты sync.Map](https://github.com/golang/go/blob/master/src/sync/map_test.go)