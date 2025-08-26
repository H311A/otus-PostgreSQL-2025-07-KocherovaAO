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
- задаю новый `max_wal_size`, чтобы позволить накопиться большему объему WAL перед принудительным чекпоинтом;
- и задаю новый минимальный размер `min_wal_size`;

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
max_wal_size = 4GB
min_wal_size = 1GB
```
