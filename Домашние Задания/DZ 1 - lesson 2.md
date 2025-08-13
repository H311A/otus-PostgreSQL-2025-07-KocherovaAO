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
