---
tags:
  - Программирование
  - SQL
  - Основы_языка_SQL
  - Индексы
---
### **Индексы в SQL: полное руководство**

---

#### **1. Основы индексирования**

**Что такое индекс?**  
Структура данных, ускоряющая поиск записей по определенным столбцам. Работает как оглавление в книге.

**Принцип работы:**
1. Создается отдельная структура данных (B-дерево, хеш-таблица и др.)
2. При запросе СУБД сначала проверяет индекс
3. По индексу находится точное расположение данных

**Синтаксис создания:**
```sql
CREATE [UNIQUE] INDEX index_name 
ON table_name (column1 [ASC|DESC], ...)
[USING method]  -- тип индекса
[WITH (option = value)]; -- дополнительные опции
```

---

#### **2. Типы индексов**

##### **2.1. B-Tree (балансированное дерево)**
**Поддержка:** MySQL, PostgreSQL (по умолчанию)  
**Лучшее применение:** 
- Диапазонные запросы (`BETWEEN`, `>`, `<`)
- Сортировка (`ORDER BY`)
- Точные совпадения (`=`)

**Пример:**
```sql
CREATE INDEX idx_customer_name ON customers(last_name, first_name);
```

**Особенности:**
- Автоматически создается для PRIMARY KEY и UNIQUE
- Поддерживает составные индексы (до 32 столбцов в PostgreSQL)

##### **2.2. Hash индекс**
**Поддержка:** PostgreSQL, Memory-движки MySQL  
**Лучшее применение:** 
- Точные совпадения (`=`)
- Нет поддержки диапазонов

**Пример:**
```sql
CREATE INDEX idx_product_hash ON products USING HASH (product_code);
```

**Ограничения:**
- Нет сортировки
- Только простые условия равенства

##### **2.3. GIN (Generalized Inverted Index)**
**Поддержка:** PostgreSQL  
**Лучшее применение:**
- Составные данные (массивы, JSON, полнотекстовый поиск)
- Множественные значения в одном поле

**Пример:**
```sql
-- Для JSON
CREATE INDEX idx_profile_tags ON users USING GIN ((profile->'tags'));

-- Для массивов
CREATE INDEX idx_product_categories ON products USING GIN (categories);
```

##### **2.4. GiST (Generalized Search Tree)**
**Поддержка:** PostgreSQL  
**Лучшее применение:**
- Геометрические данные
- Полнотекстовый поиск
- Нечеткий поиск

**Пример:**
```sql
CREATE INDEX idx_geom ON polygons USING GiST (geom);
```

##### **2.5. FULLTEXT**
**Поддержка:** MySQL  
**Лучшее применение:** 
- Текстовый поиск по словам
- Релевантный поиск

**Пример:**
```sql
CREATE FULLTEXT INDEX idx_content ON articles(title, body);
```

**Поиск:**
```sql
SELECT * FROM articles 
WHERE MATCH(title, body) AGAINST('database optimization');
```

##### **2.6. SP-GiST (Space-Partitioned GiST)**
**Поддержка:** PostgreSQL  
**Лучшее применение:**
- Неравномерные структуры данных (IP-адреса, координаты)

##### **2.7. BRIN (Block Range INdex)**
**Поддержка:** PostgreSQL  
**Лучшее применение:**
- Очень большие таблицы с коррелированными данными
- Диапазонные запросы по временным меткам

**Пример:**
```sql
CREATE INDEX idx_logs_brin ON logs USING BRIN (created_at);
```

---

#### **3. Составные индексы**

**Принцип "левостороннего" правила:**
Индекс `(A, B, C)` может использоваться для:
- `WHERE A = 1 AND B = 2 AND C = 3`
- `WHERE A = 1 AND B = 2`
- `WHERE A = 1`
- `ORDER BY A, B, C`

**Пример оптимизации:**
```sql
-- Хороший составной индекс
CREATE INDEX idx_orders_date_status ON orders(order_date, status);

-- Использование:
SELECT * FROM orders 
WHERE order_date > '2023-01-01' AND status = 'shipped';
```

---

#### **4. Управление индексами**

**Просмотр индексов:**
```sql
-- MySQL
SHOW INDEXES FROM table_name;

-- PostgreSQL
SELECT * FROM pg_indexes WHERE tablename = 'table_name';
```

**Перестроение индекса:**
```sql
-- MySQL
ALTER TABLE table_name REBUILD INDEX index_name;

-- PostgreSQL
REINDEX INDEX index_name;
```

**Анализ использования:**
```sql
EXPLAIN ANALYZE SELECT * FROM table WHERE condition;
```

---

#### **5. Оптимизация индексов**

**Когда создавать:**
1. Столбцы в условиях WHERE/JOIN
2. Столбцы в ORDER BY
3. Столбцы с высокой кардинальностью

**Когда избегать:**
1. Часто изменяемые таблицы
2. Очень маленькие таблицы
3. Столбцы с низкой кардинальностью (пол, статус)

**Паттерны плохих индексов:**
1. Избыточные индексы (`(A,B)` и `(A)`)
2. Неиспользуемые индексы
3. Индексы по вычисляемым столбцам без persistant

---

#### **6. Сравнение СУБД**

| Характеристика      | MySQL            | PostgreSQL       |
|---------------------|------------------|------------------|
| Основной тип        | B-Tree           | B-Tree           |
| Хеш-индексы        | Только Memory    | Полная поддержка |
| Специальные индексы | FULLTEXT         | GIN, GiST, BRIN  |
| Частичные индексы  | Нет              | Да (`WHERE`)     |

---

#### **7. Практические примеры**

**Оптимальные индексы для интернет-магазина:**
```sql
-- Пользователи
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_phone ON users(phone) USING HASH;

-- Товары
CREATE INDEX idx_products_price ON products(price);
CREATE INDEX idx_products_category ON products(category_id);

-- Заказы
CREATE INDEX idx_orders_user_date ON orders(user_id, order_date DESC);
```

**Проблемный случай:**
```sql
-- Плохо: индекс не используется из-за функции
CREATE INDEX idx_lower_name ON users(lower(name));

-- Решение (PostgreSQL):
CREATE INDEX idx_lower_name ON users(lower(name));
-- Теперь запрос использует индекс:
SELECT * FROM users WHERE lower(name) = 'иван';
```

---

**См. также:**  
- [[Оптимизация запросов]]  
- [[Партиционирование таблиц]]  
- [[Анализ производительности]]  

**Дополнительные материалы:**  
- `EXPLAIN` и `EXPLAIN ANALYZE`  
- Индексные сканы vs Seq сканы  
- Covering indexes (индексы-покрытия)