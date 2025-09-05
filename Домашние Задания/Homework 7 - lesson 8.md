```
Окружение: Oracle Linux 8.10.
```
Редактирую конфигурацию `/var/lib/pgsql/17/data/postgresql.conf`;  
Настраиваю выполнение контрольной точки раз в 30 секунд: `checkpoint_timeout = 30s`;  
Перезагружаю сервер, чтобы применить изменения: `sudo systemctl restart postgresql-17`.  
