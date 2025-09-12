```
Окружение: Oracle Linux 8.10, PostgreSQL 17.
```
# Работа с join'ами, статистикой.

Подготавливаю тестовые таблицы и наполняю их данными для отработки типов соединений:
```sql
-- Таблица "departments"
CREATE TABLE departments (
    dept_id SERIAL PRIMARY KEY,
    dept_name VARCHAR(50) NOT NULL
);

-- Таблица "employees"
CREATE TABLE employees (
    emp_id SERIAL PRIMARY KEY,
    emp_name VARCHAR(50) NOT NULL,
    dept_id INT,
    salary INT
);

-- Вставляю данные в "departments"
INSERT INTO departments (dept_name) VALUES
('Разработка'),
('Маркетинг'),
('Продажи'),
('Поддержка'); -- Этот отдел пока без сотрудников

-- Вставляю данные в "employees"
INSERT INTO employees (emp_name, dept_id, salary) VALUES
('Иван Иванов', 1, 100000),
('Петр Петров', 1, 95000),
('Мария Сидорова', 2, 80000),
('Анна Козлова', 3, 75000),
('Сергей Сергеев', NULL, 60000); -- Этот сотрудник пока не прикреплен к отделу
```
Таблица `departments` содержит информацию об отделах, а `employees` - о сотрудниках. Поле `dept_id` в `employees` является внешним ключом, связывающим ее с `departments`. 

