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
29853 - 2 транзакция:
```
postgres=# \c new_db
You are now connected to database "new_db" as user "postgres".
new_db=# begin;
BEGIN
new_db=*# update users set name = 'Lera Murkina' where id = 2;
UPDATE 1
```
29744 - 1 транзакция:
```
new_db=*# select locktype, pid, relation::regclass, virtualxid as virtxid,
new_db-*#        transactionid as xid, mode, granted
new_db-*# from pg_locks ;
   locktype    |  pid  | relation | virtxid |   xid   |       mode       | granted
---------------+-------+----------+---------+---------+------------------+---------
 relation      | 29853 | users    |         |         | RowExclusiveLock | t
 virtualxid    | 29853 |          | 5/4     |         | ExclusiveLock    | t
 relation      | 29744 | users    |         |         | AccessShareLock  | t
 relation      | 29744 | pg_locks |         |         | AccessShareLock  | t
 virtualxid    | 29744 |          | 4/22    |         | ExclusiveLock    | t
 transactionid | 29853 |          |         | 2349839 | ExclusiveLock    | t
(6 rows)
new_db=*# select * from users;
 id |     name
----+---------------
  1 | Vera Fedotova
  2 | Lera Fedotova
(2 rows)
```
Блокировка отношений relation - т.к. я пыталась изменить таблицу второй транзакцией. 
Пробую сделать три апдейта в разных сеансах:
первый pid:
```
 pg_backend_pid
----------------
          30349
(1 row)
new_db=# begin;
BEGIN
new_db=*# update users set name = 'Vera Trudova' where id = 1;
UPDATE 1
```
второй pid:
```
 pg_backend_pid
----------------
          30337
(1 row)
new_db=# begin;
BEGIN
new_db=*# update users set name = 'Vera Nud' where id = 1;
```
третий pid:
```
 pg_backend_pid
----------------
          30333
new_db=# begin;
BEGIN
new_db=*# update users set name = 'Vera Mur' where id = 1;
(1 row)
```
Смотрим блокировку:
```
new_db=*# select locktype, pid, relation::regclass, virtualxid as virtxid,
new_db-*# transactionid as xid, mode, granted
new_db-*# from pg_locks ;
   locktype    |  pid  |         relation          | virtxid |   xid   |       mode       | granted
---------------+-------+---------------------------+---------+---------+------------------+---------
 relation      | 30349 | pg_stat_activity          |         |         | AccessShareLock  | t
 relation      | 30349 | pg_locks                  |         |         | AccessShareLock  | t
 relation      | 30349 | users                     |         |         | RowExclusiveLock | t
 virtualxid    | 30349 |                           | 8/3     |         | ExclusiveLock    | t
 transactionid | 30349 |                           |         | 2349840 | ExclusiveLock    | t
 relation      | 30349 | pg_authid_oid_index       |         |         | AccessShareLock  | t
 relation      | 30349 | pg_database_oid_index     |         |         | AccessShareLock  | t
 relation      | 30349 | pg_database_datname_index |         |         | AccessShareLock  | t
 relation      | 30349 | pg_database               |         |         | AccessShareLock  | t
 relation      | 30349 | pg_authid                 |         |         | AccessShareLock  | t
 relation      | 30349 | pg_authid_rolname_index   |         |         | AccessShareLock  | t

 relation      | 30337 | users                     |         |         | RowExclusiveLock | t
 virtualxid    | 30337 |                           | 7/5     |         | ExclusiveLock    | t
 tuple         | 30337 | users                     |         |         | ExclusiveLock    | t
 transactionid | 30337 |                           |         | 2349840 | ShareLock        | f
 transactionid | 30337 |                           |         | 2349841 | ExclusiveLock    | t

 relation      | 30333 | users                     |         |         | RowExclusiveLock | t
 virtualxid    | 30333 |                           | 6/3     |         | ExclusiveLock    | t
 tuple         | 30333 | users                     |         |         | ExclusiveLock    | f
 transactionid | 30333 |                           |         | 2349842 | ExclusiveLock    | t
(27 rows)
```
Лог:
```
postgres@otus-db-pg-vm-1:~$ tail -n 10 /var/log/postgresql/postgresql-16-main.log
2024-05-12 12:58:47.063 UTC [30333] postgres@new_db STATEMENT:  update users set name = 'Vera Mur' where id = 1;
2024-05-12 12:58:47.263 UTC [30333] postgres@new_db LOG:  process 30333 still waiting for ShareLock on transaction 2349841 after 200.122 ms
2024-05-12 12:58:47.263 UTC [30333] postgres@new_db DETAIL:  Process holding the lock: 30337. Wait queue: 30333.
2024-05-12 12:58:47.263 UTC [30333] postgres@new_db CONTEXT:  while rechecking updated tuple (0,4) in relation "users"
2024-05-12 12:58:47.263 UTC [30333] postgres@new_db STATEMENT:  update users set name = 'Vera Mur' where id = 1;
2024-05-12 12:58:55.983 UTC [30333] postgres@new_db LOG:  process 30333 acquired ShareLock on transaction 2349841 after 8920.577 ms
2024-05-12 12:58:55.983 UTC [30333] postgres@new_db CONTEXT:  while rechecking updated tuple (0,4) in relation "users"
2024-05-12 12:58:55.983 UTC [30333] postgres@new_db STATEMENT:  update users set name = 'Vera Mur' where id = 1;
2024-05-12 12:59:14.875 UTC [29682] LOG:  checkpoint starting: time
2024-05-12 12:59:14.990 UTC [29682] LOG:  checkpoint complete: wrote 2 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.102 s, sync=0.003 s, total=0.115 s; sync files=2, longest=0.002 s, average=0.002 s; distance=0 kB, estimate=2531 kB; lsn=0/6F4D3C78, redo lsn=0/6F4D3C38
```
ExclusiveLock - транзакция всегда удерживает исключительную  блокировку собственного номера.
RowExclusiveLock - блокировку в этом режиме получает любая команда, которая изменяет данные в таблице.
Тип tuple- Блокировка версии строки. Используется в некоторых случаях для установки приоритета среди нескольких транзакций, ожидающих блокировку одной и той же строки.

Мы видим, что каждая транзакция удерживает блокировку ExclusiveLock на своей виртуальной транзакции virtualxid
Первая транзакция получала блокировку с типом relation в режиме RowExclusiveLock, т.к. мы меняли данные в таблице.
А вторая и третья транзации заблокированы и есть строки с типом tuple и ExclusiveLock.

