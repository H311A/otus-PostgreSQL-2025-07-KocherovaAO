```
Окружение: Oracle Linux 8.10. 2 CPU 2 RAM 40GB SSD.
```
## Произвожу установку PostgreSQL 17, инициализирую кластер: 
```
[root@postgresql ~]# yum install -y postgresql17-server postgresql17-contrib
Установлен:
  postgresql17-contrib-17.6-1PGDG.rhel8.x86_64                                   postgresql17-server-17.6-1PGDG.rhel8.x86_64
[root@postgresql ~]# /usr/pgsql-17/bin/postgresql-17-setup initdb
Initializing database ... OK
[root@postgresql ~]# systemctl enable postgresql-17
Created symlink /etc/systemd/system/multi-user.target.wants/postgresql-17.service → /usr/lib/systemd/system/postgresql-17.service.
[root@postgresql ~]# systemctl start postgresql-17
[root@postgresql ~]# systemctl status postgresql-17
● postgresql-17.service - PostgreSQL 17 database server
   Loaded: loaded (/usr/lib/systemd/system/postgresql-17.service; enabled; vendor preset: disabled)
   Active: active (running) since Tue 2025-08-26 15:31:22 MSK; 11s ago
     Docs: https://www.postgresql.org/docs/17/static/
  Process: 4334 ExecStartPre=/usr/pgsql-17/bin/postgresql-17-check-db-dir ${PGDATA} (code=exited, status=0/SUCCESS)
 Main PID: 4339 (postgres)
    Tasks: 7 (limit: 10359)
   Memory: 17.9M
   CGroup: /system.slice/postgresql-17.service
           ├─4339 /usr/pgsql-17/bin/postgres -D /var/lib/pgsql/17/data/
           ├─4341 postgres: logger
           ├─4342 postgres: checkpointer
           ├─4343 postgres: background writer
           ├─4345 postgres: walwriter
           ├─4346 postgres: autovacuum launcher
           └─4347 postgres: logical replication launcher
```
## Настраиваю кластер на максимальную производительность: 
- выключаю `fsync`, чтобы Postgres не ждал записи на диск;
- выключаю `synchronous_commit`, чтобы убрать задержки подтверждения операций;
- выключаю `full_page_writes`, чтобы убрать избыточную запись страниц, не будем писать лишние данные;
- даю буферу (`shared_buffers)` максимально возможный объем для моих 2 RAM - 50%;
- задаю 512MB объёму памяти для операций обслуживания базы данных, чтобы ускорить обслуживание БД;
- даю оставшийся объём для кэширования данных (`effective_cache_size`), чтобы планировщик запросов знал, сколько данных может поместиться в кэш и выбирал оптимальные планы выполнения;
- увеличиваю временные буферы, чтобы временные таблицы и сортировки работали в памяти, а не на диске;
- увеличиваю буфер WAL, чтобы больше операций записи в WAL буферизовалось в памяти перед сбросом на диск;
- устанавливаю WAL writer на запись раз в 10 секунд, чтобы он реже просыпался и сбрасывал данные на диск;
- и устанавливаю, чтобы он сбрасывал данные на диск только при накоплении 4MB;
- группирую коммиты, чтобы уменьшить количество дисковых операций;
- устанавливаю `commit_siblings`, чтобы копилось 10 транзакций для групповой записи;
- устанавливаю `checkpoint` на раз в час, чтобы уменьшить нагрузку на диск;
- и растягиваю `checkpoint` почти на все время между чекпоинтами;
- включаю `enable_parallel_append`, чтобы UNION запросы делались в несколько потоков вместо одного;
- включаю `enable_parallel_hash`, чтобы JOIN с большими таблицами делался быстрее;
- включаю `enable_partitionwise_join`, чтобы JOIN делался между частями параллельно, если таблицы разделены на части; 
- включаю `enable_partitionwise_aggregate`, чтобы SUM/COUNT по частям таблиц считались параллельно;
- задаю новый `max_wal_size`, чтобы позволить накопиться большему объему WAL перед принудительным чекпоинтом;
- и задаю новый минимальный размер `min_wal_size`;
- устанавливаю агрессивные значения для моих характеристик для `max_parallel_workers_per_gather`, пытаясь выжать максимум;
- оставляю `max_worker_processes`, `max_parallel_workers ` стандартными, потому что для 2 CPU больше не нужно, а больше процессов будут только конкурировать за ресурсы;
- увеличиваю `max_parallel_maintenance_workers`, чтобы операции типа VACUUM и CREATE INDEX использовали несколько потоков и выполнялись быстрее;
- задаю `random_page_cost` минимальное значение, потому что на SSD случайные чтения почти такие же быстрые, как последовательные, и планировщик должен это знать;
- задаю новый `effective_io_concurrency` для асинхронного I/O, чтобы PostgreSQL мог одновременно выполнять больше операций ввода-вывода, что особенно важно для SSD;
- выключаю вообще всё, что не нужно для выполнения запросов: мониторинг, логирование, сбор статистики и т.д., потому что сбор статистики и логирование потребляют CPU и IO, которые нужны для обработки транзакций;

### В итоге мои настройки выглядят как:
```
fsync = off
synchronous_commit = off
full_page_writes = off
shared_buffers = 1GB
work_mem = 32MB
maintenance_work_mem = 512MB
effective_cache_size = 1GB
temp_buffers = 64MB
wal_buffers = 16MB
wal_writer_delay = 10000ms
wal_writer_flush_after = 4MB
commit_delay = 10000
commit_siblings = 10
checkpoint_timeout = 1h
checkpoint_timeout = 1h
checkpoint_completion_target = 0.99
enable_parallel_append = on
enable_parallel_hash = on
enable_partitionwise_join = on
enable_partitionwise_aggregate = on
max_wal_size = 4GB
min_wal_size = 1GB
max_worker_processes = 8                
max_parallel_workers_per_gather = 4     
max_parallel_maintenance_workers = 4    
max_parallel_workers = 8
parallel_leader_participation = off
random_page_cost = 1.0
effective_io_concurrency = 256
log_statement = 'none'
log_duration = off
log_lock_waits = off
log_temp_files = -1
log_checkpoints = off
log_connections = off
log_disconnections = off
log_line_prefix = ''
track_activities = off
track_counts = off
track_io_timing = off
track_functions = 'none'
```
## Перезапускаю Postgres, создаю тестовую таблицу с масштабом 10 (~1.5GB) и иду нагружать кластер через утилиту pgbench:
```
[root@postgresql ~]# systemctl restart postgresql-17
[root@postgresql ~]# sudo -u postgres createdb pgbench_test
[root@postgresql ~]# sudo -u postgres /usr/pgsql-17/bin/pgbench -i -s 10 pgbench_test
dropping old tables...
creating tables...
generating data (client-side)...
vacuuming...
creating primary keys...
done in 7.01 s (drop tables 1.13 s, create tables 0.17 s, client-side generate 3.98 s, vacuum 0.37 s, primary keys 1.36 s).
```
Результаты теста на 16 клиентов: 
```
[root@postgresql ~]# sudo -u postgres /usr/pgsql-17/bin/pgbench -c 16 -j 2 -T 60 -P 5 pgbench_test
pgbench (17.6)
starting vacuum...end.
progress: 5.0 s, 896.6 tps, lat 17.211 ms stddev 11.107, 0 failed
progress: 10.0 s, 913.4 tps, lat 17.481 ms stddev 12.234, 0 failed
progress: 15.0 s, 793.3 tps, lat 20.071 ms stddev 15.032, 0 failed
progress: 20.0 s, 870.2 tps, lat 18.201 ms stddev 12.402, 0 failed
progress: 25.0 s, 796.4 tps, lat 20.113 ms stddev 16.539, 0 failed
progress: 30.0 s, 715.6 tps, lat 22.292 ms stddev 20.112, 0 failed
progress: 35.0 s, 831.2 tps, lat 19.105 ms stddev 14.947, 0 failed
progress: 40.0 s, 933.6 tps, lat 17.079 ms stddev 12.577, 0 failed
progress: 45.0 s, 853.5 tps, lat 18.649 ms stddev 13.999, 0 failed
progress: 50.0 s, 973.0 tps, lat 16.358 ms stddev 12.099, 0 failed
progress: 55.0 s, 919.4 tps, lat 17.313 ms stddev 13.900, 0 failed
progress: 60.0 s, 1048.4 tps, lat 15.189 ms stddev 10.979, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 10
query mode: simple
number of clients: 16
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 52736
number of failed transactions: 0 (0.000%)
latency average = 18.094 ms
latency stddev = 13.981 ms
initial connection time = 130.014 ms
tps = 878.628888 (without initial connection time)
```
Результаты тестов на 32 клиента:
```
[root@postgresql ~]# sudo -u postgres /usr/pgsql-17/bin/pgbench -c 32 -j 2 -T 60 -P 5 -M prepared pgbench_test
pgbench (17.6)
starting vacuum...end.
progress: 5.0 s, 816.6 tps, lat 34.922 ms stddev 31.858, 0 failed
progress: 10.0 s, 1035.9 tps, lat 30.910 ms stddev 21.820, 0 failed
progress: 15.0 s, 981.5 tps, lat 32.491 ms stddev 22.954, 0 failed
progress: 20.0 s, 743.1 tps, lat 42.751 ms stddev 34.738, 0 failed
progress: 25.0 s, 741.3 tps, lat 42.892 ms stddev 40.798, 0 failed
progress: 30.0 s, 843.0 tps, lat 37.942 ms stddev 30.426, 0 failed
progress: 35.0 s, 833.4 tps, lat 38.116 ms stddev 35.272, 0 failed
progress: 40.0 s, 652.7 tps, lat 48.884 ms stddev 36.484, 0 failed
progress: 45.0 s, 713.2 tps, lat 44.677 ms stddev 34.288, 0 failed
progress: 50.0 s, 670.0 tps, lat 47.392 ms stddev 37.319, 0 failed
progress: 55.0 s, 607.2 tps, lat 52.259 ms stddev 46.211, 0 failed
progress: 60.0 s, 723.8 tps, lat 44.199 ms stddev 38.926, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 10
query mode: prepared
number of clients: 32
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 46841
number of failed transactions: 0 (0.000%)
latency average = 40.572 ms
latency stddev = 34.855 ms
initial connection time = 471.163 ms
tps = 782.732710 (without initial connection time)
```
Результаты тестов только read-only:
```
[root@postgresql ~]# sudo -u postgres /usr/pgsql-17/bin/pgbench -c 16 -j 2 -T 60 -P 5 -S pgbench_test
pgbench (17.6)
starting vacuum...end.
progress: 5.0 s, 5328.8 tps, lat 2.828 ms stddev 4.755, 0 failed
progress: 10.0 s, 6743.1 tps, lat 2.286 ms stddev 2.635, 0 failed
progress: 15.0 s, 5660.2 tps, lat 2.702 ms stddev 3.951, 0 failed
progress: 20.0 s, 4674.0 tps, lat 3.254 ms stddev 5.744, 0 failed
progress: 25.0 s, 5568.7 tps, lat 2.790 ms stddev 4.880, 0 failed
progress: 30.0 s, 6311.1 tps, lat 2.421 ms stddev 3.188, 0 failed
progress: 35.0 s, 6227.6 tps, lat 2.476 ms stddev 3.477, 0 failed
progress: 40.0 s, 6472.9 tps, lat 2.364 ms stddev 2.694, 0 failed
progress: 45.0 s, 6547.8 tps, lat 2.333 ms stddev 2.482, 0 failed
progress: 50.0 s, 5577.9 tps, lat 2.741 ms stddev 3.995, 0 failed
progress: 55.0 s, 5178.5 tps, lat 2.958 ms stddev 3.983, 0 failed
progress: 60.0 s, 5628.1 tps, lat 2.718 ms stddev 3.453, 0 failed
transaction type: <builtin: select only>
scaling factor: 10
query mode: simple
number of clients: 16
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 349621
number of failed transactions: 0 (0.000%)
latency average = 2.628 ms
latency stddev = 3.811 ms
initial connection time = 112.759 ms
tps = 5826.448782 (without initial connection time)
```
## Задача со звёздочкой:
Устанавливаю sysbench, sysbench-tpcc:
```

```
