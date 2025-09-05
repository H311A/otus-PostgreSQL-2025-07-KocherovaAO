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

[root@postgresql test_pg]# sudo -u postgres psql -t -A -c "SELECT pg_current_wal_lsn();" > before_lsn.txt
[root@postgresql test_pg]# cat before_lsn.txt
0/3271250
```
Запускаю нагрузку на 10 минут: 
```
[root@postgresql test_pg]# sudo -u postgres /usr/pgsql-17/bin/pgbench -T 600 test_db
pgbench (17.6)
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 208790
number of failed transactions: 0 (0.000%)
latency average = 2.874 ms
initial connection time = 13.878 ms
tps = 347.990686 (without initial connection time)
```
Замеряю объем WAL после теста и вычисляю разницу:
```
[root@postgresql test_pg]# sudo -u postgres psql -t -A -c "SELECT pg_current_wal_lsn();" > after_lsn.txt
[root@postgresql test_pg]# cat after_lsn.txt
0/AF4ABF8

[root@postgresql test_pg]# sudo -u postgres psql -c "SELECT pg_wal_lsn_diff('$(cat after_lsn.txt)', '$(cat before_lsn.txt)') AS wal_bytes;"
 wal_bytes
-----------
 130914728
(1 строка)
```
`130914728` - объем сгенерированных WAL-данных в байтах за время теста; `130914728 / 1024 / 1024 ≈ 124.8 MB`;  
За 10 минут (600 секунд) при `checkpoint_timeout = 30s` должно быть примерно: `600 / 30 = 20` контрольных точек;
Объем на одну точку: `130914728 / 20 ≈ 6 545 736 байт (≈6.24 MB)` на контрольную точку
