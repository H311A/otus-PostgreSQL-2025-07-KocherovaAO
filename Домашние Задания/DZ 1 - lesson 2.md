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
Во второй сессии делаю select * from persons:
```
postgres-# select * from persons
postgres-#
```
Не вижу новую запись, так как AUTOCOMMIT отключён, а текущий уровень транзакции = read committed, т.е. показываются только записи, которые уже были зафиксированы (COMMIT).
