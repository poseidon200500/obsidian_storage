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
# **Внутреннее устройство sync.Cond в Go**

## **1. Базовая структура и принцип работы**

Cond (condition variable) предоставляет механизм для ожидания и сигнализации о наступлении событий между горутинами. Это самый сложный примитив синхронизации в пакете sync, построенный на основе мьютексов и семафоров.

```go
// Исходный код из src/sync/cond.go
type Cond struct {
    noCopy noCopy
    
    L Locker // Мьютекс, связанный с условием (обычно *Mutex или *RWMutex)
    
    notify  notifyList // Очередь ожидающих горутин
	checker copyChecker // Проверка на копирование в runtime
}

// Очередь ожидания - реализована как lock-free linked list
type notifyList struct {
    wait   uint32 // Счетчик для ожидания ( ticket-механизм )
    notify uint32 // Счетчик для уведомления
    lock   uintptr // Внутренний spin lock
    head   unsafe.Pointer // Голова списка ожидания
    tail   unsafe.Pointer // Хвост списка ожидания
}
```

**Назначение полей:**

- **L (Locker)**: связанный мьютекс, который должен быть захвачен перед вызовом Wait()
- **notify (notifyList)**: очередь ожидающих горутин, организованная как lock-free linked list
- **checker (copyChecker)**: runtime проверка против копирования структуры
- **wait/notify**: счетчики для реализации ticket-механизма (подобно очередям в банке)
- **lock**: внутренний spin lock для синхронизации доступа к очереди
- **head/tail**: указатели на начало и конец очереди ожидания

---

## **2. Алгоритм работы Wait()**

Метод `Wait()` блокирует текущую горутину до получения сигнала. Он атомарно освобождает мьютекс и переводит горутину в ожидание.

```go
func (c *Cond) Wait() {
    // Проверка на копирование в runtime
    c.checker.check()
    
    // Получаем "билет" для места в очереди ожидания
    t := runtime_notifyListAdd(&c.notify)
    
    // Освобождаем связанный мьютекс
    c.L.Unlock()
    
    // Ожидаем сигнала (блокируем горутину)
    runtime_notifyListWait(&c.notify, t)
    
    // После пробуждения снова захватываем мьютекс
    c.L.Lock()
}
```

**Детальный алгоритм runtime_notifyListWait:**

```go
// runtime/sema.go
func notifyListAdd(l *notifyList) uint32 {
    // Атомарно увеличиваем счетчик wait и возвращаем ticket
    return atomic.Xadd(&l.wait, 1) - 1
}

func notifyListWait(l *notifyList, t uint32) {
    // Захватываем внутренний spin lock
    lock(&l.lock)
    
    // Проверяем не пришел ли уже наш сигнал
    if t < l.notify {
        // Уже уведомлены - выходим
        unlock(&l.lock)
        return
    }
    
    // Создаем элемент очереди ожидания
    s := acquireSudog()
    s.g = getg()
    s.ticket = t
    
    // Добавляем в конец очереди
    if l.tail == nil {
        l.head = s
    } else {
        l.tail.next = s
    }
    l.tail = s
    
    // Освобождаем lock и переводим горутину в ожидание
    goparkunlock(&l.lock, waitReasonSyncCondWait, traceEvGoBlockCond, 3)
    
    // После пробуждения освобождаем sudog
    releaseSudog(s)
}
```

**Пошаговое выполнение Wait():**

1. **Получение ticket**: атомарное увеличение счетчика `wait`
2. **Освобождение мьютекса**: связанный мьютекс освобождается
3. **Добавление в очередь**: горутина помещается в lock-free очередь ожидания
4. **Блокировка**: вызов `goparkunlock` переводит горутину в состояние ожидания
5. **Пробуждение**: при получении сигнала горутина пробуждается и перезахватывает мьютекс

---

## **3. Алгоритм работы Signal() и Broadcast()**

### **Signal() - пробуждение одной горутины**

```go
func (c *Cond) Signal() {
    c.checker.check()
    runtime_notifyListNotifyOne(&c.notify)
}
```

**Детальный алгоритм notifyListNotifyOne:**

```go
func notifyListNotifyOne(l *notifyList) {
    // Атомарно увеличиваем счетчик notify
    atomic.Xadd(&l.notify, 1)
    
    // Быстрая проверка - есть ли ожидающие?
    if atomic.Load(&l.wait) == atomic.Load(&l.notify) {
        return
    }
    
    // Захватываем lock очереди
    lock(&l.lock)
    
    // Ищем следующую горутину для пробуждения
    t := l.notify
    if t == atomic.Load(&l.wait) {
        unlock(&l.lock)
        return
    }
    
    // Ищем горутину с ticket == t
    var prev *sudog
    for p := l.head; p != nil; prev, p = p, p.next {
        if p.ticket == t {
            // Нашли - удаляем из очереди и пробуждаем
            n := p.next
            if prev != nil {
                prev.next = n
            } else {
                l.head = n
            }
            if n == nil {
                l.tail = prev
            }
            unlock(&l.lock)
            p.next = nil
            readyWithTime(p, 4) // Пробуждаем горутину
            return
        }
    }
    unlock(&l.lock)
}
```

