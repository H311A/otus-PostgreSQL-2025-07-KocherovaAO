```
Окружение: Oracle Linux 8.10.
```
Редактирую конфигурацию `/var/lib/pgsql/17/data/postgresql.conf`;  
Настраиваю выполнение контрольной точки раз в 30 секунд: `checkpoint_timeout = 30s`;  
Перезагружаю сервер, чтобы применить изменения: `sudo systemctl restart postgresql-17`.  

Дальше создаю новую базу `test_db`, измеряю до теста текущую позицию WAL:
```
[root@postgresql ~]# sudo -u postgres createdb test_db
[root@postgresql ~]# sudo -u postgres /usr/pgsql-17/bin/pgbench -i test_db
dropping old tables...
creating tables...
generating data (client-side)...
vacuuming...
creating primary keys...
done in 0.70 s (drop tables 0.02 s, create tables 0.01 s, client-side generate 0.38 s, vacuum 0.15 s, primary keys 0.13 s).

[root@postgresql test_pg]# sudo -u postgres psql -c "SELECT pg_current_wal_lsn();" > before_lsn.txt
[root@postgresql test_pg]# cat before_lsn.txt
 pg_current_wal_lsn
--------------------
 0/3271250
(1 строка)
```
Запускаю нагрузку на 10 минут: 
```
[root@postgresql test_pg]# sudo -u postgres /usr/pgsql-17/bin/pgbench -T 600 test_db
pgbench (17.6)
starting vacuum...end.

```
