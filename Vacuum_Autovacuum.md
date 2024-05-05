Создала ВМ и установила пг.
Инициализация pgbench:
```
postgres@otus-db-pg-vm-1:~$ pgbench -i postgres
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.07 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.47 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 0.30 s, vacuum 0.05 s, primary keys 0.12 s).
```
```
postgres@otus-db-pg-vm-1:~$ pgbench -c8 -P 6 -T 60 -U postgres postgres
pgbench (16.2 (Ubuntu 16.2-1.pgdg22.04+1))
starting vacuum...end.
progress: 6.0 s, 355.5 tps, lat 22.151 ms stddev 29.464, 0 failed
progress: 12.0 s, 398.3 tps, lat 20.327 ms stddev 24.020, 0 failed
progress: 18.0 s, 430.5 tps, lat 18.581 ms stddev 12.377, 0 failed
progress: 24.0 s, 373.5 tps, lat 21.360 ms stddev 17.154, 0 failed
progress: 30.0 s, 282.2 tps, lat 28.428 ms stddev 22.860, 0 failed
progress: 36.0 s, 434.7 tps, lat 18.042 ms stddev 24.979, 0 failed
progress: 42.0 s, 452.0 tps, lat 18.044 ms stddev 18.058, 0 failed
progress: 48.0 s, 569.2 tps, lat 14.058 ms stddev 8.886, 0 failed
progress: 54.0 s, 471.3 tps, lat 16.959 ms stddev 12.633, 0 failed
progress: 60.0 s, 412.2 tps, lat 19.419 ms stddev 17.912, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 25084
number of failed transactions: 0 (0.000%)
latency average = 19.134 ms
latency stddev = 19.447 ms
initial connection time = 13.933 ms
tps = 417.998017 (without initial connection time)
```
Меняю настройки согласно файлу:
```
postgres=# ALTER SYSTEM SET max_connections = 40
postgres-# ;
ALTER SYSTEM
postgres=# ALTER SYSTEM SET shared_buffers = 1GB;
ERROR:  trailing junk after numeric literal at or near "1G"
LINE 1: ALTER SYSTEM SET shared_buffers = 1GB;
                                          ^
postgres=# ALTER SYSTEM SET shared_buffers = 1;
ERROR:  1 8kB is outside the valid range for parameter "shared_buffers" (16 .. 1073741823)
postgres=# ALTER SYSTEM SET shared_buffers = '1GB';
ALTER SYSTEM
postgres=# ALTER SYSTEM SET effective_cache_size = '3GB';
ALTER SYSTEM
postgres=# ALTER SYSTEM SET maintenance_work_mem = '512MB';
ALTER SYSTEM
postgres=# ALTER SYSTEM SET checkpoint_completion_target = 0.9
postgres-# ;
ALTER SYSTEM
postgres=# ALTER SYSTEM SET wal_buffers = '16MB';
ALTER SYSTEM
postgres=# ALTER SYSTEM SET default_statistics_target = 500
postgres-# ;
ALTER SYSTEM
postgres=# ALTER SYSTEM SET random_page_cost = 4;
ALTER SYSTEM
postgres=# ALTER SYSTEM SET effective_io_concurrency = 2;
ALTER SYSTEM
postgres=# ALTER SYSTEM SET work_mem = '6553kB';
ALTER SYSTEM
postgres=# ALTER SYSTEM SET min_wal_size = '4GB';
ALTER SYSTEM
postgres=# ALTER SYSTEM SET max_wal_size = '16GB';
ALTER SYSTEM
```
Смотрю снова:
```
postgres@otus-db-pg-vm-1:~$ pgbench -c8 -P 6 -T 60 -U postgres postgres
pgbench (16.2 (Ubuntu 16.2-1.pgdg22.04+1))
starting vacuum...end.
progress: 6.0 s, 476.3 tps, lat 16.724 ms stddev 12.160, 0 failed
progress: 12.0 s, 437.2 tps, lat 18.053 ms stddev 23.984, 0 failed
progress: 18.0 s, 483.8 tps, lat 16.730 ms stddev 13.710, 0 failed
progress: 24.0 s, 414.8 tps, lat 19.269 ms stddev 14.565, 0 failed
progress: 30.0 s, 363.7 tps, lat 22.042 ms stddev 15.554, 0 failed
progress: 36.0 s, 393.2 tps, lat 20.314 ms stddev 14.276, 0 failed
progress: 42.0 s, 244.8 tps, lat 32.590 ms stddev 30.023, 0 failed
progress: 48.0 s, 357.7 tps, lat 22.429 ms stddev 19.037, 0 failed
progress: 54.0 s, 462.2 tps, lat 17.334 ms stddev 13.483, 0 failed
progress: 60.0 s, 469.2 tps, lat 17.062 ms stddev 15.461, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 24625
number of failed transactions: 0 (0.000%)
latency average = 19.490 ms
latency stddev = 17.635 ms
initial connection time = 15.453 ms
tps = 410.370103 (without initial connection time)
```
Создаю таблицу, заполняю данными? смотрю на размер таблицы:
```
postgres=# CREATE TABLE random (val text);
CREATE TABLE
postgres=# INSERT INTO random
postgres-#    SELECT md5(random()::text)
postgres-#    FROM generate_series(1, 1000000) AS i;
INSERT 0 1000000
postgres=# select * from random limit 10;
               val
----------------------------------
 2c29e05b25e7183519f1fb5d3bc96042
 763c1b45cca50be6a093093ac043cb10
 fea605cc0ee0e975469cfac98432d6b0
 4161568c6d18309f816d149a9a3e6ec9
 33c064683796bf87c6cf6e5450b89a8d
 8aca4688e08e7441f38a4c456079ccd9
 e9568dc238786aee8a8e67d5f52cbf38
 2267792179db66a758c9b2002691534e
 d059202957de7c2619cdd0676b75eaf3
 4108d1d5ca11c91b93fe55e7f72d930e
(10 rows)
postgres=# select pg_size_pretty(pg_total_relation_size('random'));
 pg_size_pretty
----------------
 65 MB
(1 row)

```
Обновляю строчки и жду автовакуум:
```
postgres=# update random set val = val || '1';
UPDATE 1000000
postgres=# update random set val = val || '11';
UPDATE 1000000
postgres=# update random set val = val || '111';
UPDATE 1000000
postgres=# update random set val = val || '1111';
UPDATE 1000000
postgres=# update random set val = val || '11111';
UPDATE 1000000
postgres=# SELECT schemaname, relname, n_live_tup, n_dead_tup, last_autovacuum
FROM
pg_stat_user_tables where relname = 'random'
;
 schemaname | relname | n_live_tup | n_dead_tup |        last_autovacuum
------------+---------+------------+------------+-------------------------------
 public     | random  |    1005604 |    4999333 | 2024-05-05 12:53:48.330494+00
(1 row)

postgres=# SELECT schemaname, relname, n_live_tup, n_dead_tup, last_autovacuum
FROM
pg_stat_user_tables where relname = 'random'
;
 schemaname | relname | n_live_tup | n_dead_tup |        last_autovacuum
------------+---------+------------+------------+-------------------------------
 public     | random  |    1641439 |          0 | 2024-05-05 13:10:42.179641+00
(1 row)
```
Снова обновляю таблицу и смотрю ее размер:
```
postgres=# update random set val = val || '1';
UPDATE 1000000
postgres=# update random set val = val || '11';
UPDATE 1000000
postgres=# update random set val = val || '111';
UPDATE 1000000
postgres=# update random set val = val || '1111';
UPDATE 1000000
postgres=# update random set val = val || '11111';
UPDATE 1000000
postgres=# select pg_size_pretty(pg_total_relation_size('random'));
 pg_size_pretty
----------------
 492 MB
(1 row)
```
Отключаю автовакуум и обновляю снова:
```
postgres=# alter table random set (autovacuum_enabled = false);
ALTER TABLE
postgres=# update random set val = val || '1';
UPDATE 1000000
postgres=# update random set val = val || '11';
UPDATE 1000000
postgres=# update random set val = val || '111';
UPDATE 1000000
postgres=# update random set val = val || '1111';
UPDATE 1000000
postgres=# update random set val = val || '11111';
UPDATE 1000000
postgres=# update random set val = val || '111111';
^[[UPDATE 1000000
postgres=# update random set val = val || '1111111';
UPDATE 1000000
postgres=# update random set val = val || '11111111';
UPDATE 1000000
postgres=# update random set val = val || '111111111';
UPDATE 1000000
postgres=# update random set val = val || '10';
UPDATE 1000000
```
Проверяю размер таблицы, а он многократно увеличился т.к. был отключен автовакуум и мертные записи лежали тяжким грузом:
```
postgres=# select pg_size_pretty(pg_total_relation_size('random'));
 pg_size_pretty
----------------
 1369 MB
(1 row)
```
После очистки всё стало на свои места: 
```
postgres=# SELECT schemaname, relname, n_live_tup, n_dead_tup, last_autovacuum
FROM
pg_stat_user_tables where relname = 'random';
 schemaname | relname | n_live_tup | n_dead_tup |        last_autovacuum
------------+---------+------------+------------+-------------------------------
 public     | random  |    1000000 |          0 | 2024-05-05 13:40:44.558024+00
(1 row)

postgres=# select pg_size_pretty(pg_total_relation_size('random'));
 pg_size_pretty
----------------
 150 MB
(1 row)
```
