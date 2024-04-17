Создаю docker--сеть:
```
vera@otus-db-pg-vm-1:~$ sudo docker network create pg-net
```
Создаю контейнер с сервером:
```
vera@otus-db-pg-vm-1:~$ sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15
Unable to find image 'postgres:15' locally
15: Pulling from library/postgres
13808c22b207: Pull complete
d5f414cc6320: Pull complete
8d525e3e821d: Pull complete
b7f5a1ca5cc7: Pull complete
99fd5918b0ae: Pull complete
d45a1a3e63ad: Pull complete
e824c09e8690: Pull complete
b6efafa9ff61: Pull complete
e723a6ea1ef5: Pull complete
d58aed2eceb7: Pull complete
f7a7c0a35077: Pull complete
b61a2dcedb79: Pull complete
964cea5278d4: Pull complete
5836f6601d63: Pull complete
Digest: sha256:52495257b64779f90b46061a88d71237176613a9fb241d90ad15a643b0be6236
Status: Downloaded newer image for postgres:15
b7119acc9f7b80da20fe64cc012f194d6f2fadabf7cf70ba4596143802c1312a
docker: Error response from daemon: driver failed programming external connectivity on endpoint pg-server (dba2cc97ae8bf57f9522c716e513b83ff01940bc681e7fa46ee06c2a61710d52): Error starting userland proxy: listen tcp4 0.0.0.0:5432: bind: address already in use.
```
Кажется порт 5432 занят, попробую 5433:
```
vera@otus-db-pg-vm-1:~$ sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5433:5433 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15
docker: Error response from daemon: Conflict. The container name "/pg-server" is already in use by container "b7119acc9f7b80da20fe64cc012f194d6f2fadabf7cf70ba4596143802c1312a". You have to remove (or rename) that container to be able to reuse that name.
See 'docker run --help'.
```
Понимаю, что нужно переименовать или удалить контейнер "/pg-server". ПОпробую удалить:
```
vera@otus-db-pg-vm-1:~$ docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED         STATUS    PORTS     NAMES
b7119acc9f7b   postgres:15   "docker-entrypoint.s…"   8 minutes ago   Created             pg-server
vera@otus-db-pg-vm-1:~$ docker rm b7119acc9f7b
b7119acc9f7b
vera@otus-db-pg-vm-1:~$ docker ps -a
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
```
Снова создаю контейнер:
```
sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5433:5433 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15
```
Подключаюсь к базе с контейнером:
```
otus-db-pg-vm-1:~$ sudo docker run -it --rm --network pg-net --name pg-client postgres:15 psql -h pg-server -U postgres
Password for user postgres:
psql (15.6 (Debian 15.6-1.pgdg120+2))
Type "help" for help.
```
Создаю базу данных otus и создаю тестовую табличку:
```
postgres=# create database otus;
CREATE DATABASE
postgres=# create table test (id int);
CREATE TABLE
```
Добавляю пару строк:
```
postgres=# insert into test(id) values (1);
INSERT 0 1
postgres=# insert into test(id) values (2);
INSERT 0 1
postgres=# insert into test(id) values (3);
INSERT 0 1
postgres=# insert into test(id) values (4);
INSERT 0 1
postgres=# insert into test(id) values (5);
INSERT 0 1
```
Проверяю контейнеры:
```
vera@otus-db-pg-vm-1:~$ docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS          PORTS                                                 NAMES
8475f5d5a0d0   postgres:15   "docker-entrypoint.s…"   20 minutes ago   Up 20 minutes   5432/tcp, 0.0.0.0:5433->5433/tcp, :::5433->5433/tcp   pg-server
```
Останавливаю и удаляю контейнер. Проверяю, что удалился:
```
vera@otus-db-pg-vm-1:~$ docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS          PORTS                                                 NAMES
8475f5d5a0d0   postgres:15   "docker-entrypoint.s…"   20 minutes ago   Up 20 minutes   5432/tcp, 0.0.0.0:5433->5433/tcp, :::5433->5433/tcp   pg-server
vera@otus-db-pg-vm-1:~$ docker stop 8475f5d5a0d0
8475f5d5a0d0
vera@otus-db-pg-vm-1:~$ docker rm 8475f5d5a0d0
8475f5d5a0d0
vera@otus-db-pg-vm-1:~$ docker ps -a
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
```
Создаю заново и пытаюсь подключиться:
```
vera@otus-db-pg-vm-1:~$ sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5433:5433 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15
f446317cfdeb019618d9e63afb557b845bb95ef3821cd05570d6c620e94fc3a5
vera@otus-db-pg-vm-1:~$ psql -h localhost -U postgres -d postgres
Password for user postgres:
psql: error: connection to server at "localhost" (::1), port 5432 failed: FATAL:  password authentication failed for user "postgres"
connection to server at "localhost" (::1), port 5432 failed: FATAL:  password authentication failed for user "postgres"
```
Пробую подключиться к порту 5433:
```
@otus-db-pg-vm-1:~$ psql -h localhost -p 5433 -U postgres -d postgres
Password for user postgres:
psql (16.2 (Ubuntu 16.2-1.pgdg22.04+1), server 15.6 (Debian 15.6-1.pgdg120+2))
Type "help" for help.
```
Проверяю, что данные на месте:
```
                                                      List of databases
   Name    |  Owner   | Encoding | Locale Provider |  Collate   |   Ctype    | ICU Locale | ICU Rules |   Access privileges
-----------+----------+----------+-----------------+------------+------------+------------+-----------+-----------------------
 otus      | postgres | UTF8     | libc            | en_US.utf8 | en_US.utf8 |            |           |
 postgres  | postgres | UTF8     | libc            | en_US.utf8 | en_US.utf8 |            |           |
 template0 | postgres | UTF8     | libc            | en_US.utf8 | en_US.utf8 |            |           | =c/postgres          +
           |          |          |                 |            |            |            |           | postgres=CTc/postgres
 template1 | postgres | UTF8     | libc            | en_US.utf8 | en_US.utf8 |            |           | =c/postgres          +
           |          |          |                 |            |            |            |           | postgres=CTc/postgres
(4 rows)
```
Проверяю, что данные остались:
```
postgres=# \c otus
psql (16.2 (Ubuntu 16.2-1.pgdg22.04+1), server 15.6 (Debian 15.6-1.pgdg120+2))
You are now connected to database "otus" as user "postgres".
otus=# select * from test;
ERROR:  relation "test" does not exist
LINE 1: select * from test;
                      ^
otus=# \c postgres
psql (16.2 (Ubuntu 16.2-1.pgdg22.04+1), server 15.6 (Debian 15.6-1.pgdg120+2))
You are now connected to database "postgres" as user "postgres".
postgres=# select * from test
postgres-# ;
 id
----
  1
  2
  3
  4
  5
(5 rows)
```
