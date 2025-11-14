---
tags:
  - Программирование
  - SQL
  - Продвинутые_темы
  - Оконные_функции
---
# **Оконные функции PostgreSQL**  
*Аналитические запросы, ранжирование, скользящие агрегаты*  

---

## **Введение в оконные функции**

Оконные функции (Window Functions) — это мощный инструмент PostgreSQL для выполнения вычислений над набором строк, связанных с текущей строкой. В отличие от агрегатных функций, которые сворачивают множество строк в одну, оконные функции сохраняют все исходные строки, добавляя к ним вычисленные значения.

### **Основные концепции**
- **Окно (Window)** — набор строк, над которым выполняются вычисления
- **PARTITION BY** — разделение данных на группы для независимых вычислений
- **ORDER BY** — определение порядка строк внутри окна
- **FRAME** — определение подмножества строк внутри окна для вычислений

---

## **Синтаксис OVER() и PARTITION BY**

### **Базовый синтаксис оконных функций**

```sql
function_name([arguments]) OVER (
    [PARTITION BY partition_expression]
    [ORDER BY sort_expression [ASC | DESC]]
    [frame_clause]
)
```

### **PARTITION BY — разделение на группы**

**Назначение:** Разбивает данные на независимые группы (партиции), внутри которых выполняются вычисления.

**Пример 1: Базовая партиция**
```sql
SELECT 
    department,
    employee_name,
    salary,
    AVG(salary) OVER (PARTITION BY department) as avg_department_salary
FROM employees;
```

**Пример 2: Множественные партиции**
```sql
SELECT 
    department,
    job_title,
    employee_name,
    salary,
    AVG(salary) OVER (PARTITION BY department, job_title) as avg_position_salary
FROM employees;
```

**Пример 3: Сравнение с агрегатными функциями**
```sql
-- Агрегатная функция (группирует строки)
SELECT department, AVG(salary) 
FROM employees 
GROUP BY department;

-- Оконная функция (сохраняет все строки)
SELECT department, employee_name, salary,
       AVG(salary) OVER (PARTITION BY department) as avg_salary
FROM employees;
```

---

## **Ранжирующие функции (ROW_NUMBER, RANK, DENSE_RANK)**

### **ROW_NUMBER — последовательная нумерация**

**Назначение:** Присваивает уникальный последовательный номер каждой строке в рамках партиции.

**Пример: Рейтинг сотрудников по зарплате**
```sql
SELECT 
    employee_name,
    department,
    salary,
    ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) as salary_rank
FROM employees;
```

### **RANK — ранжирование с пропусками**

**Назначение:** Присваивает ранг с пропусками при одинаковых значениях.

**Пример: Ранжирование с учетом одинаковых зарплат**
```sql
SELECT 
    employee_name,
    salary,
    RANK() OVER (ORDER BY salary DESC) as rank_with_gaps
FROM employees;

-- Результат для одинаковых зарплат:
-- salary: 5000, 5000, 4000 → rank: 1, 1, 3
```

### **DENSE_RANK — ранжирование без пропусков**

**Назначение:** Присваивает ранг без пропусков при одинаковых значениях.

**Пример: Плотное ранжирование**
```sql
SELECT 
    employee_name,
    salary,
    DENSE_RANK() OVER (ORDER BY salary DESC) as dense_rank
FROM employees;

-- Результат для одинаковых зарплат:
-- salary: 5000, 5000, 4000 → rank: 1, 1, 2
```

### **Сравнение ранжирующих функций**
```sql
SELECT 
    employee_name,
    salary,
    ROW_NUMBER() OVER (ORDER BY salary DESC) as row_num,
    RANK() OVER (ORDER BY salary DESC) as rank,
    DENSE_RANK() OVER (ORDER BY salary DESC) as dense_rank
FROM employees;
```

---

## **Агрегатные функции с окнами (SUM, AVG, COUNT)**

