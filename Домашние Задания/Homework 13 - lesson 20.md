```
Окружение: Oracle Linux 8.10. PostgreSQL17.
```
# Бэкапы
### Создаю БД, схему и в ней таблицу. Заполняю таблицу автосгенерированными 100 записями.
```sql
CREATE DATABASE homework_db;
CREATE DATABASE

\c homework_db
Вы подключены к базе данных "homework_db" как пользователь "postgres".

CREATE SCHEMA IF NOT EXISTS homework_schema;
CREATE SCHEMA

SET search_path TO homework_schema;
SET

CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    email VARCHAR(100),
    salary DECIMAL(10,2),
    hire_date DATE
);

CREATE TABLE departments (
    id SERIAL PRIMARY KEY,
    department_name VARCHAR(100) NOT NULL,
    manager_id INTEGER
);
```
### Заполняю таблицы автосгенерированными данными (100 записей)
```sql
INSERT INTO employees (first_name, last_name, email, salary, hire_date)
SELECT 
    'FirstName_' || seq,
    'LastName_' || seq,
    'user_' || seq || '@company.com',
    (RANDOM() * 10000) + 3000,
    CURRENT_DATE - (FLOOR(RANDOM() * 3650))::INT
FROM generate_series(1, 100) seq;
INSERT 0 100

SELECT COUNT(*) FROM employees;
 count
-------
   100
(1 строка)

INSERT INTO departments (department_name, manager_id)
VALUES 
    ('IT', 1),
    ('HR', 2),
    ('Finance', 3),
    ('Marketing', 4),
    ('Sales', 5);
INSERT 0 5

SELECT * FROM employees LIMIT 5;
 id | first_name  | last_name  |       email        |  salary  | hire_date
----+-------------+------------+--------------------+----------+------------
  1 | FirstName_1 | LastName_1 | user_1@company.com |  4357.16 | 2023-10-23
  2 | FirstName_2 | LastName_2 | user_2@company.com |  4023.52 | 2021-04-23
  3 | FirstName_3 | LastName_3 | user_3@company.com | 11697.41 | 2023-06-19
  4 | FirstName_4 | LastName_4 | user_4@company.com |  3990.73 | 2016-07-07
  5 | FirstName_5 | LastName_5 | user_5@company.com |  7599.74 | 2023-09-24
(5 строк)

SELECT * FROM departments;
 id | department_name | manager_id
----+-----------------+------------
  1 | IT              |          1
  2 | HR              |          2
  3 | Finance         |          3
  4 | Marketing       |          4
  5 | Sales           |          5
(5 строк)
```
### Выхожу из psql, создаю каталог для бэкапов:
```
homework_db=# \q
[postgres@postgresql ~]$ mkdir -p /var/lib/pgsql/backups
[postgres@postgresql ~]$ ls -la /var/lib/pgsql/backups
итого 8
drwxr-xr-x. 2 postgres postgres 4096 окт 15 14:12 .
drwx------. 6 postgres postgres 4096 окт 15 14:12 ..
[postgres@postgresql ~]$ chmod 700 /var/lib/pgsql/backups
```
### Делаю логический бэкап с помощью COPY.
```
psql -d homework_db

COPY homework_schema.employees TO '/var/lib/pgsql/backups/employees_backup.csv' WITH CSV HEADER;
COPY 100

COPY homework_schema.departments TO '/var/lib/pgsql/backups/departments_backup.csv' WITH CSV HEADER;
COPY 5

ls -la /var/lib/pgsql/backups/*.csv
-rw-r--r--. 1 postgres postgres   80 окт 15 14:16 /var/lib/pgsql/backups/departments_backup.csv
-rw-r--r--. 1 postgres postgres 6745 окт 15 14:15 /var/lib/pgsql/backups/employees_backup.csv
```
### Восстанавливаю данные во вторую таблицу из бэкапа:
```
```
```
```
```
```
```
```
```
```
```
```
```
```
```
```
```
```
