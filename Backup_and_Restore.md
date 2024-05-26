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
Используя утилиту pg_dump создадим бэкап в кастомном сжатом формате двух таблиц. чет у меня не видел таблички, в итоге я создала новые и заполнила их в схеме паблик.
```
dmitrydergunov95@backup:~$ sudo -u postgres pg_dump --port 5432 -d otus --compress=9 --table=public.text --table=public.text2 -Fc > /tmp/bckp/backup_2_tables.gz
dmitrydergunov95@backup:~$ pg_restore --list /tmp/bckp/backup_2_tables.gz
;
; Archive created at 2024-05-26 22:04:41 UTC
;     dbname: otus
;     TOC Entries: 8
;     Compression: 9
;     Dump Version: 1.14-0
;     Format: CUSTOM
;     Integer: 4 bytes
;     Offset: 8 bytes
;     Dumped from database version: 15.7 (Ubuntu 15.7-1.pgdg22.04+1)
;     Dumped by pg_dump version: 15.7 (Ubuntu 15.7-1.pgdg22.04+1)
;
;
; Selected TOC Entries:
;
215; 1259 16409 TABLE public text postgres
216; 1259 16414 TABLE public text2 postgres
3324; 0 16409 TABLE DATA public text postgres
3325; 0 16414 TABLE DATA public text2 postgres
```
Восстанавливаем одну из таблиц в новую БД:
```
postgres=# CREATE DATABASE restore;
CREATE DATABASE
dmitrydergunov95@backup:~$ sudo -u postgres pg_restore -d restore --table=text2 /tmp/bckp/backup_2_tables.gz
```
```
restore=# \d
         List of relations
 Schema | Name  | Type  |  Owner
--------+-------+-------+----------
 public | text2 | table | postgres
(1 row)
restore=# select count(1) from text2;
 count
-------
   100
(1 row)
```
