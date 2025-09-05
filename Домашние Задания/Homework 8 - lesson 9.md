```
Окружение: Oracle Linux 8.10.
```
# Механизм блокировок.
### Настройка логирования длительных блокировок.
Проверяю текущие настройки:
```
postgres=# SELECT name, setting, unit FROM pg_settings WHERE name LIKE '%lock%' OR name LIKE '%deadlock%' OR name LIKE '%log%';
                name                 |      setting      | unit
-------------------------------------+-------------------+------
 block_size                          | 8192              |
 deadlock_timeout                    | 1000              | ms
 debug_logical_replication_streaming | buffered          |
 lock_timeout                        | 0                 | ms
 log_autovacuum_min_duration         | 600000            | ms
 log_checkpoints                     | on                |
 log_connections                     | off               |
 log_destination                     | stderr            |
 log_directory                       | log               |
 log_disconnections                  | off               |
 log_duration                        | off               |
 log_error_verbosity                 | default           |
 log_executor_stats                  | off               |
 log_file_mode                       | 0600              |
 log_filename                        | postgresql-%a.log |
 log_hostname                        | off               |
 log_line_prefix                     | %m [%p]           |
 log_lock_waits                      | off               |
 log_min_duration_sample             | -1                | ms
 log_min_duration_statement          | -1                | ms
 log_min_error_statement             | error             |
 log_min_messages                    | warning           |
 log_parameter_max_length            | -1                | B
 log_parameter_max_length_on_error   | 0                 | B
 log_parser_stats                    | off               |
 log_planner_stats                   | off               |
 log_recovery_conflict_waits         | off               |
 log_replication_commands            | off               |
 log_rotation_age                    | 1440              | min
 log_rotation_size                   | 0                 | kB
 log_startup_progress_interval       | 10000             | ms
 log_statement                       | none              |
 log_statement_sample_rate           | 1                 |
 log_statement_stats                 | off               |
 log_temp_files                      | -1                | kB
 log_timezone                        | Europe/Moscow     |
 log_transaction_sample_rate         | 0                 |
 log_truncate_on_rotation            | on                |
 logging_collector                   | on                |
 logical_decoding_work_mem           | 65536             | kB
 max_locks_per_transaction           | 64                |
 max_logical_replication_workers     | 4                 |
 max_pred_locks_per_page             | 2                 |
 max_pred_locks_per_relation         | -2                |
 max_pred_locks_per_transaction      | 64                |
 syslog_facility                     | local0            |
 syslog_ident                        | postgres          |
 syslog_sequence_numbers             | on                |
 syslog_split_messages               | on                |
 wal_block_size                      | 8192              |
 wal_log_hints                       | off               |
(51 строка)
```
Устанавливаю параметр логирования блокировок:
```
postgres=# ALTER SYSTEM SET log_lock_waits = on;
ALTER SYSTEM
postgres=# ALTER SYSTEM SET deadlock_timeout = '200ms';
ALTER SYSTEM
```
Применяю изменения, проверяю настройки:
```
postgres=# SELECT pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 строка)

postgres=# SHOW log_lock_waits;
 log_lock_waits
----------------
 on
(1 строка)

postgres=# SHOW deadlock_timeout;
 deadlock_timeout
------------------
 200ms
(1 строка)
```
#### Воспроизвожу ситуацию с длительной блокировкой:
Сеанс 1:
```
postgres=# CREATE TABLE test_locks (id SERIAL PRIMARY KEY, value INTEGER);
CREATE TABLE
postgres=# INSERT INTO test_locks (value) VALUES (100), (200), (300);
INSERT 0 3
postgres=# BEGIN;
BEGIN
postgres=*# UPDATE test_locks SET value = value + 1 WHERE id = 1;
UPDATE 1
```
Сеанс 2:
```
postgres=# BEGIN;
BEGIN
postgres=*# UPDATE test_locks SET value = value + 10 WHERE id = 1;
```
Лог PostgreSQL сообщает: 
```
2025-09-05 12:58:21.590 MSK [56771] СООБЩЕНИЕ:  процесс 56771 продолжает ожидать в режиме ShareLock блокировку "транзакция 273521" в течение 201.017 мс
```
### Три команды UPDATE.
Создаю представление для удобного представления блокировок:
```
CREATE VIEW locks_v AS
SELECT pid,
       locktype,
       CASE locktype
         WHEN 'relation' THEN relation::regclass::text
         WHEN 'transactionid' THEN transactionid::text
         WHEN 'tuple' THEN relation::regclass::text||':'||tuple::text
       END AS lockid,
       mode,
       granted
FROM pg_locks
WHERE locktype in ('relation','transactionid','tuple');
```
Сеанс 1:
```
postgres=# BEGIN;
BEGIN
postgres=*# UPDATE test_locks SET value = 111 WHERE id = 1;
UPDATE 1
```
Сеанс 2:
```
postgres=# BEGIN;
BEGIN
postgres=*# UPDATE test_locks SET value = 222 WHERE id = 1;
```
Сеанс 3:
```
postgres=# BEGIN;
BEGIN
postgres=*# UPDATE test_locks SET value = 333 WHERE id = 1;
```
В четвертой сессии выполняю `SELECT * FROM locks_v ORDER BY pid, granted;`
```
postgres=# SELECT * FROM locks_v ORDER BY pid, granted;
  pid  |   locktype    |     lockid      |       mode       | granted
-------+---------------+-----------------+------------------+---------
 59432 | relation      | test_locks_pkey | RowExclusiveLock | t
 59432 | relation      | test_locks      | RowExclusiveLock | t
 59432 | transactionid | 273523          | ExclusiveLock    | t
 59675 | transactionid | 273523          | ShareLock        | f
 59675 | relation      | test_locks_pkey | RowExclusiveLock | t
 59675 | tuple         | test_locks:1    | ExclusiveLock    | t
 59675 | transactionid | 273524          | ExclusiveLock    | t
 59675 | relation      | test_locks      | RowExclusiveLock | t
 60031 | tuple         | test_locks:1    | ExclusiveLock    | f
 60031 | relation      | test_locks_pkey | RowExclusiveLock | t
 60031 | transactionid | 273525          | ExclusiveLock    | t
 60031 | relation      | test_locks      | RowExclusiveLock | t
 60661 | relation      | pg_locks        | AccessShareLock  | t
 60661 | relation      | locks_v         | AccessShareLock  | t
(14 строк)
```
pid 59432 (первая транзакция):  
`relation | test_locks_pkey | RowExclusiveLock | t` - блокировка индекса для изменения данных;  
`relation | test_locks | RowExclusiveLock | t` - блокировка таблицы для изменения данных;  
`transactionid | 273523 | ExclusiveLock | t` - блокировка ID своей транзакции.  
  
