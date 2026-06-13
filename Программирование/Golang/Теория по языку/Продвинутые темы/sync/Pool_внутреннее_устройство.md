---
tags:
  - Программирование
  - Теория_по_языку_Golang
  - Sync
  - SyncPool
  - Внутренее_устройство
Связанные темы:
  - "[[Примитивы sync]]"
---
# **Внутреннее устройство sync.Pool в Go**

## **1. Базовая структура и принцип работы**

Pool предназначен для кэширования и повторного использования временных объектов, чтобы уменьшить нагрузку на сборщик мусора. Структура организована как набор per-P (процессор) пулов для минимизации конкуренции.

```go
// Исходный код из src/sync/pool.go
type Pool struct {
    noCopy noCopy
    
    local     unsafe.Pointer // Указатель на [P]poolLocal (fixed size array)
    localSize uintptr        // Размер local массива
    
    victim     unsafe.Pointer // Предыдущий цикл local из GC
    victimSize uintptr        // Размер victim массива
    
    New func() interface{}   // Фабрика для создания новых объектов
}

// Локальный пул для каждого P
type poolLocal struct {
    poolLocalInternal
    
    pad [128 - unsafe.Sizeof(poolLocalInternal{})%128]byte // Выравнивание для false sharing
}

type poolLocalInternal struct {
    private interface{}   // Приватный объект для текущего P
    shared  poolChain     // Локальная очередь для текущего P
}
```

**Назначение полей:**

- **local**: массив poolLocal структур, по одной на каждый логический процессор (P)
- **localSize**: количество элементов в local массиве (равно GOMAXPROCS)
- **victim**: кэш предыдущего поколения объектов, переживших один цикл GC
- **victimSize**: размер victim массива
- **New**: опциональная функция для создания новых объектов при пустом пуле
- **private**: эксклюзивный объект для текущего P (нет необходимости в синхронизации)
- **shared**: двухсторонняя очередь (deque) для хранения дополнительных объектов
- **pad**: padding для предотвращения false sharing между кэш-линиями процессора

---

## **2. Алгоритм работы Get()**

Метод `Get()` извлекает объект из пула, предпочитая локальные объекты текущего процессора для минимизации синхронизации.

```go
func (p *Pool) Get() interface{} {
    if race.Enabled {
        race.Disable()
    }
    
    // Получаем локальный пул для текущего P
    l, pid := p.pin()
    
    // Пытаемся получить приватный объект
    x := l.private
    l.private = nil
    
    if x == nil {
        // Пытаемся получить из локальной очереди
        x, _ = l.shared.popHead()
        if x == nil {
            // Локальный пул пуст - пытаемся украсть у других P
            x = p.getSlow(pid)
        }
    }
    
    runtime_procUnpin()
    
    if race.Enabled {
        race.Enable()
        if x != nil {
            race.Acquire(poolRaceAddr(x))
        }
    }
    
    // Если ничего не нашли и есть New функция - создаем новый объект
    if x == nil && p.New != nil {
        x = p.New()
    }
    return x
}
```

**Детальный алгоритм getSlow():**

```go
func (p *Pool) getSlow(pid int) interface{} {
    size := atomic.LoadUintptr(&p.localSize)
    locals := p.local
    
    // Пытаемся украсть объект у других P
    for i := 0; i < int(size); i++ {
        l := indexLocal(locals, (pid+i+1)%int(size))
        if x, _ := l.shared.popTail(); x != nil {
            return x
        }
    }
    
    // Проверяем victim кэш
    size = atomic.LoadUintptr(&p.victimSize)
    if uintptr(pid) >= size {
        return nil
    }
    locals = p.victim
    l := indexLocal(locals, pid)
    
    if x := l.private; x != nil {
        l.private = nil
        return x
    }
    
    // Пытаемся украсть из victim очередей других P
    for i := 0; i < int(size); i++ {
        l := indexLocal(locals, (pid+i+1)%int(size))
        if x, _ := l.shared.popTail(); x != nil {
            return x
        }
    }
    
    // Очищаем victim кэш если он пустой
    atomic.StoreUintptr(&p.victimSize, 0)
    return nil
}
```

---

## **3. Алгоритм работы Put()**

Метод `Put()` возвращает объект в пул, предпочитая локальное хранение на текущем процессоре.

```go
func (p *Pool) Put(x interface{}) {
    if x == nil {
        return
    }
    
    if race.Enabled {
        if fastrand()%4 == 0 {
            // Рандомно сбрасываем объект для детектирования гонок
            return
        }
        race.ReleaseMerge(poolRaceAddr(x))
        race.Disable()
    }
    
    // Получаем локальный пул для текущего P
    l, _ := p.pin()
    
    if l.private == nil {
        // Приватный слот свободен - сохраняем там
        l.private = x
        x = nil
    }
    
    if x != nil {
        // Добавляем в локальную очередь
        l.shared.pushHead(x)
    }
    
    runtime_procUnpin()
    
    if race.Enabled {
        race.Enable()
    }
}
```

