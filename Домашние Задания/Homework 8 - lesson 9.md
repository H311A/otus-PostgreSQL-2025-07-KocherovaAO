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
### Воспроизвожу ситуацию с длительной блокировкой:
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
