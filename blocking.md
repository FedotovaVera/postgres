Меняю настройки, перезапускаю кластер и проверяю, что настройки применились:
```
postgres=# alter system set deadlock_timeout = 200;
 set log_lock_waits = on;  ALTER SYSTEM
postgres=# alter system set log_lock_waits = on;
ALTER SYSTEM
postgres=# alter system set deadlock_timeout = 200;
ALTER SYSTEM
postgres=# alter system set log_lock_waits = on;
ALTER SYSTEM
postgres=# \q
postgres@otus-db-pg-vm-1:~$ exit
logout
dmitrydergunov95@otus-db-pg-vm-1:~$ sudo pg_ctlcluster 16 main restart
dmitrydergunov95@otus-db-pg-vm-1:~$ sudo -i -u postgres
postgres@otus-db-pg-vm-1:~$ psql
psql (16.3 (Ubuntu 16.3-1.pgdg22.04+1))
Type "help" for help.

postgres=# show deadlock_timeout;
 deadlock_timeout
------------------
 200ms
(1 row)

postgres=# show log_lock_waits;
 log_lock_waits
----------------
 on
(1 row)
```
Создаю новую бд, создаю в ней таблицу и вношу данные:
```
postgres=# create database new_db;
CREATE DATABASE
postgres=# \c new_db
You are now connected to database "new_db" as user "postgres".
new_db=# create table users (id int, "name" text);
CREATE TABLE
new_db=# insert into users values (1, 'Vera Fedotova'), (2, 'Lera Fedotova');
INSERT 0 2
```
Открыла второй терминал.
```
PID первого терминала:
new_db=*# SELECT pg_backend_pid();
 pg_backend_pid
----------------
          29744
(1 row)
```
PID второго терминала:
```
new_db=*# SELECT pg_backend_pid();
 pg_backend_pid
----------------
          29853
(1 row)
```
|29744 - 1 транзакция|29853 - 2 транзакция|
|-|--------|
|Длинная запись в первом столбце|```new_db=#  <br>  begin; <br>   BEGIN new_db=*# update users set name = 'Lera Murkina' where id = 2;    UPDATE 1```|
