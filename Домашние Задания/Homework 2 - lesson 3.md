# Установка PostgreSQL 
```
Развернута ВМ Oracle Linux 8.10 в VMware.
```
## Установка Docker Engine:
```
[root@postgresql ~]# yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
[root@postgresql ~]# sed -i 's/centos/rhel/g' /etc/yum.repos.d/docker-ce.repo
[root@postgresql ~]# yum install -y --allowerasing docker-ce docker-ce-cli containerd.io
[root@postgresql ~]# systemctl start docker & systemctl enable docker
[root@postgresql ~]# usermod -aG docker $USER
[root@postgresql ~]# newgrp docker

[root@postgresql ~]# docker --version
Docker version 28.3.3, build 980b856
[root@postgresql ~]# docker info
Client: Docker Engine - Community
 Version:    28.3.3
 Context:    default
 Debug Mode: false
 Plugins:
  buildx: Docker Buildx (Docker Inc.)
    Version:  v0.26.1
    Path:     /usr/libexec/docker/cli-plugins/docker-buildx
  compose: Docker Compose (Docker Inc.)
    Version:  v2.39.1
    Path:     /usr/libexec/docker/cli-plugins/docker-compose
```
## Делаю каталог `/var/lib/postgres` и монтирую в него контейнер с PostgreSQL 15:
```
[root@postgresql ~]# mkdir /var/lib/postgres
[root@postgresql ~]# chown 1000:1000 /var/lib/postgres

[root@postgresql postgres]# docker run -d --name my-postgres-15 -e POSTGRES_PASSWORD=12345 -v /var/lib/postgres:/var/lib/postgresql/data -p 5432:5432 postgres:15
77b9635885dce3892c4216cfca0e941820417887a63f789f396e6c566c5c4c10
```
## Разворачиваю контейнер с клиентом Postgres и делаю таблицу с парой строк:
```
[root@postgresql ~]# docker run -it --rm --name pg-client --network host postgres:15 psql -h localhost -U postgres
Password for user postgres:
psql (15.13 (Debian 15.13-1.pgdg130+1))
Type "help" for help.

postgres=# CREATE TABLE employees (
  id SERIAL PRIMARY KEY,
  name VARCHAR(50),
  salary INT
);

INSERT INTO employees (name, salary) VALUES
  ('Иван', 50000),
  ('Мария', 60000);

SELECT * FROM employees;
CREATE TABLE
INSERT 0 2
 id | name  | salary
----+-------+--------
  1 | Иван  |  50000
  2 | Мария |  60000
(2 rows)
```
## Подключаюсь с Windows-машины к серверу через PGAdmin4:
![alt text](https://github.com/H311A/otus-PostgreSQL-2025-07-KocherovaAO/blob/main/Домашние%20Задания/Скриншоты/hmwk2screen1.png)

## Удаляю контейнер с сервером Postgres:
```
[root@postgresql postgres]# docker stop my-postgres-15
my-postgres-15
[root@postgresql postgres]# docker rm my-postgres-15
my-postgres-15
[root@postgresql postgres]# docker ps -a | grep my-postgres-15
[root@postgresql postgres]#
```
## Создаю контейнер с сервером Postgres с теми же данными: 
```
[root@postgresql postgres]# docker run -d --name my-postgres-15 -e POSTGRES_PASSWORD=12345 -v /var/lib/postgres:/var/lib/postgresql/data -p 5432:5432 postgres:15
e29f163b71f97a233bc8cb1e6b38a55273d45d2b7ae107c5bb9cc5a1c64437c4
[root@postgresql postgres]# docker ps | grep my-postgres-15
e29f163b71f9   postgres:15   "docker-entrypoint.s…"   8 seconds ago   Up 7 seconds   0.0.0.0:5432->5432/tcp, [::]:5432->5432/tcp   my-postgres-15
```
## Подключаюсь из клиентского контейнера к серверу: 
```
[root@postgresql ~]# docker run -it --rm --name pg-client --network host postgres:15 psql -h localhost -U postgres
Password for user postgres:
psql (15.13 (Debian 15.13-1.pgdg130+1))
Type "help" for help.

postgres=#
```
## Проверяю, что все данные остались на месте:
```
postgres=# SELECT * FROM employees;
 id | name  | salary
----+-------+--------
  1 | Иван  |  50000
  2 | Мария |  60000
(2 rows)
```
**Итог**: Данные на месте.