### **Накопительные агрегаты**

**Пример 1: Накопительная сумма зарплат**
```sql
SELECT 
    employee_name,
    hire_date,
    salary,
    SUM(salary) OVER (ORDER BY hire_date) as cumulative_salary
FROM employees;
```

**Пример 2: Среднее по партиции с накоплением**
```sql
SELECT 
    department,
    employee_name,
    salary,
    AVG(salary) OVER (PARTITION BY department) as avg_department,
    AVG(salary) OVER (PARTITION BY department ORDER BY hire_date) as running_avg
FROM employees;
```

### **Скользящие агрегаты с фреймами**

**Пример: Скользящее среднее за 3 месяца**
```sql
SELECT 
    month,
    revenue,
    AVG(revenue) OVER (
        ORDER BY month 
        ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING
    ) as moving_avg_3month
FROM monthly_sales;
```

### **Процент от общего**

**Пример: Процент зарплаты от общей по отделу**
```sql
SELECT 
    department,
    employee_name,
    salary,
    salary * 100.0 / SUM(salary) OVER (PARTITION BY department) as salary_percent
FROM employees;
```

---

## **Функции смещения (LAG, LEAD, FIRST_VALUE)**

### **LAG — доступ к предыдущей строке**

**Назначение:** Возвращает значение из предыдущей строки.

**Пример: Сравнение с предыдущим месяцем**
```sql
SELECT 
    month,
    revenue,
    LAG(revenue) OVER (ORDER BY month) as prev_month_revenue,
    revenue - LAG(revenue) OVER (ORDER BY month) as revenue_growth
FROM monthly_sales;
```

**Пример с указанием смещения и значения по умолчанию**
```sql
SELECT 
    month,
    revenue,
    LAG(revenue, 2, 0) OVER (ORDER BY month) as revenue_2_months_ago
FROM monthly_sales;
```

### **LEAD — доступ к следующей строке**

**Назначение:** Возвращает значение из следующей строки.

**Пример: Прогнозирование трендов**
```sql
SELECT 
    month,
    revenue,
    LEAD(revenue) OVER (ORDER BY month) as next_month_revenue,
    LEAD(revenue, 3) OVER (ORDER BY month) as revenue_in_3_months
FROM monthly_sales;
```

### **FIRST_VALUE и LAST_VALUE**

**Назначение:** Возвращает первое/последнее значение в окне.

**Пример: Сравнение с лучшим результатом**
```sql
SELECT 
    department,
    employee_name,
    salary,
    FIRST_VALUE(salary) OVER (
        PARTITION BY department 
        ORDER BY salary DESC
    ) as top_salary_in_dept,
    salary * 100.0 / FIRST_VALUE(salary) OVER (
        PARTITION BY department 
        ORDER BY salary DESC
    ) as percent_of_top
FROM employees;
```

### **NTH_VALUE — доступ к произвольной строке**

**Пример: Сравнение с медианной зарплатой**
```sql
SELECT 
    employee_name,
    salary,
    NTH_VALUE(salary, 3) OVER (
        PARTITION BY department 
        ORDER BY salary
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) as median_salary
FROM employees;
```

---

## **Практические кейсы аналитики**

### **Кейс 1: Анализ продаж по категориям**

```sql
-- Сравнение продаж товаров внутри категорий
SELECT 
    product_name,
    category,
    sales_amount,
    RANK() OVER (PARTITION BY category ORDER BY sales_amount DESC) as rank_in_category,
    sales_amount * 100.0 / SUM(sales_amount) OVER (PARTITION BY category) as category_percent,
    LAG(sales_amount) OVER (PARTITION BY category ORDER BY sales_amount DESC) as prev_product_sales
FROM products
ORDER BY category, rank_in_category;
```

### **Кейс 2: Анализ временных рядов**

