#### 1. Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд. Воспроизведите ситуацию, при которой в журнале появятся такие сообщения. ####
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
#### 2. Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны. Пришлите список блокировок и объясните, что значит каждая. ####
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

Для завершения ожидания, я сначала завершила первую транзацию, затем вторую и следом третью. 

#### 3. Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений? ####

Что бы выпонить взаимоблокировку, я пробую выполнить команды update по очереди в первой транзации, второй, третьей, первой, второй, третьей. На втором круге и второй транзации получаю ошибку:
```
ERROR:  deadlock detected
DETAIL:  Process 32092 waits for ExclusiveLock on tuple (0,12) of relation 16419 of database 16413; blocked by process 32105.
Process 32105 waits for ShareLock on transaction 2349849; blocked by process 32080.
Process 32080 waits for ShareLock on transaction 2349850; blocked by process 32092.
HINT:  See server log for query details.
```
Смотрю лог:
```
postgres@otus-db-pg-vm-1:~$ tail -n 30 /var/log/postgresql/postgresql-16-main.log
2024-05-12 14:35:23.031 UTC [29744] postgres@new_db FATAL:  terminating connection due to administrator command
2024-05-12 14:37:50.176 UTC [29682] LOG:  checkpoint starting: time
2024-05-12 14:37:50.288 UTC [29682] LOG:  checkpoint complete: wrote 2 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.103 s, sync=0.003 s, total=0.113 s; sync files=2, longest=0.002 s, average=0.002 s; distance=0 kB, estimate=1661 kB; lsn=0/6F4D46E0, redo lsn=0/6F4D46A8
2024-05-12 14:38:20.295 UTC [29682] LOG:  checkpoint starting: time
2024-05-12 14:38:20.418 UTC [29682] LOG:  checkpoint complete: wrote 2 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.105 s, sync=0.005 s, total=0.123 s; sync files=2, longest=0.004 s, average=0.003 s; distance=1 kB, estimate=1495 kB; lsn=0/6F4D4B20, redo lsn=0/6F4D4AE0
2024-05-12 14:38:27.020 UTC [32105] postgres@new_db LOG:  process 32105 still waiting for ShareLock on transaction 2349849 after 200.062 ms
2024-05-12 14:38:27.020 UTC [32105] postgres@new_db DETAIL:  Process holding the lock: 32080. Wait queue: 32105.
2024-05-12 14:38:27.020 UTC [32105] postgres@new_db CONTEXT:  while updating tuple (0,12) in relation "users"
2024-05-12 14:38:27.020 UTC [32105] postgres@new_db STATEMENT:  update users set name = 'Vera Murl' where id = 1;
2024-05-12 14:38:31.875 UTC [32080] postgres@new_db LOG:  process 32080 still waiting for ShareLock on transaction 2349850 after 200.100 ms
2024-05-12 14:38:31.875 UTC [32080] postgres@new_db DETAIL:  Process holding the lock: 32092. Wait queue: 32080.
2024-05-12 14:38:31.875 UTC [32080] postgres@new_db CONTEXT:  while updating tuple (0,11) in relation "users"
2024-05-12 14:38:31.875 UTC [32080] postgres@new_db STATEMENT:  update users set name = 'Vera Mur' where id = 2;
2024-05-12 14:38:37.316 UTC [32092] postgres@new_db LOG:  process 32092 detected deadlock while waiting for ExclusiveLock on tuple (0,12) of relation 16419 of database 16413 after 200.109 ms
2024-05-12 14:38:37.316 UTC [32092] postgres@new_db DETAIL:  Process holding the lock: 32105. Wait queue: .
2024-05-12 14:38:37.316 UTC [32092] postgres@new_db STATEMENT:  update users set name = 'Vera Mur' where id = 1;
2024-05-12 14:38:37.316 UTC [32092] postgres@new_db ERROR:  deadlock detected
2024-05-12 14:38:37.316 UTC [32092] postgres@new_db DETAIL:  Process 32092 waits for ExclusiveLock on tuple (0,12) of relation 16419 of database 16413; blocked by process 32105.
        Process 32105 waits for ShareLock on transaction 2349849; blocked by process 32080.
        Process 32080 waits for ShareLock on transaction 2349850; blocked by process 32092.
        Process 32092: update users set name = 'Vera Mur' where id = 1;
        Process 32105: update users set name = 'Vera Murl' where id = 1;
        Process 32080: update users set name = 'Vera Mur' where id = 2;
2024-05-12 14:38:37.316 UTC [32092] postgres@new_db HINT:  See server log for query details.
2024-05-12 14:38:37.316 UTC [32092] postgres@new_db STATEMENT:  update users set name = 'Vera Mur' where id = 1;
2024-05-12 14:38:37.316 UTC [32080] postgres@new_db LOG:  process 32080 acquired ShareLock on transaction 2349850 after 5641.171 ms
2024-05-12 14:38:37.316 UTC [32080] postgres@new_db CONTEXT:  while updating tuple (0,11) in relation "users"
2024-05-12 14:38:37.316 UTC [32080] postgres@new_db STATEMENT:  update users set name = 'Vera Mur' where id = 2;
2024-05-12 14:38:50.448 UTC [29682] LOG:  checkpoint starting: time
2024-05-12 14:38:50.565 UTC [29682] LOG:  checkpoint complete: wrote 2 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.102 s, sync=0.004 s, total=0.118 s; sync files=2, longest=0.002 s, average=0.002 s; distance=1 kB, estimate=1345 kB; lsn=0/6F4D4F90, redo lsn=0/6F4D4F50
```
Можно посмотреть лог и разобраться как получилась deadlock detected

#### 4. Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга? ####
Если одна команда будет обновлять строки в одном порядке, а другая — в другом, они могут взаимозаблокироваться. Получить такую ситуацию маловероятно, но тем не менее она может встретиться. Либо нужно как то контролировать переход к следующей строке - например курсором. 

