# Работа с базами данных, пользователями и правами.
```
Окружение: Oracle Linux 8.10. PostgreSQL 17.
```
Инициализирую новый кластер, подключаюсь к кластеру и создаю новую БД `testdb`:
```
[root@postgresql ~]# sudo -u postgres psql
psql (17.5)
Введите "help", чтобы получить справку.

postgres=# CREATE DATABASE testdb;
CREATE DATABASE
```
Подключаюсь к БД `testdb`, создаю схему `testnm`, создаю таблицу `t1`, вставляю строку со значением `c1=1`:
```
postgres=# \c testdb
Вы подключены к базе данных "testdb" как пользователь "postgres".
testdb=# CREATE SCHEMA testnm;
CREATE SCHEMA
testdb=# CREATE TABLE t1 (c1 INTEGER);
CREATE TABLE
testdb=# INSERT INTO t1 VALUES (1);
INSERT 0 1
```
Cоздаю новую роль `readonly`, даю новой роли право на подключение к базе данных `testdb`, даю новой роли право на использование схемы `testnm`, даю новой роли право на `select` для всех таблиц схемы `testnm`, создаю пользователя `testread` с паролем `test123`, даю роль `readonly` пользователю `testread`:
```
testdb=# CREATE ROLE readonly;
CREATE ROLE
testdb=# GRANT CONNECT ON DATABASE testdb TO readonly;
GRANT
testdb=# GRANT USAGE ON SCHEMA testnm TO readonly;
GRANT
testdb=# GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;
GRANT
testdb=# CREATE USER testread WITH PASSWORD 'test123';
CREATE ROLE
testdb=# GRANT readonly TO testread;
GRANT ROLE
```
Выхожу из текущей сессии, захожу под пользователем `testread` в базу данных `testdb`, делаю `select * from t1;`:
```
testdb=# \q
[root@postgresql data]# psql -U testread -d testdb -h localhost
Пароль пользователя testread:
psql (17.5)
Введите "help", чтобы получить справку.

testdb=> SELECT * FROM t1;
ОШИБКА:  нет доступа к таблице t1
```
Вышла `ОШИБКА:  нет доступа к таблице t1`, потому что таблица создана в схеме `public` (по умолчанию), а мы дали права только на схему `testnm`. Перезахожу под пользователем `postgres`, удаляю таблицу в `public`, создаю заново в схеме `testnm`:
```
[root@postgresql data]# sudo -u postgres psql
psql (17.5)
Введите "help", чтобы получить справку.

postgres=# \c testdb
Вы подключены к базе данных "testdb" как пользователь "postgres".
testdb=# DROP TABLE t1;
DROP TABLE
testdb=# CREATE TABLE testnm.t1 (c1 INTEGER);
CREATE TABLE
testdb=# INSERT INTO testnm.t1 VALUES (1);
INSERT 0 1
```
Снова подключаюсь под `testread`, пробую `select * from testnm.t1;`:
```
[root@postgresql data]# psql -U testread -d testdb -h localhost
Пароль пользователя testread:
psql (17.5)
Введите "help", чтобы получить справку.

testdb=> SELECT * FROM testnm.t1;
ОШИБКА:  нет доступа к таблице t1
```
Снова возникла ошибка. Ошибка возникает, потому что права `select` были выданы до создания таблицы, нужно дать права на все будущие таблицы в схеме `testnm` и на существующую таблицу:
```
Вы подключены к базе данных "testdb" как пользователь "postgres".
testdb=# ALTER DEFAULT PRIVILEGES IN SCHEMA testnm GRANT SELECT ON TABLES TO readonly;
ALTER DEFAULT PRIVILEGES
testdb=# GRANT SELECT ON testnm.t1 TO readonly;
GRANT
```
Переключаюсь обратно на пользователя `testread`, снова проверяю:
```
[root@postgresql data]# psql -U testread -d testdb -h localhost
Пароль пользователя testread:
psql (17.5)
Введите "help", чтобы получить справку.

testdb=> SELECT * FROM testnm.t1;
 c1
----
  1
(1 строка)
```
Теперь все работает.
