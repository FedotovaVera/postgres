Я создала 3 ВМ

На первой из них создала таблицы test, test2
Создаю публикацию таблицы test и подписываемся на публикацию таблицы test2 с ВМ №2.

***Первая ВМ***
```
postgres=# create publication test_pub for table test;
```
```
postgres=# \dRp+
                            Publication test_pub
  Owner   | All tables | Inserts | Updates | Deletes | Truncates | Via root
----------+------------+---------+---------+---------+-----------+----------
 postgres | f          | t       | t       | t       | t         | f
Tables:
    "public.test"
```
***Вторая ВМ***
```
postgres=# create publication test_pub for table test2;
```
```
postgres=# \dRp+
                            Publication test_pub
  Owner   | All tables | Inserts | Updates | Deletes | Truncates | Via root
----------+------------+---------+---------+---------+-----------+----------
 postgres | f          | t       | t       | t       | t         | f
Tables:
    "public.test2"
```
***Первая ВМ***
```
postgres=# create subscription test_pub connection 'host=replication-vn-2 port=5432 user=postgres password=postgres dbname=postgres' publication test_pub with (copy_data = false);
ERROR:  could not connect to the publisher: connection to server at "replication-vn-2" (10.128.0.19), port 5432 failed: Connection refused
        Is the server running on that host and accepting TCP/IP connections?
```
***Пытаюсь навести порядок***
Прослушку локалхоста заменила на *
```
# - Connection Settings -
listen_addresses = '*'
```
проверила wal_level - на всех 3-х вм установлено в значение replica
```
postgres=# show wal_level;
 wal_level
-----------
 replica
(1 row)
postgres=# alter system set wal_level = logical;
ALTER SYSTEM
```
И перезагрузилась
```
dmitrydergunov95@replication-vn-2:~$ sudo pg_ctlcluster 14 main restart
```
Проверила
```
postgres=# show wal_level;
 wal_level
-----------
 logical
(1 row)
```
***Попытка номер два с первой вм***
```
postgres=# create subscription test_pub connection 'host=replication-vn-2 port=5432 user=postgres password=postgres dbname=postgres' publication test_pub with (copy_data = false);
ERROR:  could not connect to the publisher: connection to server at "replication-vn-2" (10.128.0.19), port 5432 failed: FATAL:  no pg_hba.conf entry for host "10.128.0.15", user "postgres", database "postgres", SSL encryption
connection to server at "replication-vn-2" (10.128.0.19), port 5432 failed: FATAL:  no pg_hba.conf entry for host "10.128.0.15", user "postgres", database "postgres", no encryption
```
Добавила ip в pg_hba.conf
Попытка номер 3
```
postgres=# create subscription test_pub connection 'host=replication-vn-2 port=5432 user=postgres password=postgres dbname=postgres' publication test_pub with (copy_data = false);
ERROR:  could not connect to the publisher: connection to server at "replication-vn-2" (10.128.0.19), port 5432 failed: FATAL:  password authentication failed for user "postgres"
connection to server at "replication-vn-2" (10.128.0.19), port 5432 failed: FATAL:  password authentication failed for user "postgres"
```
Поняла, исправляю
```
postgres=# alter user postgres password 'postgres';
ALTER ROLE
```
***Попытка номер 4***
```
postgres=# create subscription test_pub connection 'host=replication-vn-2 port=5432 user=postgres password=postgres dbname=postgres' publication test_pub with (copy_data = false);
NOTICE:  created replication slot "test_pub" on publisher
CREATE SUBSCRIPTION
```
***Вторая ВМ***
```
postgres=# create subscription test_sub connection 'host=replication-vn-1 port=5432 user=postgres password=postgres dbname=postgres' publication test_pub with (copy_data = false);
NOTICE:  created replication slot "test_sub" on publisher
CREATE SUBSCRIPTION
```
****ВМ 1****
```
postgres=# insert into test values(1);
INSERT 0 1
postgres=# insert into test values(11);
INSERT 0 1
postgres=# insert into test values(111);
INSERT 0 1
```
****ВМ 2****
```
postgres=# insert into test2 values(2);
INSERT 0 1
postgres=# insert into test2 values(22);
INSERT 0 1
postgres=# insert into test2 values(222);
INSERT 0 1
```
****ВМ 1****
```
postgres=# select * from test2;
  i
-----
   2
  22
 222
(3 rows)
```
****ВМ 2****
```
postgres=# select * from test;
  i
-----
   1
  11
 111
(3 rows)
```
