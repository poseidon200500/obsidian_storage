---
tags:
  - Программирование
  - SQL
  - Основы_языка_SQL
---
### **Синтаксис SQL (Расширенная версия)**

---

#### **1. Команда SELECT**
**Определение:**
Команда `SELECT` используется для извлечения данных из одной или нескольких таблиц. Она позволяет выбирать конкретные столбцы, фильтровать данные, группировать их и сортировать.

**Полное описание:**
```sql
SELECT 
    [DISTINCT] column_list
    [FROM table_name]
    [WHERE conditions]
    [GROUP BY grouping_columns]
    [HAVING group_conditions]
    [ORDER BY sorting_columns [ASC|DESC]]
    [LIMIT count [OFFSET start]]
```

**Особенности:**
- `DISTINCT` удаляет дубликаты
- `AS` задает псевдонимы столбцам
- Можно выбирать данные без FROM (SELECT 1+1, NOW())
---

#### **2. Команда INSERT**
**Определение:**
Команда `INSERT` используется для добавления новых записей в таблицу. Она позволяет вставлять данные как из значений, так и из других таблиц или источников.

**Полные формы синтаксиса:**
```sql
-- Стандартная форма
INSERT INTO table_name (col1, col2) 
VALUES (val1, val2), (val3, val4);

-- Вставка из запроса
INSERT INTO table_name (col1, col2)
SELECT col1, col2 FROM source_table WHERE condition;

-- Вставка с DEFAULT значениями
INSERT INTO table_name DEFAULT VALUES;
```

---

#### **3. Команда UPDATE**
**Определение:**
Команда `UPDATE` используется для изменения существующих записей в таблице. Она позволяет обновлять значения в одном или нескольких столбцах на основе заданных условий.

**Полный синтаксис:**
```sql
UPDATE table_name
SET 
    column1 = value1,
    column2 = value2
[WHERE conditions]
[ORDER BY column]
[LIMIT count]
```

---

#### **4. Команда DELETE**
**Определение:**
Команда `DELETE` используется для удаления записей из таблицы. Она позволяет удалять данные на основе заданных условий и может работать с несколькими записями одновременно.

**Полный синтаксис:**
```sql
DELETE [LOW_PRIORITY] [QUICK] [IGNORE] 
FROM table_name
[WHERE conditions]
[ORDER BY column]
[LIMIT row_count]
```

--- 

Эти команды являются основными в SQL и позволяют выполнять широкий спектр операций с данными в реляционных базах данных.