## 1. Реализовать прямое соединение (INNER JOIN) двух или более таблиц.
```sql
SELECT e.emp_name, d.dept_name
FROM employees e
INNER JOIN departments d ON e.dept_id = d.dept_id;
```
`INNER JOIN` возвращает только те строки из обеих таблиц, для которых условие соединения `(e.dept_id = d.dept_id)` выполняется. Мы видим только сотрудников, которые прикреплены к существующим отделам.
```sql
    emp_name    | dept_name
----------------+------------
 Иван Иванов    | Разработка
 Петр Петров    | Разработка
 Мария Сидорова | Маркетинг
 Анна Козлова   | Продажи
(4 строки)
```
## 2. Реализовать левостороннее (LEFT OUTER JOIN) соединение двух или более таблиц.
```sql
SELECT e.emp_name, d.dept_name
FROM employees e
LEFT OUTER JOIN departments d ON e.dept_id = d.dept_id;
```
`LEFT JOIN` возвращает все строки из левой таблицы `employees`, даже если для них нет совпадений в правой таблице `departments`. Для сотрудников без отдела в столбцах из `departments` будут значения `NULL`. Это полезно, когда нужно увидеть все записи из основной таблицы, даже если связанные данные отсутствуют.
```sql
    emp_name    | dept_name
----------------+------------
 Иван Иванов    | Разработка
 Петр Петров    | Разработка
 Мария Сидорова | Маркетинг
 Анна Козлова   | Продажи
 Сергей Сергеев | (NULL)
(5 строк)
```
## 3. Реализовать кросс-соединение (CROSS JOIN) двух или более таблиц.
```sql
SELECT e.emp_name, d.dept_name
FROM employees e
CROSS JOIN departments d;
```
`CROSS JOIN` возвращает декартово произведение двух таблиц: каждая строка из `employees` соединяется с каждой строкой из `departments`. Количество строк в результате равно произведению количества строк в обеих таблицах (5 сотрудников на 4 отдела = 20 строк). Используется редко, но может быть полезен для генерации всевозможных комбинаций.
```sql
    emp_name    | dept_name
----------------+------------
 Иван Иванов    | Разработка
 Петр Петров    | Разработка
 Мария Сидорова | Разработка
 Анна Козлова   | Разработка
 Сергей Сергеев | Разработка
 Иван Иванов    | Маркетинг
 Петр Петров    | Маркетинг
 Мария Сидорова | Маркетинг
 Анна Козлова   | Маркетинг
 Сергей Сергеев | Маркетинг
 Иван Иванов    | Продажи
 Петр Петров    | Продажи
 Мария Сидорова | Продажи
 Анна Козлова   | Продажи
 Сергей Сергеев | Продажи
 Иван Иванов    | Поддержка
 Петр Петров    | Поддержка
 Мария Сидорова | Поддержка
 Анна Козлова   | Поддержка
 Сергей Сергеев | Поддержка
(20 строк)
```
## 4. Реализовать полное соединение (FULL OUTER JOIN) двух или более таблиц.
```sql
SELECT e.emp_name, d.dept_name
FROM employees e
FULL OUTER JOIN departments d ON e.dept_id = d.dept_id;
```
`FULL JOIN` возвращает все строки из обеих таблиц. Если для сотрудника нет отдела, поля отдела будут `NULL`. Если для отдела нет сотрудников, поля сотрудника будут `NULL`. Это соединение полезно для полного аудита связей.
```sql
    emp_name    | dept_name
----------------+------------
 Иван Иванов    | Разработка
 Петр Петров    | Разработка
 Мария Сидорова | Маркетинг
 Анна Козлова   | Продажи
 Сергей Сергеев | (NULL)
 (NULL)         | Поддержка
(6 строк)
```
## 5. Комбинирование разных типов соединений.
Создам дополнительную таблицу `projects` с проектами. 
```sql
CREATE TABLE projects (
    project_id SERIAL PRIMARY KEY,
    project_name VARCHAR(50),
    emp_id INT
);

INSERT INTO projects (project_name, emp_id) VALUES
('Проект А', 1),
('Проект Б', 2),
('Проект В', 9);
```
#### 5.1 INNER JOIN + LEFT JOIN. Найду всех сотрудников, у которых есть отдел, и покажу их проекты (даже если проектов нет):
```sql
SELECT e.emp_name, d.dept_name, p.project_name
FROM employees e
INNER JOIN departments d ON e.dept_id = d.dept_id    -- Только сотрудники с отделом
LEFT JOIN projects p ON e.emp_id = p.emp_id;         -- Но все их проекты
```
Сотрудников без отделов (как Сергеев) не будет, но у тех, кто есть, будут все их проекты, а если проектов нет - NULL:
```sql
    emp_name    | dept_name  | project_name
----------------+------------+--------------
 Иван Иванов    | Разработка | Проект А
 Петр Петров    | Разработка | Проект Б
 Анна Козлова   | Продажи    | (NULL)
 Мария Сидорова | Маркетинг  | (NULL)
(4 строки)
```
#### 5.2 FULL JOIN + LEFT JOIN. Я хочу получить полную картину по сотрудникам и отделам, а для тех, у кого есть отдел, показать проекты:
```sql
SELECT e.emp_name, d.dept_name, p.project_name
FROM employees e
FULL JOIN departments d ON e.dept_id = d.dept_id     -- Все сотрудники и все отделы
LEFT JOIN projects p ON e.emp_id = p.emp_id          -- Все сотрудники и все отделы
```
Получаю всех сотрудников (даже без отдела), все отделы (даже без сотрудников), и проекты только для тех записей, где был реальный сотрудник (из левой части FULL JOIN):
```sql
    emp_name    | dept_name  | project_name
----------------+------------+--------------
 Иван Иванов    | Разработка | Проект А
 Петр Петров    | Разработка | Проект Б
 (NULL)         | Поддержка  | (NULL)
 Сергей Сергеев | (NULL)     | (NULL)
 Анна Козлова   | Продажи    | (NULL)
 Мария Сидорова | Маркетинг  | (NULL)
(6 строк)
```
#### 5.3 CROSS JOIN + LEFT JOIN. Я хочу сгенерировать матрицу "все отделы vs все проекты" и показать, какой сотрудник мог бы быть их связующим звеном, если бы он существовал.
```sql
SELECT d.dept_name, p.project_name, e.emp_name
FROM departments d
CROSS JOIN projects p                               -- Все комбинации отделов и проектов                 
LEFT JOIN employees e ON d.dept_id = e.dept_id      -- Попытка найти сотрудника в отделе
AND e.emp_id = p.emp_id;                            -- Который работает именно над этим проектом
```
Я получаю все возможные пары "отдел-проект". Для большинства пар в `emp_name` будет `NULL`, так как такой связки не существует. Но если в отделе есть сотрудник, работающий над проектом, его имя будет указано:
```sql
 dept_name  | project_name |  emp_name
------------+--------------+-------------
 Разработка | Проект А     | Иван Иванов
 Разработка | Проект Б     | Петр Петров
 Разработка | Проект В     | (NULL)
 Маркетинг  | Проект А     | (NULL)
 Маркетинг  | Проект Б     | (NULL)
 Маркетинг  | Проект В     | (NULL)
 Продажи    | Проект А     | (NULL)
 Продажи    | Проект Б     | (NULL)
 Продажи    | Проект В     | (NULL)
 Поддержка  | Проект А     | (NULL)
 Поддержка  | Проект Б     | (NULL)
 Поддержка  | Проект В     | (NULL)
(12 строк)
```

