# Установка и настройка PostgreSQL
```
Окружение: Oracle Linux 8.10. PotsgreSQL 17.

[root@postgresql ~]# sudo -u postgres psql -c "SELECT version();"
                                                 version
---------------------------------------------------------------------------------------------------------
 PostgreSQL 17.5 on x86_64-pc-linux-gnu, compiled by gcc (GCC) 8.5.0 20210514 (Red Hat 8.5.0-26), 64-bit
(1 строка)
```
Проверьте что кластер запущен через sudo -u postgres pg_lsclusters. Так как у меня не Ubuntu, проверяю через pg_ctl.
```
[root@postgresql ~]# sudo -u postgres /usr/pgsql-17/bin/pg_ctl -D /var/lib/pgsql/17/data/ status
pg_ctl: сервер работает (PID: 3170)
/usr/pgsql-17/bin/postgres "-D" "/var/lib/pgsql/17/data/"
```