### **Broadcast() - пробуждение всех горутин**

```go
func (c *Cond) Broadcast() {
    c.checker.check()
    runtime_notifyListNotifyAll(&c.notify)
}
```

**Детальный алгоритм notifyListNotifyAll:**

```go
func notifyListNotifyAll(l *notifyList) {
    // Устанавливаем notify в значение wait (пробуждаем всех)
    atomic.Store(&l.notify, atomic.Load(&l.wait))
    
    // Захватываем lock очереди
    lock(&l.lock)
    
    // Пробуждаем все горутины в очереди
    for s := l.head; s != nil; s = s.next {
        next := s.next
        s.next = nil
        readyWithTime(s, 4) // Пробуждаем каждую горутину
    }
    
    // Очищаем очередь
    l.head = nil
    l.tail = nil
    unlock(&l.lock)
}
```

---

## **4. Структура очереди ожидания notifyList**

### **Элемент очереди sudog**

```go
// runtime/runtime2.go
type sudog struct {
    g *g           // Горутина
    next *sudog    // Следующий элемент
    prev *sudog    // Предыдущий элемент
    elem unsafe.Pointer // Элемент данных
    ticket uint32  // Номер билета для условия
    // ... другие поля
}
```

### **Lock-free linked list**

Очередь организована как singly-linked list с lock-based модификациями:

```
head → sudog1 → sudog2 → sudog3 → tail
```

**Операции с очередью:**

- **Добавление**: добавление в хвост под защитой spin lock
- **Удаление**: удаление из головы или середины при пробуждении
- **Обход**: итерация по списку для поиска горутин по ticket

---

## **5. Взаимодействие с планировщиком**

### **Блокировка горутины**

```go
func goparkunlock(lock *mutex, reason waitReason, traceEv byte, traceskip int) {
    gopark(parkunlock_c, unsafe.Pointer(lock), reason, traceEv, traceskip)
}
```

**Процесс блокировки:**

1. **Захват планировщика**: `acquirem()` для предотвращения вытеснения
2. **Изменение статуса**: установка горутины в `Gwaiting`
3. **Освобождение ресурсов**: unlock переданного мьютекса
4. **Перепланирование**: вызов `schedule()` для переключения на другую горутину

### **Пробуждение горутины**

```go
func readyWithTime(s *sudog, traceskip int) {
    if s.g != nil {
        readyG(s.g, traceskip, true)
    }
}

func readyG(gp *g, traceskip int, next bool) {
    // Переводим горутину в Runnable состояние
    status := readgstatus(gp)
    if status&^_Gscan != _Gwaiting {
        dumpgstatus(gp)
        throw("bad g status")
    }
    
    // Помещаем в локальную очередь выполнения
    casgstatus(gp, _Gwaiting, _Grunnable)
    runqput(getg().m.p.ptr(), gp, next)
    
    // Если есть свободные процессоры - будим
    if atomic.Load(&sched.npidle) != 0 && atomic.Load(&sched.nmspinning) == 0 {
        wakep()
    }
}
```

---

## **6. Ticket-механизм и справедливость**

### **Принцип работы ticket системы**

```go
// При вызове Wait():
t := atomic.Xadd(&l.wait, 1) - 1  // Получаем уникальный ticket

// При вызове Signal():
atomic.Xadd(&l.notify, 1)         // Увеличиваем счетчик уведомлений

// При пробуждении:
if t < l.notify {                 // Проверяем наш ticket
    // Нас уведомили - выходим
}
```

**Аналогия с банковской очередью:**
- `wait`: номер последнего взятого талона
- `notify`: номер последнего обслуженного талона
- Горутина пробуждается когда `ticket == notify`

### **Гарантии справедливости**

- **FIFO порядок**: горутины пробуждаются в порядке вызова Wait()
- **No starvation**: каждая горутина гарантированно получит сигнал
- **Atomic операции**: гарантии памяти через атомарные счетчики

---

## **7. Оптимизации производительности**

### **Быстрая проверка в Signal()**

```go
// Быстрая проверка без захвата lock
if atomic.Load(&l.wait) == atomic.Load(&l.notify) {
    return // Нет ожидающих - сразу выходим
}
```

### **Lock-free чтение счетчиков**

Использование атомарных операций для чтения wait/notify:
- `atomic.Load()` для чтения без блокировок
- `atomic.Xadd()` для атомарного увеличения
- Memory barriers для гарантий видимости

### **Spin lock для коротких операций**

```go
lock(&l.lock)   // Кратковременный захват
// операции с очередью
unlock(&l.lock) // Быстрое освобождение
```

---

## **8. Обработка ошибок и проверки**

### **Проверка на копирование**

```go
type copyChecker uintptr

func (c *copyChecker) check() {
    if uintptr(*c) != uintptr(unsafe.Pointer(c)) &&
        !atomic.CompareAndSwapUintptr((*uintptr)(c), 0, uintptr(unsafe.Pointer(c))) &&
        uintptr(*c) != uintptr(unsafe.Pointer(c)) {
        panic("sync.Cond is copied")
    }
}
```