### Задача со звёздочкой. 
Сначала подготовлю кластер. В `postgresql.conf` изменяю:
```sql
shared_preload_libraries = 'pg_stat_statements'
track_io_timing = on  # Включить отслеживание времени I/O
```
Перезагружаю сервер. Создаю расширение в базе данных:
```sql
postgres=# CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
CREATE EXTENSION
```
#### Метрика 1. 15 самых ресурсоемких JOIN-запросов в системе, анализ их производительность, потенциальные проблемы, предположительный характер соединений таблиц.
```sql
SELECT 
    queryid as "ID",
    calls as "Вызовы",
    round(total_exec_time::numeric, 2) as "Общее время мс",
    round(mean_exec_time::numeric, 2) as "Среднее время мс",
    rows as "Строк",
    CASE 
        WHEN mean_exec_time > 100 THEN '🔴 ' || round(mean_exec_time::numeric, 2)
        WHEN mean_exec_time > 10 THEN '🟡 ' || round(mean_exec_time::numeric, 2)
        WHEN mean_exec_time > 1 THEN '🟢 ' || round(mean_exec_time::numeric, 2)
        ELSE '✅ ' || round(mean_exec_time::numeric, 2)
    END as "Время",
    CASE 
        WHEN mean_exec_time > 10 AND calls > 5 THEN '🔴 Медленно + часто'
        WHEN rows > 1000 AND mean_exec_time > 5 THEN '🟡 Много данных + медленно'
        WHEN calls > 100 AND mean_exec_time > 1 THEN '🟡 Часто + медленно'
        ELSE '✅ Норма'
    END as "Проблема",
    CASE 
        WHEN query LIKE '%JOIN%JOIN%JOIN%' THEN '🔴 Много JOIN'
        WHEN query LIKE '%WHERE%.% = %.%' THEN '✅ Индексный JOIN'
        WHEN query LIKE '%WHERE%IS NOT NULL%' THEN '🟡 Фильтр перед JOIN'
        ELSE '⚪ Обычный JOIN'
    END as "Тип JOIN"
FROM pg_stat_statements 
WHERE query LIKE '%JOIN%' 
AND query NOT LIKE '%pg_stat_%'
AND mean_exec_time > 1
ORDER BY (total_exec_time * calls) DESC
LIMIT 15;
```
Пример вывода:
```sql
          ID          | Вызовы | Общее время мс | Среднее время мс | Строк |   Время   | Проблема |     Тип JOIN
----------------------+--------+----------------+------------------+-------+-----------+----------+-------------------
 -8917144814669508499 |      1 |         220.55 |           220.55 |   499 | 🔴 220.55 | ✅ Норма | ⚪ Обычный JOIN
 -4269329529793253551 |      1 |          18.95 |            18.95 |   300 | 🟡 18.95  | ✅ Норма | 🔴 Много JOIN
 -2378463660404591078 |      1 |          13.03 |            13.03 |   500 | 🟡 13.03  | ✅ Норма | ⚪ Обычный JOIN
  1834376733575628938 |      1 |          12.42 |            12.42 |   500 | 🟡 12.42  | ✅ Норма | ⚪ Обычный JOIN
 -2428091786180286149 |      1 |           2.40 |             2.40 |   500 | 🟢 2.40   | ✅ Норма | ✅ Индексный JOIN
  7398335915666730765 |      1 |           1.21 |             1.21 |   200 | 🟢 1.21   | ✅ Норма | ✅ Индексный JOIN

```

