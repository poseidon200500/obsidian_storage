---
tags:
  - Программирование
  - SQL
  - Основы_языка_SQL
---
### **Типы данных в SQL (MySQL и PostgreSQL)**

---

#### **1. Числовые типы**

**Целочисленные:**
- `TINYINT` (1 байт): -128 до 127 (0 до 255 UNSIGNED)
- `SMALLINT` (2 байта): -32,768 до 32,767
- `INT/INTEGER` (4 байта): -2^31 до 2^31-1
- `BIGINT` (8 байт): -2^63 до 2^63-1

**Пример:**
```sql
CREATE TABLE users (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    age TINYINT UNSIGNED
);
```

**Дробные числа:**
- `FLOAT` (4 байта): ~7 цифр точности
- `DOUBLE` (8 байт): ~15 цифр точности
- `DECIMAL(p,s)` (точные): p - общее число цифр, s - после запятой

**Пример денежных значений:**
```sql
CREATE TABLE products (
    price DECIMAL(10,2) -- 99999999.99
);
```

**Особенности:**
- В PostgreSQL: `NUMERIC` = `DECIMAL`
- В MySQL: `DECIMAL` обычно предпочтительнее `FLOAT`

---

#### **2. Строковые типы**

**Фиксированная длина:**
- `CHAR(n)`: Всегда занимает n символов (дополняется пробелами)

**Переменная длина:**
- `VARCHAR(n)`: До n символов (MySQL до 65,535, PostgreSQL до 1GB)
- `TEXT`: Длинный текст (MySQL: TINYTEXT, TEXT, MEDIUMTEXT, LONGTEXT)

**Пример:**
```sql
CREATE TABLE articles (
    title VARCHAR(100),
    content TEXT,
    short_code CHAR(5)
);
```

**Бинарные данные:**
- `BINARY`/`VARBINARY` (аналоги CHAR/VARCHAR для байтов)
- `BLOB` (для больших бинарных объектов)

**Кодировки:**
- MySQL: `utf8mb4` (настоящий UTF-8)
- PostgreSQL: автоматически UTF-8

---

#### **3. Дата и время**

**Основные типы:**
- `DATE`: Только дата (1000-01-01 до 9999-12-31)
- `TIME`: Только время (-838:59:59 до 838:59:59)
- `DATETIME`: Дата + время (MySQL: 1000-01-01 00:00:00 до 9999-12-31 23:59:59)
- `TIMESTAMP`: Дата + время с временной зоной (1970-2038, 4 байта)

**Пример:**
```sql
CREATE TABLE events (
    event_date DATE,
    start_time TIME,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Особенности:**
- В PostgreSQL: `TIMESTAMP WITH TIME ZONE` (рекомендуется)
- В MySQL: `TIMESTAMP` конвертируется в UTC при хранении

---

#### **4. JSON/XML**

**JSON:**
- MySQL (5.7+): `JSON` тип с валидацией
- PostgreSQL: `JSON` (текст) и `JSONB` (бинарный, быстрее)

**Пример:**
```sql
-- MySQL
CREATE TABLE users (
    settings JSON
);
INSERT INTO users VALUES ('{"theme":"dark","notifications":true}');

-- PostgreSQL
CREATE TABLE products (
    attributes JSONB
);
```

**XML:**
- MySQL: `XML` функции, но хранение как TEXT
- PostgreSQL: `XML` тип с валидацией

**Операции:**
```sql
-- Доступ к JSON (MySQL)
SELECT settings->>"$.theme" FROM users;

-- Доступ к JSONB (PostgreSQL)
SELECT attributes->>'color' FROM products;
```

---

#### **5. Специальные типы**

**PostgreSQL:**
- `UUID`: Уникальные идентификаторы
- `ARRAY`: Многомерные массивы
- `HSTORE`: Ключ-значение
- `GEOMETRY`: Геопространственные данные

**MySQL:**
- `ENUM`: Перечисление ('small','medium','large')
- `SET`: Множество значений ('read','write','execute')

---

#### **6. Выбор типа данных**

**Рекомендации:**
1. Для денег → `DECIMAL/NUMERIC`
2. Для дат → `DATE` (если не нужно время)
3. Для временных меток → `TIMESTAMP` (MySQL), `TIMESTAMPTZ` (PG)
4. Для текста → `VARCHAR(n)` (если известна макс. длина), иначе `TEXT`
5. Для настроек → `JSON` (MySQL), `JSONB` (PostgreSQL)

---

#### **7. Примеры проблем**

**1. Переполнение:**
```sql
-- Ошибка при вставке слишком большого числа
INSERT INTO tiny_table VALUES (300); -- Для TINYINT(255)
```

**2. Потеря точности:**
```sql
-- FLOAT vs DECIMAL
INSERT INTO numbers VALUES (1234567.89); -- Округление в FLOAT
```

**3. Неожиданное поведение дат:**
```sql
-- MySQL: невалидная дата → 0000-00-00
INSERT INTO dates VALUES ('2023-02-30');
```

**См. также:**  
- [[Оптимизация хранения данных]]  
- [[Индексы по сложным типам данных]]