**Принцип работы:**
- При первом использовании сохраняется указатель на себя
- При копировании указатель не совпадает - паника

### **Проверки в Wait()**

```go
func (c *Cond) Wait() {
    // Неявная проверка что мьютекс захвачен
    c.L.Unlock() // Если не захвачен - паника
    
    // После пробуждения
    c.L.Lock()   // Перезахватываем мьютекс
}
```

---

## **9. Практические примеры работы**

### **Сценарий 1: Producer-Consumer с одним consumer**
```go
var cond = sync.NewCond(&sync.Mutex{})
var queue []int

func producer() {
    cond.L.Lock()
    queue = append(queue, 1)
    cond.Signal() // Пробуждаем одного consumer
    cond.L.Unlock()
}

func consumer() {
    cond.L.Lock()
    for len(queue) == 0 {
        cond.Wait() // Освобождает мьютекс и ждет
    }
    item := queue[0]
    queue = queue[1:]
    cond.L.Unlock()
    process(item)
}
```

### **Сценарий 2: Broadcast для multiple consumers**
```go
var cond = sync.NewCond(&sync.Mutex{})
var dataReady bool

func broadcaster() {
    cond.L.Lock()
    dataReady = true
    cond.Broadcast() // Пробуждаем всех ожидающих
    cond.L.Unlock()
}

func worker(id int) {
    cond.L.Lock()
    for !dataReady {
        cond.Wait() // Ждем сигнала
    }
    cond.L.Unlock()
    fmt.Printf("Worker %d started\n", id)
}
```

### **Сценарий 3: Complex condition с timeout**
```go
func waitWithTimeout(cond *sync.Cond, condition func() bool, timeout time.Duration) bool {
    cond.L.Lock()
    defer cond.L.Unlock()
    
    deadline := time.Now().Add(timeout)
    for !condition() {
        if time.Now().After(deadline) {
            return false
        }
        // Ждем сигнала или таймаута
        cond.Wait()
    }
    return true
}
```

---

## **10. Производительность и характеристики**

### **Временные затраты**

| Операция | Время (нс) | Условия |
|----------|------------|---------|
| Wait() (блокировка) | 50-100 | Захват lock, добавление в очередь |
| Wait() (пробуждение) | 100-200 | Восстановление в очереди планировщика |
| Signal() (нет ожидающих) | 5-10 | Быстрая проверка счетчиков |
| Signal() (с ожидающими) | 20-50 | Поиск и пробуждение горутины |
| Broadcast() | 50-200 + N*20 | Зависит от количества ожидающих |

### **Потребление памяти**

- **Базовая структура Cond**: 24-32 байта
- **Каждая ожидающая горутина**: ~200 байт (sudog + структуры планировщика)
- **Очередь notifyList**: 20 байт + N*sudog

### **Scalability**

- **Хорошая масштабируемость** при редких сигналах
- **Contention на мьютексе** при частых операциях
- **Эффективный Broadcast** благодаря batch пробуждению

---

## **11. Сравнение с альтернативами**

### **Cond vs Channel**
```go
// С Cond
var cond = sync.NewCond(&sync.Mutex{})
cond.L.Lock()
for !condition {
    cond.Wait()
}
cond.L.Unlock()

// С Channel
ch := make(chan struct{})
// Где-то: close(ch) или ch <- struct{}{}
<-ch
```

**Преимущества Cond:**
- Переиспользование мьютекса для сложных условий
- Broadcast для multiple receivers
- Меньше аллокаций памяти

### **Cond vs WaitGroup**
```go
// Cond для сложных условий
cond.L.Lock()
for !complexCondition() {
    cond.Wait()
}
cond.L.Unlock()

// WaitGroup для простого ожидания
wg.Wait()
```

---

## **12. Особые случаи и предостережения**

### **Spurious wakeups защита**
```go
// Всегда используйте цикл с условием
for !condition {
    cond.Wait() // Защита от случайных пробуждений
}
```

### **Порядок операций с мьютексом**
```go
// НЕПРАВИЛЬНО:
cond.L.Lock()
cond.Wait() // Забыли проверить условие
cond.L.Unlock()

// ПРАВИЛЬНО:
cond.L.Lock()
for !condition {
    cond.Wait()
}
cond.L.Unlock()
```

### **Broadcast vs Signal**
```go
// Используйте Signal когда достаточно одной горутины
if len(queue) > 0 {
    cond.Signal() // Только один consumer нужен
}

// Используйте Broadcast когда нужно всем
dataReady = true
cond.Broadcast() // Все workers могут начать
```

---

**Ссылки на исходный код:**
- [sync/cond.go](https://github.com/golang/go/blob/master/src/sync/cond.go)
- [runtime/sema.go](https://github.com/golang/go/blob/master/src/runtime/sema.go) (notifyList функции)
- [runtime/proc.go](https://github.com/golang/go/blob/master/src/runtime/proc.go) (gopark/ready)
- [runtime/runtime2.go](https://github.com/golang/go/blob/master/src/runtime/runtime2.go) (sudog структура)
- https://ubiklab.net/posts/go-sync-cond/ (классная статья с объяснением)