#### Метрика 2. Анализ эффективности индексов (которые занимают место, но никогда или очень редко используются).
```sql
SELECT
    relname as "Таблица",
    indexrelname as "Индекс",
    pg_size_pretty(pg_relation_size(indexrelid)) as "Размер",
    idx_scan as "Сканирований",
    CASE
        WHEN idx_scan = 0 THEN '❌ Никогда'
        WHEN idx_scan < 10 THEN '🟡 Редко'
        WHEN idx_scan < 100 THEN '🟢 Иногда'
        ELSE '✅ Часто'
    END as "Использование",
    CASE
        WHEN idx_scan = 0 AND pg_relation_size(indexrelid) > 1048576 THEN '🗑️ Удалить'
        WHEN idx_scan < 100 AND pg_relation_size(indexrelid) > 1048576 THEN '👀 Проверить'
        ELSE '✅ OK'
    END as "Рекомендация"
FROM
    pg_stat_all_indexes
WHERE
    schemaname NOT IN ('pg_catalog', 'pg_toast', 'information_schema')
ORDER BY
    pg_relation_size(indexrelid) DESC,
    idx_scan ASC
LIMIT 15;
```
Пример вывода:
```sql
     Таблица    |           Индекс           | Размер  | Сканирований | Использование | Рекомендация
---------------+----------------------------+---------+--------------+---------------+--------------
 products      | products_pkey              | 21 MB   |          885 | ✅ Часто      | ✅ OK
 products      | idx_products_category      | 6848 kB |            0 | ❌ Никогда    | 🗑️ Удалить
 products      | idx_products_supplier_id   | 6816 kB |            0 | ❌ Никогда    | 🗑️ Удалить
 deadlock_test | deadlock_test_pkey         | 6600 kB |            0 | ❌ Никогда    | 🗑️ Удалить
 clients       | clients_pkey               | 5496 kB |          152 | ✅ Часто      | ✅ OK
 clients       | idx_clients_status         | 1728 kB |            0 | ❌ Никогда    | 🗑️ Удалить
 order_items   | order_items_pkey           | 672 kB  |            0 | ❌ Никогда    | ✅ OK
 order_items   | idx_order_items_order_id   | 448 kB  |            6 | 🟡 Редко      | ✅ OK
 order_items   | idx_order_items_product_id | 240 kB  |            2 | 🟡 Редко      | ✅ OK
 orders        | orders_pkey                | 240 kB  |            6 | 🟡 Редко      | ✅ OK
 orders        | idx_orders_date            | 96 kB   |            1 | 🟡 Редко      | ✅ OK
 suppliers     | suppliers_pkey             | 72 kB   |           51 | 🟢 Иногда     | ✅ OK
 projects      | idx_projects_emp_id        | 16 kB   |            0 | ❌ Никогда    | ✅ OK
 test_locks    | test_locks_pkey            | 16 kB   |            0 | ❌ Никогда    | ✅ OK
 departments   | departments_pkey           | 16 kB   |            0 | ❌ Никогда    | ✅ OK
```
### Метрика 3. Отслеживание активности и эффективности процессов очистки (VACUUM и AUTOVACUUM).
```sql
SELECT
    relname AS "Таблица",
    n_live_tup AS "Живых",
    n_dead_tup AS "Мёртвых",
    pg_size_pretty(pg_table_size(relid)) AS "Размер",
    CASE 
        WHEN last_autovacuum IS NULL THEN 'Никогда'
        ELSE (now() - last_autovacuum::timestamp)::text
    END AS "Последний autovacuum",
    CASE
        WHEN n_dead_tup = 0 THEN '✅'
        WHEN n_dead_tup > 0 AND n_dead_tup <= 100 THEN '⚠️ '
        WHEN n_dead_tup > 100 THEN '❌'
        ELSE '❓'
    END AS "С"
FROM
    pg_stat_user_tables
WHERE
    n_dead_tup > 0 OR last_autovacuum IS NOT NULL
ORDER BY
    n_dead_tup DESC
LIMIT 10;
```
Пример вывода: 
```sql
   Таблица   |  Живых  | Мёртвых | Размер  | Последний autovacuum | С
-------------+---------+---------+---------+----------------------+----
 orders      |   10000 |       0 | 688 kB  | 03:37:14.569708      | ✅
 suppliers   |    2500 |       0 | 176 kB  | 03:38:13.86726       | ✅
 order_items |   30000 |       0 | 1800 kB | 03:37:13.060113      | ✅
 clients     |  250000 |       0 | 16 MB   | 03:38:14.465631      | ✅
 products    | 1000000 |       0 | 68 MB   | 03:37:14.026433      | ✅
```