pid 59675 (вторая транзакция, ждет первую):  
`transactionid | 273523 | ShareLock | f` - ожидает завершения транзакции 273523 (первой);  
`relation | test_locks_pkey | RowExclusiveLock | t` - блокировка индекса;  
`tuple | test_locks:1 | ExclusiveLock | t` - блокировка версии строки (держит место в очереди);  
`transactionid | 273524 | ExclusiveLock | t` - блокировка ID своей транзакции;  
`relation | test_locks | RowExclusiveLock | t` - блокировка таблицы.  
  
pid 60031 (третья транзакция, ждет вторую):  
`tuple | test_locks:1 | ExclusiveLock | f` - ожидает освобождения версии строки (её держит вторая транзакция);  
`relation | test_locks_pkey | RowExclusiveLock | t` - блокировка индекса;  
`transactionid | 273525 | ExclusiveLock | t` - блокировка ID своей транзакции;  
`relation | test_locks | RowExclusiveLock | t` - блокировка таблицы.  
  
pid 60661 (служебная сессия):  
`relation | pg_locks | AccessShareLock | t` - блокировка для чтения системной таблицы;  
`relation | locks_v | AccessShareLock | t` - блокировка для чтения представления.  

### Воспроизведение взаимоблокировки.
Сеанс 1:
```
postgres=# BEGIN;
BEGIN
postgres=*# UPDATE test_locks SET value = 111 WHERE id = 1;
UPDATE 1
```
Сеанс 2:
```
postgres=# BEGIN;
BEGIN
postgres=*# UPDATE test_locks SET value = 222 WHERE id = 2;
UPDATE 1
```
Сеанс 3:
```
postgres=# BEGIN;
BEGIN
postgres=*# UPDATE test_locks SET value = 333 WHERE id = 3;
UPDATE 1
```
Сеанс 1:
```
postgres=*# UPDATE test_locks SET value = 444 WHERE id = 2;
```
Сеанс 2:
```
postgres=*# UPDATE test_locks SET value = 555 WHERE id = 3;
```
Сеанс 3:
```
postgres=*# UPDATE test_locks SET value = 666 WHERE id = 1;
ОШИБКА:  обнаружена взаимоблокировка
ПОДРОБНОСТИ:  Процесс 68532 ожидает в режиме ShareLock блокировку "транзакция 273527"; заблокирован процессом 66510.
Процесс 66510 ожидает в режиме ShareLock блокировку "транзакция 273528"; заблокирован процессом 66547.
Процесс 66547 ожидает в режиме ShareLock блокировку "транзакция 273529"; заблокирован процессом 68532.
ПОДСКАЗКА:  Подробности запроса смотрите в протоколе сервера.
КОНТЕКСТ:  при изменении кортежа (0,1) в отношении "test_locks"
```
Журнал PostgreSQL сообщает:
```
2025-09-05 13:29:18.246 MSK [66547] СООБЩЕНИЕ:  процесс 66547 продолжает ожидать в режиме ShareLock блокировку "транзакция 273529" в течение 200.383 мс
2025-09-05 13:29:18.246 MSK [66547] ПОДРОБНОСТИ:  Process holding the lock: 68532. Wait queue: 66547.
2025-09-05 13:29:18.246 MSK [66547] КОНТЕКСТ:  при изменении кортежа (0,3) в отношении "test_locks"
2025-09-05 13:29:18.246 MSK [66547] ОПЕРАТОР:  UPDATE test_locks SET value = 555 WHERE id = 3;
2025-09-05 13:29:37.415 MSK [68532] СООБЩЕНИЕ:  процесс 68532 обнаружил взаимоблокировку, ожидая в режиме ShareLock блокировку "транзакция 273527" в течение 200.551 мс
2025-09-05 13:29:37.415 MSK [68532] ПОДРОБНОСТИ:  Process holding the lock: 66510. Wait queue: .
2025-09-05 13:29:37.415 MSK [68532] КОНТЕКСТ:  при изменении кортежа (0,1) в отношении "test_locks"
2025-09-05 13:29:37.415 MSK [68532] ОПЕРАТОР:  UPDATE test_locks SET value = 666 WHERE id = 1;
2025-09-05 13:29:37.415 MSK [68532] ОШИБКА:  обнаружена взаимоблокировка
2025-09-05 13:29:37.415 MSK [68532] ПОДРОБНОСТИ:  Процесс 68532 ожидает в режиме ShareLock блокировку "транзакция 273527"; заблокирован процессом 66510.
        Процесс 66510 ожидает в режиме ShareLock блокировку "транзакция 273528"; заблокирован процессом 66547.
        Процесс 66547 ожидает в режиме ShareLock блокировку "транзакция 273529"; заблокирован процессом 68532.
        Процесс 68532: UPDATE test_locks SET value = 666 WHERE id = 1;
        Процесс 66510: UPDATE test_locks SET value = 444 WHERE id = 2;
        Процесс 66547: UPDATE test_locks SET value = 555 WHERE id = 3;
2025-09-05 13:29:37.415 MSK [68532] ПОДСКАЗКА:  Подробности запроса смотрите в протоколе сервера.
2025-09-05 13:29:37.415 MSK [68532] КОНТЕКСТ:  при изменении кортежа (0,1) в отношении "test_locks"
2025-09-05 13:29:37.415 MSK [68532] ОПЕРАТОР:  UPDATE test_locks SET value = 666 WHERE id = 1;
2025-09-05 13:29:37.416 MSK [66547] СООБЩЕНИЕ:  процесс 66547 получил в режиме ShareLock блокировку "транзакция 273529" через 19370.298 мс
2025-09-05 13:29:37.416 MSK [66547] КОНТЕКСТ:  при изменении кортежа (0,3) в отношении "test_locks"
2025-09-05 13:29:37.416 MSK [66547] ОПЕРАТОР:  UPDATE test_locks SET value = 555 WHERE id = 3;
```
По журналу можно полностью разобраться в ситуации взаимоблокировки трех транзакций постфактум. PostgreSQL показывает какой процесс кого блокирует, циклическую цепочку ожидания, какие ресурсы вовлечены, какие команды выполнялись.
