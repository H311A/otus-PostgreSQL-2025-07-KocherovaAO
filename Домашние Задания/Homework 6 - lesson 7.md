# Настройка autovacuum с учетом особеностей производительности.
```
Окружение: Oracle Linux 8.10, 2 CPU, 4 RAM, 40 GB SSD (не буду пересоздавать новую машину с 10 SSD, использую текущую с предыдущих занятий).
```
Устанавливаю чистый PostgeSQL 17 с дефолтными настройками, создаю новую БД для тестов:
```
[root@postgresql ~]# yum remove postgresql17*
Удален:
  postgresql17-17.6-1PGDG.rhel8.x86_64                postgresql17-contrib-17.6-1PGDG.rhel8.x86_64         postgresql17-libs-17.6-1PGDG.rhel8.x86_64
  postgresql17-server-17.6-1PGDG.rhel8.x86_64
[root@postgresql pgsql]# rm -rf /var/lib/pgsql/17
[root@postgresql pgsql]# yum install postgresql17*
Установлен:
  postgresql17-17.6-1PGDG.rhel8.x86_64                                       postgresql17-libs-17.6-1PGDG.rhel8.x86_64
  postgresql17-server-17.6-1PGDG.rhel8.x86_64                                postgresql17-contrib-17.6-1PGDG.rhel8.x86_64


```
