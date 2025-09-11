```
Окружение: Oracle Limux 8.10, PostgreSQL 17.
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

## 1. Реализовать прямое соединение двух или более таблиц:
