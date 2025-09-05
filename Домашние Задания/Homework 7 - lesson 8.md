```
Окружение: Oracle Linux 8.10.
```
# Работа с журналами
Редактирую конфигурацию `/var/lib/pgsql/17/data/postgresql.conf`;  
Настраиваю выполнение контрольной точки раз в 30 секунд: `checkpoint_timeout = 30s`;  
Перезагружаю сервер, чтобы применить изменения: `sudo systemctl restart postgresql-17`.  

#### Cоздаю новую базу `test_db`, измеряю до теста текущую позицию WAL:
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
#### Запускаю нагрузку на 10 минут: 
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
#### Замеряю объем WAL после теста и вычисляю разницу:
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
Объем на одну точку: `130914728 / 20 ≈ 6 545 736 байт (≈6.24 MB)` на контрольную точку.

#### Проверяю статистику контрольных точек из логов:
```
2025-09-05 11:08:25.907 MSK [12348] СООБЩЕНИЕ:  начата контрольная точка: time
2025-09-05 11:08:29.433 MSK [12348] СООБЩЕНИЕ:  контрольная точка завершена: записано буферов: 38 (0.2%); добавлено файлов WAL 0, удалено: 0, переработано: 0; запись=3.517 сек., синхр.=0.002 сек., всего=3.526 сек.; синхронизировано_файлов=11, самая_долгая_синхр.=0.001 сек., средняя=0.001 сек.; расстояние=197 kB, ожидалось=197 kB; lsn=0/1567EE0, lsn redo=0/1567E88
2025-09-05 11:18:25.534 MSK [12348] СООБЩЕНИЕ:  начата контрольная точка: time
2025-09-05 11:22:47.590 MSK [12348] СООБЩЕНИЕ:  контрольная точка завершена: записано буферов: 2610 (15.9%); добавлено файлов WAL 0, удалено: 0, переработано: 2; запись=262.033 сек., синхр.=0.006 сек., всего=262.056 сек.; синхронизировано_файлов=328, самая_долгая_синхр.=0.002 сек., средняя=0.001 сек.; расстояние=29732 kB, ожидалось=29732 kB; lsn=0/32712E0, lsn redo=0/3271250
2025-09-05 11:23:25.616 MSK [12348] СООБЩЕНИЕ:  начата контрольная точка: time
2025-09-05 11:26:47.742 MSK [12348] СООБЩЕНИЕ:  контрольная точка завершена: записано буферов: 2015 (12.3%); добавлено файлов WAL 0, удалено: 0, переработано: 1; запись=202.109 сек., синхр.=0.006 сек., всего=202.127 сек.; синхронизировано_файлов=19, самая_долгая_синхр.=0.003 сек., средняя=0.001 сек.; расстояние=18643 kB, ожидалось=28624 kB; lsn=0/6EEB948, lsn redo=0/44A60A0
2025-09-05 11:28:25.808 MSK [12348] СООБЩЕНИЕ:  начата контрольная точка: time
2025-09-05 11:32:52.493 MSK [12348] СООБЩЕНИЕ:  контрольная точка завершена: записано буферов: 2678 (16.3%); добавлено файлов WAL 0, удалено: 0, переработано: 3; запись=266.647 сек., синхр.=0.019 сек., всего=266.686 сек.; синхронизировано_файлов=18, самая_долгая_синхр.=0.016 сек., средняя=0.002 сек.; расстояние=56866 kB, ожидалось=56866 kB; lsn=0/AE28B60, lsn redo=0/7C2E8D8
2025-09-05 11:33:25.514 MSK [12348] СООБЩЕНИЕ:  начата контрольная точка: time
2025-09-05 11:37:36.621 MSK [12348] СООБЩЕНИЕ:  контрольная точка завершена: записано буферов: 2508 (15.3%); добавлено файлов WAL 0, удалено: 0, переработано: 3; запись=251.093 сек., синхр.=0.003 сек., всего=251.107 сек.; синхронизировано_файлов=9, самая_долгая_синхр.=0.003 сек., средняя=0.001 сек.; расстояние=52288 kB, ожидалось=56408 kB; lsn=0/AF4AB48, lsn redo=0/AF3E960
```
Контрольные точки не выполнялись по расписанию раз в 30 секунд. Из-за интенсивной нагрузки `pgbench` генерировалось так много WAL-данных, что контрольные точки срабатывали досрочно при достижении `max_wal_size`, а не по истечении 30 секунд. Дисковая подсистема не успевала обрабатывать такой объем данных, что приводило к длительному выполнению контрольных точек.

## Tps в синхронном/асинхронном режиме: 
#### Синхронный режим:
```
[root@postgresql log]# sudo -u postgres /usr/pgsql-17/bin/pgbench -T 60 test_db
pgbench (17.6)
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 23592
number of failed transactions: 0 (0.000%)
latency average = 2.543 ms
initial connection time = 13.315 ms
tps = 393.272919 (without initial connection time)
```
#### Асинхронный режим:
```
[root@postgresql log]# sudo -u postgres psql -c "ALTER SYSTEM SET synchronous_commit TO off;"
ALTER SYSTEM
[root@postgresql log]# systemctl restart postgresql-17
[root@postgresql log]# sudo -u postgres /usr/pgsql-17/bin/pgbench -T 60 test_db
pgbench (17.6)
starting vacuum...end.
```
