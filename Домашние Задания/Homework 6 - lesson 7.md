# Настройка autovacuum с учетом особеностей производительности.
```
Окружение: Oracle Linux 8.10, 2 CPU, 4 RAM, 40 GB SSD (не буду пересоздавать новую машину с 10 SSD, использую текущую с предыдущих занятий).
```
Устанавливаю чистый PostgeSQL 17 с дефолтными настройками, создаю новую БД для тестов:
```
[root@postgresql ~]# yum remove postgresql17*
Удален:
  postgresql17-17.6-1PGDG.rhel8.x86_64                                       postgresql17-contrib-17.6-1PGDG.rhel8.x86_64
  postgresql17-libs-17.6-1PGDG.rhel8.x86_64                                  postgresql17-server-17.6-1PGDG.rhel8.x86_64
[root@postgresql pgsql]# rm -rf /var/lib/pgsql/17

[root@postgresql pgsql]# yum install postgresql17*
Установлен:
  postgresql17-17.6-1PGDG.rhel8.x86_64                                       postgresql17-libs-17.6-1PGDG.rhel8.x86_64
  postgresql17-server-17.6-1PGDG.rhel8.x86_64                                postgresql17-contrib-17.6-1PGDG.rhel8.x86_64
[root@postgresql pgsql]# /usr/pgsql-17/bin/postgresql-17-setup initdb
Initializing database ... OK
[root@postgresql pgsql]# systemctl start postgresql-17
[root@postgresql pgsql]# systemctl enable postgresql-17
Created symlink /etc/systemd/system/multi-user.target.wants/postgresql-17.service → /usr/lib/systemd/system/postgresql-17.service.

[root@postgresql pgsql]# sudo -u postgres /usr/pgsql-17/bin/pgbench -i postgres
dropping old tables...
creating tables...
generating data (client-side)...
vacuuming...
creating primary keys...
done in 0.47 s (drop tables 0.01 s, create tables 0.01 s, client-side generate 0.24 s, vacuum 0.10 s, primary keys 0.11 s).
```
Запускаю тест `pgbench`:
```
[root@postgresql pgsql]# sudo -u postgres /usr/pgsql-17/bin/pgbench -c8 -P6 -T60 -U postgres postgres
pgbench (17.6)
starting vacuum...end.
progress: 6.0 s, 441.7 tps, lat 17.716 ms stddev 24.476, 0 failed
progress: 12.0 s, 424.2 tps, lat 18.813 ms stddev 14.123, 0 failed
progress: 18.0 s, 441.5 tps, lat 18.005 ms stddev 13.858, 0 failed
progress: 24.0 s, 364.5 tps, lat 21.984 ms stddev 19.018, 0 failed
progress: 30.0 s, 434.2 tps, lat 18.404 ms stddev 13.088, 0 failed
progress: 36.0 s, 446.5 tps, lat 17.875 ms stddev 13.044, 0 failed
progress: 42.0 s, 453.3 tps, lat 17.598 ms stddev 12.139, 0 failed
progress: 48.0 s, 414.2 tps, lat 19.247 ms stddev 13.218, 0 failed
progress: 54.0 s, 416.5 tps, lat 19.113 ms stddev 13.242, 0 failed
progress: 60.0 s, 435.2 tps, lat 18.422 ms stddev 13.441, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 25637
number of failed transactions: 0 (0.000%)
latency average = 18.653 ms
latency stddev = 15.397 ms
initial connection time = 109.682 ms
tps = 427.785439 (without initial connection time)
```
Изменяю параметры postgresql.conf:
```
max_connections = 40
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 512MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 500
random_page_cost = 4
effective_io_concurrency = 2
work_mem = 6553kB
min_wal_size = 4GB
max_wal_size = 16GB
```
Повторно запускаю тест `pgbench`:
```
[root@postgresql data]# systemctl restart postgresql-17
[root@postgresql data]# sudo -u postgres /usr/pgsql-17/bin/pgbench -c8 -P6 -T60 -U postgres postgres
pgbench (17.6)
starting vacuum...end.
progress: 6.0 s, 410.4 tps, lat 19.064 ms stddev 14.010, 0 failed
progress: 12.0 s, 401.7 tps, lat 19.948 ms stddev 15.087, 0 failed
progress: 18.0 s, 416.8 tps, lat 19.184 ms stddev 14.304, 0 failed
progress: 24.0 s, 444.6 tps, lat 17.879 ms stddev 11.822, 0 failed
progress: 30.0 s, 473.8 tps, lat 16.904 ms stddev 13.005, 0 failed
progress: 36.0 s, 458.4 tps, lat 17.340 ms stddev 12.254, 0 failed
progress: 42.0 s, 399.7 tps, lat 20.004 ms stddev 15.606, 0 failed
progress: 48.0 s, 425.3 tps, lat 18.823 ms stddev 13.181, 0 failed
progress: 54.0 s, 467.2 tps, lat 17.090 ms stddev 13.034, 0 failed
progress: 60.0 s, 480.8 tps, lat 16.597 ms stddev 11.738, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 26280
number of failed transactions: 0 (0.000%)
latency average = 18.208 ms
latency stddev = 13.451 ms
initial connection time = 72.786 ms
tps = 438.150439 (without initial connection time)
```
После применения новых параметров конфигурации производительность PostgreSQL улучшилась: количество транзакций в секунду увеличилось, средняя задержка уменьшилась, а стабильность работы повысилась (снижение стандартного отклонения задержки). Это произошло благодаря оптимизации использования оперативной памяти (увеличение shared_buffers и work_mem), более эффективной работе с WAL (настройка размеров и контрольных точек) и улучшению планирования запросов за счет точной статистики.<br/>
Создаю новую таблицу с текстовым полем, заполняю данными и смотрю размер файла с таблицей:
```
postgres=# CREATE TABLE test_table (id serial PRIMARY KEY, text_data text);
CREATE TABLE
postgres=# INSERT INTO test_table (text_data)
postgres-# SELECT md5(random()::text)
postgres-# FROM generate_series(1, 1000000);
INSERT 0 1000000

postgres=# SELECT pg_size_pretty(pg_total_relation_size('test_table'));
 pg_size_pretty
----------------
 87 MB
(1 строка)
```
5 раз обновляю все строчки и добавляю к каждой строчке символ. Смотрю количество мертвых строчек в таблице, и когда последний раз приходил автовакуум:
```
postgres=# DO $$
postgres$# BEGIN
postgres$#     FOR i IN 1..5 LOOP
postgres$#         UPDATE test_table SET text_data = text_data || 'X';
postgres$#     END LOOP;
postgres$# END $$;
DO

postgres=# SELECT n_dead_tup, last_autovacuum
postgres-# FROM pg_stat_user_tables
postgres-# WHERE relname = 'test_table';
 n_dead_tup |        last_autovacuum
------------+-------------------------------
    5000000 | 2025-08-29 13:50:19.030668+03
(1 строка)
```
Жду некоторое время, переодически проверяя, произошёл ли автовакуум:
```
postgres=# SELECT n_dead_tup, last_autovacuum
FROM pg_stat_user_tables
WHERE relname = 'test_table';
 n_dead_tup |        last_autovacuum
------------+-------------------------------
          0 | 2025-08-29 13:56:30.628072+03
```
Теперь снова обновляю 5 раз строчки с добавлением символа, смотрю размер файла с таблицей:
```
postgres=# DO $$
postgres$# BEGIN
postgres$#     FOR i IN 1..5 LOOP
postgres$#         UPDATE test_table SET text_data = text_data || 'Y';
postgres$#     END LOOP;
postgres$# END $$;
DO

postgres=# SELECT pg_size_pretty(pg_total_relation_size('test_table'));
 pg_size_pretty
----------------
 524 MB
(1 строка)
```
Теперь отключаю автовакуум на своей таблице, 10 раз обновляю все строчки с добавлением к каждой строчке символа, и смотрю размер файла с таблицей:
```
postgres=# ALTER TABLE test_table SET (autovacuum_enabled = false);
ALTER TABLE
postgres=# DO $$
postgres$# BEGIN
postgres$#     FOR i IN 1..10 LOOP
postgres$#         UPDATE test_table SET text_data = text_data || 'Z';
postgres$#     END LOOP;
postgres$# END $$;
DO

postgres=# SELECT pg_size_pretty(pg_total_relation_size('test_table'));
 pg_size_pretty
----------------
 1008 MB
(1 строка)
```
При отключенном автовакууме размер таблицы значительно увеличился, так как PostgreSQL не очищал мертвые строки. Каждое обновление создавало новые версии строк, а старые оставались в таблице, занимая место. Автовакуум автоматически удаляет эти мертвые строки, возвращая пространство для повторного использования. Включаю автовакуум обратно:
```
postgres=# ALTER TABLE test_table SET (autovacuum_enabled = true);
ALTER TABLE
```
## Задача со звёздочкой.
```
postgres=# DO $$
postgres$# DECLARE
postgres$#     i INTEGER;
postgres$#     table_size TEXT;
postgres$# BEGIN
postgres$#     FOR i IN 1..10 LOOP
postgres$#         UPDATE test_table SET text_data = text_data || 'U';
postgres$#         SELECT pg_size_pretty(pg_total_relation_size('test_table')) INTO table_size;
postgres$#         RAISE NOTICE 'Шаг цикла: %, Размер таблицы: %', i, table_size;
postgres$#         PERFORM pg_sleep(1);
postgres$#     END LOOP;
postgres$# END $$;
ЗАМЕЧАНИЕ:  Шаг цикла: 1, Размер таблицы: 1008 MB
ЗАМЕЧАНИЕ:  Шаг цикла: 2, Размер таблицы: 1008 MB
ЗАМЕЧАНИЕ:  Шаг цикла: 3, Размер таблицы: 1008 MB
ЗАМЕЧАНИЕ:  Шаг цикла: 4, Размер таблицы: 1008 MB
ЗАМЕЧАНИЕ:  Шаг цикла: 5, Размер таблицы: 1008 MB
ЗАМЕЧАНИЕ:  Шаг цикла: 6, Размер таблицы: 1008 MB
ЗАМЕЧАНИЕ:  Шаг цикла: 7, Размер таблицы: 1008 MB
ЗАМЕЧАНИЕ:  Шаг цикла: 8, Размер таблицы: 1008 MB
ЗАМЕЧАНИЕ:  Шаг цикла: 9, Размер таблицы: 1032 MB
ЗАМЕЧАНИЕ:  Шаг цикла: 10, Размер таблицы: 1128 MB
DO
```
До шага 9 автовакуум успевал очищать мертвые строки между итерациями цикла, возвращая пространство для повторного использования. На шаге 9 количество мертвых строк превысило порог срабатывания автовакуума, что потребовало расширения файла.
