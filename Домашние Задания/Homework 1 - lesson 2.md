```
Окружение: ВМ на Oracle Linux 8.10. PosgreSQL 17.
```

Запускаю две сессии psql из-под пользователя postgres. Выключаю AUTOCOMMIT:
```
postgres=# \set AUTOCOMMIT off
postgres=# \echo :AUTOCOMMIT
off
```
Делаю в первой сессии таблицу:
```
postgres=# create table persons(id serial, first_name text, second_name text); insert into persons(first_name, second_name) values('ivan', 'ivanov'); insert into persons(first_name, second_name) values('petr', 'petrov'); commit;
CREATE TABLE
INSERT 0 1
INSERT 0 1
COMMIT
```
Уровень изоляции:
```
postgres=*# show transaction isolation level;
 transaction_isolation
-----------------------
 read committed
(1 строка)
```
Начинаю новую транзакцию, в первой сессии добавляю новую запись:
```
postgres=# begin;
BEGIN
postgres=*# insert into persons(first_name, second_name) values('sergey', 'sergeev');
INSERT 0 1
```
Во второй сессии делаю `select * from persons`:
```
postgres-# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 строки)
postgres-#
```
Не вижу новую запись, так как AUTOCOMMIT отключён, а текущий уровень транзакции = `read committed`, т.е. показываются только записи, которые уже были зафиксированы (COMMIT). <br>
Возвращаюсь в первую сессию, делаю COMMIT, переключаюсь во вторую сессию, делаю `select * from persons`:
```
postgres=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 строки)
```
Новая запись про Сергеева Сергея появилась, т.к. мы зафиксировали (COMMIT) транзакцию в первой сессии. <br>
Переключаюсь на другой уровень изоляции `repeatable read`: 
```
postgres=# set transaction isolation level repeatable read;
SET
```
В первой сессии добавляю новую запись:
```
postgres=*# insert into persons(first_name, second_name) values('sveta', 'svetova');
INSERT 0 1
```
Во второй сессии делаю `select * from persons`. Вижу:
```
postgres=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 строки)
```
Новая запись не появилась, т.к. мы работаем на уровне изоляции = `repeatable read`, что означает, что будут видны только те данные, которые были зафиксированы до начала транзакции, а новые и незафиксированные изменения видны не будут. <br>
Завершаю транзакцию в первой сессии, делаю `select * from persons`:
```
postgres=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 строки)
```
Снова вижу отсутствие изменений, что опять же объясняется уровнем изоляции = `repeatable read.` <br>
Завершаю транзакцию во второй сессии, повторно делаю `select * from persons`:
```
postgres=*# commit;
COMMIT
postgres=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
(4 строки)

postgres=*# show transaction isolation level;
 transaction_isolation
-----------------------
 read committed
(1 строка)

```
Теперь новая запись видна, т.к. после фиксации транзакции мы перешли на уровень изоляции read committed.
