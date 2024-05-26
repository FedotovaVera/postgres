Создаю БД, схему, таблцу, заполняю данными:
```
postgres=# CREATE DATABASE otus;
CREATE DATABASE
postgres=# create schema otus;
CREATE SCHEMA
postgres=# create table otus.text (name text);
CREATE TABLE
postgres=# insert into otus.text (name) select md5(random()::text) from generate_series(1,100);
INSERT 0 100
postgres=# select * from otus.text;
```
Создаю директорию:
```
dmitrydergunov95@backup:~$ cd /tmp/
dmitrydergunov95@backup:/tmp$ mkdir bckp
dmitrydergunov95@backup:/tmp$ sudo -u postgres psql
psql (15.7 (Ubuntu 15.7-1.pgdg22.04+1))
Type "help" for help.

postgres=# \c otus
You are now connected to database "otus" as user "postgres".
otus=# \copy otus.text to '/tmp/bckp/backuptext.sql';
/tmp/bckp/backuptext.sql: Permission denied
```
Ошибочка вышла. Меняю права на папку:
```
dmitrydergunov95@backup:/tmp$ sudo chown -R postgres /tmp/bckp/
```
Логический бэкап:
```
postgres=# \copy otus.text to '/tmp/bckp/backuptext.sql';
COPY 100
```
Проверяю:
```
dmitrydergunov95@backup:/tmp$ cd bckp/
dmitrydergunov95@backup:/tmp/bckp$ ls -la
total 12
drwxrwxr-x  2 postgres dmitrydergunov95 4096 May 26 20:39 .
drwxrwxrwt 13 root     root             4096 May 26 20:31 ..
-rw-rw-r--  1 postgres postgres         3300 May 26 20:39 backuptext.sql
```
Восстанавливаю данные во вторую таблицу:
```
postgres=# \copy otus.text2 from '/tmp/bckp/backuptext.sql';
COPY 200
postgres=# select * from otus.text2;
```