**Детальный алгоритм pushHead() в poolChain:**

```go
func (c *poolChain) pushHead(val interface{}) {
    d := c.head
    if d == nil {
        // Первый элемент в цепочке
        const initSize = 8 // Начальный размер deque
        d = new(poolChainElt)
        d.vals = make([]eface, initSize)
        c.head = d
        storePoolChainElt(&c.tail, d)
    }
    
    // Пытаемся добавить в текущий deque
    if d.pushHead(val) {
        return
    }
    
    // Текущий deque полон - создаем новый в 2 раза больше
    newSize := len(d.vals) * 2
    if newSize >= dequeueLimit {
        newSize = dequeueLimit
    }
    
    d2 := &poolChainElt{prev: d}
    d2.vals = make([]eface, newSize)
    c.head = d2
    storePoolChainElt(&d.next, d2)
    d2.pushHead(val)
}
```

---

## **4. Взаимодействие со сборщиком мусора**

### **Механизм victim кэширования**

Pool использует двух-поколенческую схему для выживания объектов через GC циклы:

```go
// Вызывается runtime'ом во время GC
func poolCleanup() {
    // Уничтожаем старые victim пулы
    for _, p := range oldPools {
        p.victim = nil
        p.victimSize = 0
    }
    
    // Текущие local пулы становятся victim
    for _, p := range allPools {
        p.victim = p.local
        p.victimSize = p.localSize
        p.local = nil
        p.localSize = 0
    }
    
    // Обновляем списки пулов
    oldPools, allPools = allPools, nil
}
```

**Цикл жизни объектов:**
1. **Local пул**: активные объекты текущего GC цикла
2. **Victim пул**: объекты, пережившие один GC цикл
3. **Уничтожение**: объекты, не использованные в течение двух GC циклов

### **Регистрация пулов**

```go
var (
    allPools []*Pool
    oldPools []*Pool
)

func init() {
    runtime_registerPoolCleanup(poolCleanup)
}
```

---

## **5. Структуры данных poolChain**

### **Двухсторонняя очередь (deque)**

```go
type poolChain struct {
    head *poolChainElt
    tail *poolChainElt
}

type poolChainElt struct {
    poolDequeue
    next, prev *poolChainElt
}

type poolDequeue struct {
    headTail uint64
    vals []eface
}

type eface struct {
    typ, val unsafe.Pointer
}
```

**Особенности poolChain:**

- **Динамическое расширение**: каждый следующий deque в 2 раза больше предыдущего
- **Ограничение размера**: максимальный размер `dequeueLimit = (1 << 32) / 4`
- **Lock-free операции**: использование атомарных операций для headTail

### **Атомарные операции с deque**

```go
func (d *poolDequeue) pushHead(val interface{}) bool {
    ptrs := atomic.LoadUint64(&d.headTail)
    head, tail := d.unpack(ptrs)
    
    if (tail+uint32(len(d.vals)))&(1<<dequeueBits-1) == head {
        // Deque полон
        return false
    }
    
    slot := &d.vals[head&uint32(len(d.vals)-1)]
    
    // Проверка что слот пустой
    typ := atomic.LoadPointer(&slot.typ)
    if typ != nil {
        // Все еще занят
        return false
    }
    
    // Сохраняем значение
    if val == nil {
        val = nildeleted
    }
    *(*interface{})(unsafe.Pointer(slot)) = val
    
    // Атомарно увеличиваем head
    atomic.AddUint64(&d.headTail, 1<<dequeueBits)
    return true
}
```

---

## **6. Pinning и взаимодействие с планировщиком**

### **Механизм pin/unpin**

```go
func (p *Pool) pin() (*poolLocal, int) {
    pid := runtime_procPin()
    
    // Загружаем текущий размер local
    s := atomic.LoadUintptr(&p.localSize)
    l := p.local
    
    if uintptr(pid) < s {
        // Уже инициализирован для этого P
        return indexLocal(l, pid), pid
    }
    
    // Нужно переинициализировать - требуется lock
    return p.pinSlow()
}
```

**Назначение pin:**

- **Предотвращает вытеснение горутины** с текущего P
- **Гарантирует атомарность** операций с локальным пулом
- **Минимизирует конкуренцию** между горутинами на одном P

### **Медленная инициализация pinSlow()**

```go
func (p *Pool) pinSlow() (*poolLocal, int) {
    runtime_procUnpin()
    
    // Захватываем глобальный lock всех пулов
    allPoolsMu.Lock()
    defer allPoolsMu.Unlock()
    
    pid := runtime_procPin()
    
    // Двойная проверка под lock'ом
    s := p.localSize
    l := p.local
    if uintptr(pid) < s {
        return indexLocal(l, pid), pid
    }
    
    // Инициализируем local массив
    if p.local == nil {
        allPools = append(allPools, p)
    }
    
    size := runtime.GOMAXPROCS(0)
    local := make([]poolLocal, size)
    
    atomic.StorePointer(&p.local, unsafe.Pointer(&local[0]))
    atomic.StoreUintptr(&p.localSize, uintptr(size))
    
    return &local[pid], pid
}
```

