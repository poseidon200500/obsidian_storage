---
tags:
  - Шаблоны
  - Программирование
  - Теория_по_языку_Golang
Связанные темы:
---
# **Управление и настройка GC в Go**  
*Программные интерфейсы и переменные окружения для контроля сборщика мусора*  

---

## **Переменные окружения для управления GC**

### **GOGC - основной контроллер агрессивности GC**

**Назначение:** Определяет процент роста heap'а между циклами GC  
**Формат:** числовое значение или `off`  
**По умолчанию:** `100`

```bash
# Запуск с различными настройками GOGC
GOGC=50 ./app    # Агрессивный GC
GOGC=100 ./app   # По умолчанию  
GOGC=200 ./app   # Менее агрессивный
GOGC=off ./app   # Полное отключение авто-GC
```

**Значения и их эффект:**
- `GOGC=50` - GC запускается при росте heap'а на 50%
- `GOGC=100` - при росте на 100% (удвоение)
- `GOGC=200` - при росте на 200% (утроение)
- `GOGC=off` - автоматический GC отключен

### **GODEBUG - расширенная диагностика**

**Назначение:** Включение детальной трассировки и отладочной информации

```bash
# Базовая трассировка GC
GODEBUG=gctrace=1 ./app

# Комплексная диагностика
GODEBUG=gctrace=1,gcpacertrace=1 ./app

# Для приложений с большим количеством горутин
GODEBUG=gctrace=1,scavtrace=1 ./app
```

**Доступные флаги:**
- `gctrace=1` - трассировка каждого цикла GC
- `gcpacertrace=1` - трассировка pacing алгоритма
- `scavtrace=1` - трассировка освобождения памяти
- `madvdontneed=1` - использование madvise вместо madvise_dontneed

### **GOMEMLIMIT - управление лимитом памяти (Go 1.19+)**

**Назначение:** Установка мягкого лимита памяти для всего процесса  
**Формат:** размер в байтах с суффиксом (500MiB, 2GiB)

```bash
# Установка лимита памяти
GOMEMLIMIT=500MiB ./app
GOMEMLIMIT=2GiB ./app

# Отключение лимита
GOMEMLIMIT=0 ./app
```

---

## **Программные интерфейсы управления GC**

### **Пакет runtime/debug - основное API**

```go
import "runtime/debug"
```

#### **SetGCPercent - управление агрессивностью GC**

```go
// Установка целевого процента роста heap'а
func SetGCPercent(percent int) int

// Примеры использования:
func main() {
    // Агрессивный GC - запуск при росте на 50%
    previous := debug.SetGCPercent(50)
    
    // Менее агрессивный - рост на 200%
    debug.SetGCPercent(200)
    
    // Полное отключение автоматического GC
    debug.SetGCPercent(-1)
    
    // Восстановление предыдущего значения
    debug.SetGCPercent(previous)
}
```

#### **SetMemoryLimit - мягкий лимит памяти (Go 1.19+)**

```go
// Установка мягкого лимита памяти
func SetMemoryLimit(limit int64) int64

// Примеры использования:
func main() {
    // Лимит 512 MB
    debug.SetMemoryLimit(512 * 1024 * 1024)
    
    // Лимит 1 GB
    debug.SetMemoryLimit(1 << 30)
    
    // Отключение лимита
    debug.SetMemoryLimit(0)
    
    // Автоматическое определение для контейнеров
    setContainerAwareLimits()
}

func setContainerAwareLimits() {
    // В production можно определить лимит из cgroups
    if cgroupLimit := getCgroupMemoryLimit(); cgroupLimit > 0 {
        // Оставить 10% запаса
        limit := cgroupLimit * 90 / 100
        debug.SetMemoryLimit(limit)
    }
}
```

#### **FreeOSMemory - принудительное освобождение памяти**

```go
// Принудительный возврат памяти ОС
debug.FreeOSMemory()

// Использование в сценариях:
func processBatch(data []byte) {
    // Обработка данных...
    process(data)
    
    // После обработки больших данных освобождаем память
    debug.FreeOSMemory()
}
```

### **Пакет runtime - низкоуровневое управление**

```go
import "runtime"
```

#### **Принудительный запуск GC**

```go
// Немедленный запуск полного цикла GC
runtime.GC()

// Примеры использования:
func criticalOperation() {
    // Перед критической операцией очищаем память
    runtime.GC()
    
    performCriticalOperation()
}

func memorySensitiveTask() {
    // После работы с большими данными
    processLargeDataset()
    runtime.GC()
    
    // Перед переходом к следующей задаче
    prepareNextTask()
}
```

#### **ReadMemStats - мониторинг статистики памяти**

```go
// Получение детальной статистики памяти
var stats runtime.MemStats
runtime.ReadMemStats(&stats)

// Ключевые метрики для мониторинга:
fmt.Printf("Alloc: %v\n", stats.Alloc)           // Текущие аллокации
fmt.Printf("TotalAlloc: %v\n", stats.TotalAlloc) // Всего аллоцировано
fmt.Printf("Sys: %v\n", stats.Sys)               // Память от ОС
fmt.Printf("NumGC: %v\n", stats.NumGC)           // Количество GC
fmt.Printf("PauseTotalNs: %v\n", stats.PauseTotalNs) // Суммарное время пауз
```

### **Управление через контекст (Go 1.20+)**

```go
import "context"

// Создание контекста с меткой для трассировки GC
ctx := context.WithValue(context.Background(), "gc.trace", "memory_intensive_operation")

// Использование в функциях, которые могут влиять на GC
func processWithGCContext(ctx context.Context, data []byte) {
    if trace, ok := ctx.Value("gc.trace").(string); ok {
        // Логирование для анализа GC поведения
        log.Printf("GC trace: %s", trace)
    }
    processData(data)
}
```

---

## **Важные замечания по использованию**

### **Взаимодействие параметров**

- `GOGC` и `GOMEMLIMIT` работают совместно - GC пытается удовлетворить оба ограничения
- `SetGCPercent()` переопределяет переменную окружения `GOGC`
- `SetMemoryLimit()` переопределяет `GOMEMLIMIT`
