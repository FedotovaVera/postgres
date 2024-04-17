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

```


