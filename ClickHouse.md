---
tags:
  - Шаблоны
  - Программирование
  - "#БазыДанных"
  - "#ClickHouse"
Связанные темы:
---

Отлично! Вот страница по ClickHouse в точном формате образца:

### **ClickHouse - колоночная СУБД для аналитики**  
**Файл:** `clickhouse.md`  
**Теги:** `#базы_данных #OLAP #колоночные_СУБД #аналитика`  

---

## **1. Основные понятия**  
ClickHouse — это **высокопроизводительная колоночная СУБД**, разработанная для обработки аналитических запросов (OLAP) в реальном времени.

### **Ключевая философия**  
- **Оптимизация под чтение**: 100x быстрее PostgreSQL для агрегаций  
- **Колоночное хранение**: Данные хранятся по столбцам, а не строкам  
- **Массовая параллельная обработка**: Распределенное выполнение запросов  

### **Базовый синтаксис**  
```sql
-- Создание таблицы
CREATE TABLE analytics.events
(
    timestamp DateTime,
    user_id UInt32,
    event_type String,
    value Float64
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (timestamp, user_id);
```

---

## **2. Типы движков таблиц**  

### **a) MergeTree (основной)**  
```sql
ENGINE = MergeTree()
ORDER BY (timestamp, user_id)
PARTITION BY toYYYYMM(timestamp)
```
- **Сортировка данных** по первичному ключу  
- **Партиционирование** для эффективного управления данными  
- **Поддержка репликации** через ReplicatedMergeTree  

### **b) AggregatingMergeTree**  
```sql
ENGINE = AggregatingMergeTree()
ORDER BY (date, metric)
```
- **Автоматическая агрегация** данных при слиянии  
- Используется для предварительно агрегированных данных  

### **c) Distributed**  
```sql
ENGINE = Distributed(cluster, database, table, sharding_key)
```
- **Виртуальная таблица** для работы с кластером  
- **Прозрачное распределение** запросов по шардам  

---

## **3. Операции с данными**  

### **a) Вставка данных**  
```sql
INSERT INTO analytics.events VALUES
('2024-01-15 10:00:00', 123, 'click', 1.5),
('2024-01-15 10:00:01', 456, 'view', 2.0);
```

### **b) Выборка с агрегацией**  
```sql
SELECT 
    toStartOfHour(timestamp) as hour,
    count(*) as events_count,
    avg(value) as avg_value
FROM analytics.events 
WHERE timestamp >= now() - INTERVAL 1 DAY
GROUP BY hour
ORDER BY hour;
```

### **c) Модификация данных**  
```sql
-- Осторожно! Мутации тяжелые
ALTER TABLE analytics.events 
DELETE WHERE timestamp < '2024-01-01';
```

---

## **4. Паттерны использования**  

### **a) Веб-аналитика**  
```sql
-- Анализ пользовательской активности
SELECT
    user_id,
    countIf(event_type = 'purchase') as purchases,
    sumIf(value, event_type = 'purchase') as revenue
FROM analytics.events
GROUP BY user_id
HAVING purchases > 5;
```

### **b) Мониторинг и логи**  
```sql
-- Анализ ошибок по минутам
SELECT
    toStartOfMinute(timestamp) as minute,
    count(*) as errors
FROM server_logs
WHERE level = 'ERROR'
GROUP BY minute
ORDER BY minute DESC
LIMIT 10;
```

### **c) Финансовая отчетность**  
```sql
-- Ежедневная статистика
SELECT
    toDate(timestamp) as date,
    sum(amount) as daily_total,
    countDistinct(user_id) as unique_payers
FROM financial_transactions
GROUP BY date
ORDER BY date DESC;
```

---

## **5. Внутреннее устройство**  
ClickHouse оптимизирован для **аналитических рабочих нагрузок**:

### **Колоночное хранение**  
- **Эффективное сжатие**: Одинаковые типы данных в колонке  
- **Чтение только нужных колонок**: Минимизация I/O операций  
- **Векторное выполнение**: Обработка данных порциями  

### **Индексы**  
- **Первичный ключ**: Разреженный индекс для быстрого поиска  
- **Партиционирование**: Автоматическое разделение данных  
- **Сканирование**: Полное сканирование быстрее, чем точечные запросы  

---

## **6. Отличия от PostgreSQL**  

### **a) Модель хранения**  
| ClickHouse | PostgreSQL |
|------------|------------|
| Колоночная | Строчная |
| Оптимизирована для чтения | Сбалансированная |

### **b) Производительность**  
```sql
-- ClickHouse: ~0.1 сек на 1 млрд записей
SELECT count(*) FROM large_table WHERE date = today();

-- PostgreSQL: ~10+ сек на 1 млрд записей  
SELECT count(*) FROM large_table WHERE date = NOW()::date;
```

### **c) Транзакции**  
- **ClickHouse**: Ограниченная поддержка, нет полноценного ACID  
- **PostgreSQL**: Полная поддержка транзакций  

### **d) JOIN операции**  
- **ClickHouse**: Избегайте сложных JOIN, предпочитайте денормализацию  
- **PostgreSQL**: Эффективные JOIN с оптимизатором запросов  

---

## **7. Опасные ситуации**  

### **a) Частые вставки маленькими порциями**  
```sql
-- ПЛОХО: Много мелких вставок
INSERT INTO events VALUES (now(), 1, 'click');
INSERT INTO events VALUES (now(), 2, 'view');

-- ХОРОШО: Пакетная вставка  
INSERT INTO events VALUES 
(now(), 1, 'click'), (now(), 2, 'view'), (now(), 3, 'purchase');
```

### **b) Слишком много партиций**  
```sql
-- ПЛОХО: Партиционирование по дням на года данных
PARTITION BY toYYYYMMDD(timestamp)

-- ХОРОШО: Партиционирование по месяцам  
PARTITION BY toYYYYMM(timestamp)
```

### **c) Отсутствие ORDER BY**  
```sql
-- ПЛОХО: Без порядка в MergeTree
ENGINE = MergeTree()

-- ХОРОШО: С явным порядком
ENGINE = MergeTree()
ORDER BY (timestamp, user_id, event_type)
```

---

**Дополнительно**:  
- [Официальная документация](https://clickhouse.com/docs/)  
- [ClickHouse для начинающих](https://clickhouse.com/docs/ru/getting-started/tutorial)  

---

### **Когда использовать?**  
✔️ Аналитика и отчетность (OLAP)  
✔️ Обработка больших объемов данных  
✔️ Агрегации и оконные функции  
✔️ Хранение временных рядов  

❌ Транзакционные системы (OLTP)  
❌ Частые обновления данных  
❌ Сложные JOIN между большими таблицами  
❌ Системы с требованием ACID