---

## **7. Оптимизации производительности**

### **False sharing prevention**

```go
type poolLocal struct {
    poolLocalInternal
    pad [128 - unsafe.Sizeof(poolLocalInternal{})%128]byte
}
```

Padding гарантирует что каждый poolLocal попадает в отдельную кэш-линию (обычно 64 или 128 байт), предотвращая contention между ядрами процессора.

### **Приоритеты доступа**

1. **private слот**: самый быстрый, нет синхронизации
2. **локальная очередь**: lock-free операции на текущем P
3. **кража у других P**: атомарные операции, но медленнее
4. **victim кэш**: объекты, пережившие GC
5. **создание нового**: через New() функцию

### **Размеры deque**

- **Начальный размер**: 8 элементов
- **Максимальный размер**: 1<<30 / 4 элементов
- **Экспоненциальный рост**: каждый новый deque в 2 раза больше

---

## **8. Обработка race conditions**

### **Детектирование гонок**

```go
func poolRaceAddr(x interface{}) unsafe.Pointer {
    ptr := uintptr((*[2]unsafe.Pointer)(unsafe.Pointer(&x))[1])
    return unsafe.Pointer(ptr &^ (1<<0 | 1<<1 | 1<<2 | 1<<3))
}
```

### **Рандомный сброс объектов**

```go
if fastrand()%4 == 0 {
    // С вероятностью 25% не кэшируем объект
    return
}
```

Это помогает обнаруживать use-after-free ошибки и гонки данных.

---

## **9. Практические примеры работы**

### **Сценарий 1: Базовое использование**
```go
var bufferPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

func getBuffer() *bytes.Buffer {
    return bufferPool.Get().(*bytes.Buffer)
}

func putBuffer(buf *bytes.Buffer) {
    buf.Reset()
    bufferPool.Put(buf)
}
```

### **Сценарий 2: Высокая конкуренция**
```go
// 1000 горутин используют общий пул
for i := 0; i < 1000; i++ {
    go func() {
        buf := getBuffer()
        defer putBuffer(buf)
        
        buf.WriteString("hello")
        // работа с буфером
    }()
}
```

**Поведение:**
- Каждая горутина работает со своим P-локальным пулом
- При нехватке объектов - кража у других P или создание новых
- После завершения - объекты возвращаются в локальные пулы

### **Сценарий 3: Влияние GC**
```go
var pool sync.Pool
pool.Put("object1")
pool.Put("object2")

runtime.GC() // Объекты перемещаются в victim кэш

// Get сначала проверит victim кэш
obj := pool.Get() // Может вернуть "object1" или "object2"

runtime.GC() // Оставшиеся объекты уничтожаются
```

---

## **10. Производительность и характеристики**

### **Временные затраты**

| Операция | Время (нс) | Условия |
|----------|------------|---------|
| Get (private hit) | 5-10 | Объект в private слоте |
| Get (local queue hit) | 15-30 | Объект в локальной очереди |
| Get (steal from other P) | 50-200 | Кража у другого процессора |
| Get (new creation) | 100-1000+ | Создание через New() |
| Put (private empty) | 5-10 | Private слот свободен |
| Put (to local queue) | 15-40 | Добавление в локальную очередь |

### **Потребление памяти**

- **Базовая структура Pool**: ~40 байт
- **Каждый poolLocal**: 128 байт (из-за padding)
- **Deque элементы**: 16 байт на элемент (eface)
- **Динамическое расширение**: зависит от размера хранимых объектов

### **Scalability**

- **Отлично масштабируется** благодаря per-P дизайну
- **Минимальная конкуренция** при работе с локальными пулами
- **Эффективное распределение** нагрузки между процессорами

---

## **11. Ограничения и предостережения**

### **Негарантированное хранение**
```go
pool.Put(obj)
runtime.GC()
val := pool.Get() // Может вернуть nil - объект мог быть собран
```

### **Типобезопасность**
```go
pool.Put(&MyStruct{})
val := pool.Get().(*MyStruct) // Необходимо приведение типов
```

### **Размер объектов**
```go
// Не эффективно для маленьких объектов
pool.Put(42) // Накладные расходы больше чем выгода

// Эффективно для больших объектов
pool.Put(make([]byte, 1024*1024)) // Экономит аллокации
```

---

**Ссылки на исходный код:**
- [sync/pool.go](https://github.com/golang/go/blob/master/src/sync/pool.go)
- [sync/poolqueue.go](https://github.com/golang/go/blob/master/src/sync/poolqueue.go)
- [Runtime интеграция](https://github.com/golang/go/blob/master/src/runtime/mgc.go)