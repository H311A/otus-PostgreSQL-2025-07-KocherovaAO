```
Окружение: Oracle Linux 8.10. 2 CPU 2 RAM 40 GB HDD.
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
- выключаю `full_page_writes`, чтобы убрать избыточную запись страниц;
- даю буферу максимально возможный объем для моих 2 RAM - 50%;
- задаю 512MB объёму памяти для операций обслуживания базы данных, чтобы ускорить обслуживание БД;
- даю оставшийся объём для кэширования данных (`effective_cache_size`);
- увеличиваю временные буферы;
- увеличиваю буфер WAL;
- устанавливаю WAL writer на запись раз в 10 секунд;
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
- оставляю `max_worker_processes`, `max_parallel_workers ` стандартными;
- увеличиваю `max_parallel_maintenance_workers`;
- задаю `random_page_cost` минимальное значение, чтобы планировщик думал, что random I/O почти бесплатен;
- задаю новый `effective_io_concurrency` для асинхронного I/O;
- выключаю вообще всё, что не нужно для выполнения запросов: мониторинг, логирование, логирование времени чекпоинтов, замер времени выполнения и всякая остальная статистика;

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
