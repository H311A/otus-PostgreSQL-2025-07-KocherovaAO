```
Окружение: Oracle Linux 8.10, PostgreSQL 17.
```
# Работа с join'ами, статистикой.

Подготавливаю тестовые таблицы и наполняю их данными для отработки типов соединений:
```
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
```
SELECT e.emp_name, d.dept_name
FROM employees e
INNER JOIN departments d ON e.dept_id = d.dept_id;
```
`INNER JOIN` возвращает только те строки из обеих таблиц, для которых условие соединения `(e.dept_id = d.dept_id)` выполняется. Мы видим только сотрудников, которые прикреплены к существующим отделам.
```
    emp_name    | dept_name
----------------+------------
 Иван Иванов    | Разработка
 Петр Петров    | Разработка
 Мария Сидорова | Маркетинг
 Анна Козлова   | Продажи
(4 строки)
```
## 2. Реализовать левостороннее (LEFT OUTER JOIN) соединение двух или более таблиц.
```
SELECT e.emp_name, d.dept_name
FROM employees e
LEFT OUTER JOIN departments d ON e.dept_id = d.dept_id;
```
`LEFT JOIN` возвращает все строки из левой таблицы `employees`, даже если для них нет совпадений в правой таблице `departments`. Для сотрудников без отдела в столбцах из `departments` будут значения `NULL`. Это полезно, когда нужно увидеть все записи из основной таблицы, даже если связанные данные отсутствуют.
```
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
```
SELECT e.emp_name, d.dept_name
FROM employees e
CROSS JOIN departments d;
```
`CROSS JOIN` возвращает декартово произведение двух таблиц: каждая строка из `employees` соединяется с каждой строкой из `departments`. Количество строк в результате равно произведению количества строк в обеих таблицах (5 сотрудников на 4 отдела = 20 строк). Используется редко, но может быть полезен для генерации всевозможных комбинаций.
```
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
```
SELECT e.emp_name, d.dept_name
FROM employees e
FULL OUTER JOIN departments d ON e.dept_id = d.dept_id;
```
`FULL JOIN` возвращает все строки из обеих таблиц. Если для сотрудника нет отдела, поля отдела будут `NULL`. Если для отдела нет сотрудников, поля сотрудника будут `NULL`. Это соединение полезно для полного аудита связей.
```
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
```
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
```
SELECT e.emp_name, d.dept_name, p.project_name
FROM employees e
INNER JOIN departments d ON e.dept_id = d.dept_id    -- Только сотрудники с отделом
LEFT JOIN projects p ON e.emp_id = p.emp_id;         -- Но все их проекты
```
Сотрудников без отделов (как Сергеев) не будет, но у тех, кто есть, будут все их проекты, а если проектов нет - NULL:
```
    emp_name    | dept_name  | project_name
----------------+------------+--------------
 Иван Иванов    | Разработка | Проект А
 Петр Петров    | Разработка | Проект Б
 Анна Козлова   | Продажи    | (NULL)
 Мария Сидорова | Маркетинг  | (NULL)
(4 строки)
```
#### 5.2 FULL JOIN + LEFT JOIN. Я хочу получить полную картину по сотрудникам и отделам, а для тех, у кого есть отдел, показать проекты:
```
SELECT e.emp_name, d.dept_name, p.project_name
FROM employees e
FULL JOIN departments d ON e.dept_id = d.dept_id     -- Все сотрудники и все отделы
LEFT JOIN projects p ON e.emp_id = p.emp_id          -- Все сотрудники и все отделы
```
Получаю всех сотрудников (даже без отдела), все отделы (даже без сотрудников), и проекты только для тех записей, где был реальный сотрудник (из левой части FULL JOIN):
```
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
```
SELECT d.dept_name, p.project_name, e.emp_name
FROM departments d
CROSS JOIN projects p                               -- Все комбинации отделов и проектов                 
LEFT JOIN employees e ON d.dept_id = e.dept_id      -- Попытка найти сотрудника в отделе
AND e.emp_id = p.emp_id;                            -- Который работает именно над этим проектом
```
Я получаю все возможные пары "отдел-проект". Для большинства пар в `emp_name` будет `NULL`, так как такой связки не существует. Но если в отделе есть сотрудник, работающий над проектом, его имя будет указано:
```
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
```
shared_preload_libraries = 'pg_stat_statements'
track_io_timing = on  # Включить отслеживание времени I/O
```
Перезагружаю сервер. Создаю расширение в базе данных:
```
postgres=# CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
CREATE EXTENSION
```
#### Метрика 1. Выявление самых медленных JOIN в системе для их оптимизации:
```
SELECT
    query,
    calls,
    total_exec_time,
    mean_exec_time,
    rows,
    regexp_replace(query, '.*(FROM.*JOIN.*WHERE).*', '\1', 'i') as join_section
FROM
    pg_stat_statements
WHERE
    query LIKE '%JOIN%'
    AND query NOT LIKE '%pg_stat_%'
ORDER BY
    total_exec_time DESC
LIMIT 10;
```
Запросы наверху списка дают наибольшую нагрузку на систему. Их стоит изучать в первую очередь: проверять индексы, условия соединения, возможность денормализации.
#### Метрика 2. Неоптимальные JOIN (nested loop без индексов):
```
SELECT
    s.query,
    s.calls,
    s.mean_exec_time,
    CASE
        WHEN s.query LIKE '%Nested Loop%' AND s.query LIKE '%Seq Scan%' THEN 'Nested Loop with Seq Scan'
        WHEN s.query LIKE '%Hash Join%' THEN 'Hash Join'
        WHEN s.query LIKE '%Merge Join%' THEN 'Merge Join'
        ELSE 'Other'
    END as join_type
FROM
    pg_stat_statements s
WHERE
    s.query LIKE '%JOIN%'
    AND s.query NOT LIKE '%pg_stat_%'
    AND (s.query LIKE '%Nested Loop%' AND s.query LIKE '%Seq Scan%')
ORDER BY
    s.total_exec_time DESC
LIMIT 10;
```
Вывод показывает самые неэффективные JOIN, где PostgreSQL вынужден в цикле полностью сканировать одну из таблиц. 
#### Метрика 3. Статистика по типам JOIN и их эффективности:
```
SELECT
    CASE
        WHEN query ILIKE '%INNER JOIN%' THEN 'INNER JOIN'
        WHEN query ILIKE '%LEFT JOIN%' THEN 'LEFT JOIN'
        WHEN query ILIKE '%RIGHT JOIN%' THEN 'RIGHT JOIN'
        WHEN query ILIKE '%FULL JOIN%' THEN 'FULL JOIN'
        WHEN query ILIKE '%CROSS JOIN%' THEN 'CROSS JOIN'
        ELSE 'Other JOIN'
    END AS join_type,
    COUNT(*) AS query_count,
    SUM(calls) AS total_calls,
    ROUND(SUM(total_exec_time)::numeric, 2) AS total_time,
    ROUND(AVG(mean_exec_time)::numeric, 2) AS avg_time,
    ROUND(
        ((SUM(rows)::numeric / NULLIF(SUM(total_exec_time)::numeric, 0)) * 1000)::numeric,
        2
    ) AS rows_per_second
FROM
    pg_stat_statements
WHERE
    query LIKE '%JOIN%'
    AND query NOT LIKE '%pg_stat_%'
GROUP BY
    join_type
ORDER BY
    total_time DESC;
```
