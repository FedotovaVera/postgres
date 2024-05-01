Создаю БД и схему:
```
postgres=# create database testdb;
CREATE DATABASE
postgres=# create schema testnm;
CREATE SCHEMA
```
Создаю таблицу и добавляю значение:
```
testdb=# create table testnm.t1 (c1 integer);
CREATE TABLE
testdb=# insert into testnm.t1(c1) values (1);
INSERT 0 1
```
Создаю роль и раздаю права:
```
testdb=# create role readonly;
CREATE ROLE
testdb=# grant connect on DATABASE testdb TO readonly;
GRANT
testdb=# grant usage on SCHEMA testnm to readonly;
GRANT
testdb=# grant SELECT on all TABLEs in SCHEMA testnm TO readonly;
GRANT
```
Создаю пользователя и даю ему роль:
```
testdb=# CREATE USER testread with password 'test123';
CREATE ROLE
testdb=# grant readonly TO testread;
GRANT ROLE
```
Зайшла под пользователем, пробую просмотреть таблицу и вижу ошибку:
```
testdb=> select * from t1;
ERROR:  relation "t1" does not exist
LINE 1: select * from t1;
```
С указанием схемы получилось:
```
testdb=> select * from testnm.t1;
 c1
----
  1
(1 row)
```
Возвращаюсь под пользователем postgres и удаляю таблицу:
```
testdb=# DROP TABLE t1;
ERROR:  table "t1" does not exist
testdb=# DROP TABLE testnm.t1;
DROP TABLE
```
Создаю таблицу заново и добавляю значение:
```
testdb=# create table testnm.t1 (c1 integer);
CREATE TABLE
testdb=# insert into testnm.t1(c1) values (1);
INSERT 0 1
```
Возвращаюсь с тестовому пользователю и пытаюсь просмотреть таблицу:
```
testdb=> select * from testnm.t1;
ERROR:  permission denied for table t1
```
Посмотрела подсказку. Я дала доступ только для существующих на тот момент таблиц, а t1 пересоздалась. Поэтому меняю дефолтные привелегии: 
```
postgres=# ALTER default privileges in SCHEMA testnm grant SELECT on TABLES to readonly;
ALTER DEFAULT PRIVILEGES
```
Но снова не получается, потому что таблица была создана до вышеуказанных изменений:
```
testdb=> select * from testnm.t1;
ERROR:  permission denied for table t1
```
Поэтому я возвращаюсь и заново раздаю права и всё получается:
```
postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
testdb=# grant SELECT on all TABLEs in SCHEMA testnm TO readonly;
GRANT
testdb=# \q
dmitrydergunov95@otus-db-pg-vm-1:~$ psql -h 127.0.0.1 -U testread -d testdb -W
Password:
psql (16.2 (Ubuntu 16.2-1.pgdg22.04+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, compression: off)
Type "help" for help.

testdb=>  select * from testnm.t1;
 c1
----
  1
(1 row)
```
Пробую создать таблицу:
```
testdb=> create table t2(c1 integer);
ERROR:  permission denied for schema public
LINE 1: create table t2(c1 integer);
                     ^
testdb=> create table testnm.t2(c1 integer);
ERROR:  permission denied for schema testnm
LINE 1: create table testnm.t2(c1 integer);
```
Но права на создание и изменение я не давала. 
Поэтому я поменяла у поли readonly привелегии, хотя логично исходя из названия создать другую роль (тут у меня вопрос: у одного пользователя может быть несколько ролей одновременно?)
```
testdb=# GRANT ALL PRIVILEGES ON testnm TO readonly;
ERROR:  relation "testnm" does not exist
testdb=# GRANT ALL ON SCHEMA testnm TO readonly;
GRANT
testdb=# \q
dmitrydergunov95@otus-db-pg-vm-1:~$ psql -h 127.0.0.1 -U testread -d testdb -W
Password:
psql (16.2 (Ubuntu 16.2-1.pgdg22.04+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, compression: off)
Type "help" for help.

testdb=> create table t3(c1 integer); insert into t2 values (2);
ERROR:  permission denied for schema public
LINE 1: create table t3(c1 integer);
                     ^
ERROR:  relation "t2" does not exist
LINE 1: insert into t2 values (2);
                    ^
testdb=> create table testnm.t3(c1 integer);
CREATE TABLE

testdb=> insert into testnm.t3 values (2);
INSERT 0 1
```
теперь т.к. у роли есть все привелегии на схему testnm , то таблица создалась
