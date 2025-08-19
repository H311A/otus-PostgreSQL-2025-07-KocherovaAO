# Установка и настройка PostgreSQL
```
Окружение: VMware. Oracle Linux 8.10. LVM + ext4. PotsgreSQL 17.

[root@postgresql ~]# sudo -u postgres psql -c "SELECT version();"
                                                 version
---------------------------------------------------------------------------------------------------------
 PostgreSQL 17.5 on x86_64-pc-linux-gnu, compiled by gcc (GCC) 8.5.0 20210514 (Red Hat 8.5.0-26), 64-bit
(1 строка)
```
Так как у меня не Ubuntu, проверяю через pg_ctl или systemctl:
```
[root@postgresql ~]# sudo -u postgres /usr/pgsql-17/bin/pg_ctl -D /var/lib/pgsql/17/data/ status
pg_ctl: сервер работает (PID: 3170)
/usr/pgsql-17/bin/postgres "-D" "/var/lib/pgsql/17/data/"

[root@postgresql ~]# systemctl status postgresql-17
● postgresql-17.service - PostgreSQL 17 database server
   Loaded: loaded (/usr/lib/systemd/system/postgresql-17.service; enabled; vendor preset: disable>
   Active: active (running) since Tue 2025-08-19 11:54:48 MSK; 1s ago
     Docs: https://www.postgresql.org/docs/17/static/
  Process: 3165 ExecStartPre=/usr/pgsql-17/bin/postgresql-17-check-db-dir ${PGDATA} (code=exited,>
 Main PID: 3170 (postgres)
    Tasks: 7 (limit: 10359)
   Memory: 52.7M
   CGroup: /system.slice/postgresql-17.service
           ├─3170 /usr/pgsql-17/bin/postgres -D /var/lib/pgsql/17/data/
           ├─3172 postgres: logger
           ├─3173 postgres: checkpointer
           ├─3174 postgres: background writer
           ├─3176 postgres: walwriter
           ├─3177 postgres: autovacuum launcher
           └─3178 postgres: logical replication launcher

авг 19 11:54:48 postgresql systemd[1]: Starting PostgreSQL 17 database server...
авг 19 11:54:48 postgresql postgres[3170]: 2025-08-19 11:54:48.570 MSK [3170] СООБЩЕНИЕ:  передач>
авг 19 11:54:48 postgresql postgres[3170]: 2025-08-19 11:54:48.570 MSK [3170] ПОДСКАЗКА:  В дальн>
авг 19 11:54:48 postgresql systemd[1]: Started PostgreSQL 17 database server.
lines 1-21/21 (END)...skipping...
```
Создаю таблицу с произвольными данными и останавливаю postgres:
```
[root@postgresql ~]# sudo -u postgres psql
psql (17.5)
Введите "help", чтобы получить справку.

postgres=# CREATE TABLE test(c1 text);
CREATE TABLE
postgres=# INSERT INTO test VALUES('1');
INSERT 0 1
postgres=# \q

[root@postgresql ~]# sudo systemctl stop postgresql-17
[root@postgresql ~]# systemctl status postgresql-17
● postgresql-17.service - PostgreSQL 17 database server
   Loaded: loaded (/usr/lib/systemd/system/postgresql-17.service; enabled; vendor preset: disabled)
   Active: inactive (dead) since Tue 2025-08-19 12:25:27 MSK; 6s ago
     Docs: https://www.postgresql.org/docs/17/static/
  Process: 1183 ExecStart=/usr/pgsql-17/bin/postgres -D ${PGDATA} (code=exited, status=0/SUCCESS)
  Process: 1162 ExecStartPre=/usr/pgsql-17/bin/postgresql-17-check-db-dir ${PGDATA} (code=exited, status=0/SUCCESS)
 Main PID: 1183 (code=exited, status=0/SUCCESS)

авг 19 12:25:01 postgresql systemd[1]: Starting PostgreSQL 17 database server...
авг 19 12:25:03 postgresql postgres[1183]: 2025-08-19 12:25:03.887 MSK [1183] СООБЩЕНИЕ:  передача вывода в протокол процессу сбора протоколов
авг 19 12:25:03 postgresql postgres[1183]: 2025-08-19 12:25:03.887 MSK [1183] ПОДСКАЗКА:  В дальнейшем протоколы будут выводиться в каталог "log".
авг 19 12:25:04 postgresql systemd[1]: Started PostgreSQL 17 database server.
авг 19 12:25:27 postgresql systemd[1]: Stopping PostgreSQL 17 database server...
авг 19 12:25:27 postgresql systemd[1]: postgresql-17.service: Killing process 1312 (postgres) with signal SIGKILL.
авг 19 12:25:27 postgresql systemd[1]: postgresql-17.service: Succeeded.
авг 19 12:25:27 postgresql systemd[1]: Stopped PostgreSQL 17 database server.
```
Добавляю на ВМ новый диск 10гб:

![новый диск](https://github.com/H311A/otus-PostgreSQL-2025-07-KocherovaAO/raw/main/Домашние%20Задания/Скриншоты/hmwk3screen1.png)
```
[root@postgresql ~]# lsblk
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda           8:0    0   40G  0 disk
├─sda1        8:1    0   39G  0 part
│ ├─ol-root 252:0    0   37G  0 lvm  /
│ └─ol-swap 252:1    0  2,1G  0 lvm  [SWAP]
└─sda2        8:2    0    1G  0 part /boot
sdb           8:16   0   10G  0 disk
sr0          11:0    1 13,2G  0 rom
```