```sql
-- Анализ ежемесячных продаж с скользящими показателями
SELECT 
    year,
    month,
    revenue,
    LAG(revenue) OVER (ORDER BY year, month) as prev_month,
    revenue - LAG(revenue) OVER (ORDER BY year, month) as month_growth,
    AVG(revenue) OVER (
        ORDER BY year, month 
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) as moving_avg_3month,
    SUM(revenue) OVER (
        PARTITION BY year 
        ORDER BY month
    ) as ytd_revenue
FROM monthly_sales
ORDER BY year, month;
```

### **Кейс 3: Customer Analytics**

```sql
-- Анализ клиентского поведения
SELECT 
    customer_id,
    order_date,
    order_amount,
    SUM(order_amount) OVER (
        PARTITION BY customer_id 
        ORDER BY order_date
    ) as cumulative_spent,
    COUNT(*) OVER (
        PARTITION BY customer_id 
        ORDER BY order_date
    ) as order_count,
    AVG(order_amount) OVER (
        PARTITION BY customer_id
    ) as avg_order_value,
    FIRST_VALUE(order_date) OVER (
        PARTITION BY customer_id 
        ORDER BY order_date
    ) as first_order_date
FROM orders
ORDER BY customer_id, order_date;
```

### **Кейс 4: Employee Performance Analysis**

```sql
-- Анализ производительности сотрудников
SELECT 
    department,
    employee_name,
    salary,
    performance_score,
    RANK() OVER (PARTITION BY department ORDER BY performance_score DESC) as performance_rank,
    salary - AVG(salary) OVER (PARTITION BY department) as salary_vs_avg,
    PERCENT_RANK() OVER (PARTITION BY department ORDER BY performance_score) as performance_percentile,
    CUME_DIST() OVER (PARTITION BY department ORDER BY salary) as salary_distribution
FROM employees
WHERE department IS NOT NULL;
```

### **Кейс 5: Advanced Frame Specifications**

```sql
-- Различные типы фреймов для агрегации
SELECT 
    date,
    revenue,
    -- Текущая строка и все предыдущие
    SUM(revenue) OVER (ORDER BY date ROWS UNBOUNDED PRECEDING) as cumulative,
    
    -- Текущая строка и 2 предыдущие
    AVG(revenue) OVER (ORDER BY date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) as avg_3day,
    
    -- 1 предыдущая, текущая и 1 следующая
    SUM(revenue) OVER (ORDER BY date ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING) as sum_3day_centered,
    
    -- Все строки в партиции
    FIRST_VALUE(revenue) OVER (ORDER BY date ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) as first_value
FROM daily_sales;
```

---

## **Оптимизация производительности**

### **Индексы для оконных функций**

```sql
-- Создание индексов для оптимизации оконных функций
CREATE INDEX idx_employees_dept_salary ON employees(department, salary DESC);
CREATE INDEX idx_sales_date ON monthly_sales(month);
CREATE INDEX idx_orders_customer_date ON orders(customer_id, order_date);
```

### **Материализация CTE с оконными функциями**

```sql
-- Для сложных вычислений с оконными функциями
WITH ranked_employees AS (
    SELECT 
        employee_name,
        department,
        salary,
        RANK() OVER (PARTITION BY department ORDER BY salary DESC) as rank
    FROM employees
    WHERE department IS NOT NULL
)
SELECT *
FROM ranked_employees 
WHERE rank <= 3;  -- Топ-3 по зарплате в каждом отделе
```

### **Ограничения и лучшие практики**

1. **Избегайте излишне сложных окон** — разбивайте на несколько CTE
2. **Используйте WHERE для фильтрации до оконных функций** когда возможно
3. **Проверяйте план выполнения** для больших наборов данных
4. **Рассмотрите материализацию** часто используемых оконных вычислений

Оконные функции PostgreSQL предоставляют мощный инструментарий для сложной аналитики, позволяя выполнять вычисления, которые ранее требовали множественных подзапросов или обработки на стороне приложения.