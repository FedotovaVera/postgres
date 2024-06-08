Котрольная точка каждые 30 сек:
```
postgres=# alter system set checkpoint_timeout = '30s';
ALTER SYSTEM
```
Текущий LSN и количество выполненных до текущего момента контрольных точек:
```
postgres=# select pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn
---------------------------
 0/DCC9708
(1 row)

postgres=# select checkpoints_timed from pg_stat_bgwriter;
 checkpoints_timed
-------------------
                18
(1 row)
```
Нагрузка:
```
postgres@otus-db-pg-vm-1:~$ pgbench -c 8 -P 60 -T 600 -U postgres postgres
pgbench (16.3 (Ubuntu 16.3-1.pgdg22.04+1))
starting vacuum...end.
progress: 60.0 s, 582.6 tps, lat 13.726 ms stddev 9.657, 0 failed
progress: 120.0 s, 542.3 tps, lat 14.751 ms stddev 10.561, 0 failed
progress: 180.0 s, 568.2 tps, lat 14.077 ms stddev 9.971, 0 failed
progress: 240.0 s, 561.0 tps, lat 14.261 ms stddev 10.197, 0 failed
progress: 300.0 s, 559.5 tps, lat 14.299 ms stddev 10.146, 0 failed
progress: 360.0 s, 554.6 tps, lat 14.423 ms stddev 9.824, 0 failed
progress: 420.0 s, 549.1 tps, lat 14.568 ms stddev 10.029, 0 failed
progress: 480.0 s, 582.5 tps, lat 13.733 ms stddev 9.973, 0 failed
progress: 540.0 s, 575.7 tps, lat 13.895 ms stddev 10.027, 0 failed
progress: 600.0 s, 554.4 tps, lat 14.431 ms stddev 10.790, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 337802
number of failed transactions: 0 (0.000%)
latency average = 14.209 ms
latency stddev = 10.123 ms
initial connection time = 15.587 ms
tps = 563.002347 (without initial connection time)
```
Проверяем повторно: 
```
postgres=# select checkpoints_timed from pg_stat_bgwriter;
 checkpoints_timed
-------------------
                40
(1 row)

postgres=# select pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn
---------------------------
 0/294543F8
(1 row)
```
Объем байтов между ними:
```
postgres=# select '0/294543F8'::pg_lsn - '0/DCC9708'::pg_lsn as byte_size;
 byte_size
-----------
 460893424
(1 row)
```
Смотрим лог:
```
2024-05-11 17:53:11.169 UTC [17446] LOG:  checkpoint starting: time
2024-05-11 17:53:15.326 UTC [17446] LOG:  checkpoint complete: wrote 44 buffers (0.3%); 0 WAL file(s) added, 0 removed, 0 recycled; write=4.118 s, sync=0.014 s, total=4.157 s; sync files=11, longest=0.011 s, average=0.002 s; distance=260 kB, estimate=260 kB; lsn=0/1528398, redo lsn=0/1528360
2024-05-11 17:58:11.411 UTC [17446] LOG:  checkpoint starting: time
2024-05-11 17:58:11.630 UTC [17446] LOG:  checkpoint complete: wrote 3 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.203 s, sync=0.006 s, total=0.219 s; sync files=3, longest=0.004 s, average=0.002 s; distance=2 kB, estimate=234 kB; lsn=0/1528D40, redo lsn=0/1528D08
2024-05-11 18:03:11.727 UTC [17446] LOG:  checkpoint starting: time
2024-05-11 18:06:48.332 UTC [17446] LOG:  checkpoint complete: wrote 2181 buffers (13.3%); 1 WAL file(s) added, 1 removed, 0 recycled; write=216.340 s, sync=0.012 s, total=216.605 s; sync files=55, longest=0.012 s, average=0.001 s; distance=18796 kB, estimate=18796 kB; lsn=0/6E97A90, redo lsn=0/27840D0
2024-05-11 18:08:11.412 UTC [17446] LOG:  checkpoint starting: time
2024-05-11 18:12:41.122 UTC [17446] LOG:  checkpoint complete: wrote 3216 buffers (19.6%); 0 WAL file(s) added, 0 removed, 6 recycled; write=269.608 s, sync=0.009 s, total=269.710 s; sync files=19, longest=0.009 s, average=0.001 s; distance=94813 kB, estimate=94813 kB; lsn=0/DA513B8, redo lsn=0/841B5A8
2024-05-11 18:13:11.152 UTC [17446] LOG:  checkpoint starting: time
2024-05-11 18:17:41.072 UTC [17446] LOG:  checkpoint complete: wrote 3229 buffers (19.7%); 0 WAL file(s) added, 0 removed, 5 recycled; write=269.899 s, sync=0.003 s, total=269.921 s; sync files=18, longest=0.003 s, average=0.001 s; distance=90750 kB, estimate=94406 kB; lsn=0/DCC9658, redo lsn=0/DCBB078
2024-05-11 19:11:48.544 UTC [17445] LOG:  parameter "checkpoint_timeout" changed to "30"
2024-05-11 19:13:18.602 UTC [17446] LOG:  checkpoint starting: time
2024-05-11 19:13:45.105 UTC [17446] LOG:  checkpoint complete: wrote 1989 buffers (12.1%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.368 s, sync=0.070 s, total=26.504 s; sync files=17, longest=0.039 s, average=0.005 s; distance=18598 kB, estimate=86826 kB; lsn=0/1037AAB8, redo lsn=0/EEE4A40
2024-05-11 19:13:48.107 UTC [17446] LOG:  checkpoint starting: time
2024-05-11 19:14:15.112 UTC [17446] LOG:  checkpoint complete: wrote 2016 buffers (12.3%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.867 s, sync=0.058 s, total=27.005 s; sync files=14, longest=0.037 s, average=0.005 s; distance=21988 kB, estimate=80342 kB; lsn=0/118DE9A0, redo lsn=0/1045DE38
2024-05-11 19:14:18.115 UTC [17446] LOG:  checkpoint starting: time
2024-05-11 19:14:45.086 UTC [17446] LOG:  checkpoint complete: wrote 1994 buffers (12.2%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.864 s, sync=0.024 s, total=26.972 s; sync files=7, longest=0.024 s, average=0.004 s; distance=21858 kB, estimate=74494 kB; lsn=0/12E34D30, redo lsn=0/119B6828
2024-05-11 19:14:48.089 UTC [17446] LOG:  checkpoint starting: time
2024-05-11 19:15:15.063 UTC [17446] LOG:  checkpoint complete: wrote 2132 buffers (13.0%); 0 WAL file(s) added, 1 removed, 0 recycled; write=26.901 s, sync=0.025 s, total=26.974 s; sync files=16, longest=0.012 s, average=0.002 s; distance=21766 kB, estimate=69221 kB; lsn=0/14296900, redo lsn=0/12EF80F8
2024-05-11 19:15:18.066 UTC [17446] LOG:  checkpoint starting: time
2024-05-11 19:15:45.122 UTC [17446] LOG:  checkpoint complete: wrote 1979 buffers (12.1%); 0 WAL file(s) added, 1 removed, 1 recycled; write=26.970 s, sync=0.015 s, total=27.057 s; sync files=9, longest=0.015 s, average=0.002 s; distance=20911 kB, estimate=64390 kB; lsn=0/158361B8, redo lsn=0/14363FB0
2024-05-11 19:15:48.126 UTC [17446] LOG:  checkpoint starting: time
2024-05-11 19:16:15.109 UTC [17446] LOG:  checkpoint complete: wrote 2122 buffers (13.0%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.878 s, sync=0.025 s, total=26.984 s; sync files=15, longest=0.017 s, average=0.002 s; distance=22253 kB, estimate=60176 kB; lsn=0/16D80B58, redo lsn=0/1591F4D8
2024-05-11 19:16:18.112 UTC [17446] LOG:  checkpoint starting: time
2024-05-11 19:16:45.138 UTC [17446] LOG:  checkpoint complete: wrote 1978 buffers (12.1%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.968 s, sync=0.013 s, total=27.026 s; sync files=8, longest=0.013 s, average=0.002 s; distance=21818 kB, estimate=56340 kB; lsn=0/182D1B78, redo lsn=0/16E6DE60
2024-05-11 19:16:48.141 UTC [17446] LOG:  checkpoint starting: time
2024-05-11 19:17:15.105 UTC [17446] LOG:  checkpoint complete: wrote 2110 buffers (12.9%); 0 WAL file(s) added, 2 removed, 0 recycled; write=26.872 s, sync=0.027 s, total=26.965 s; sync files=15, longest=0.017 s, average=0.002 s; distance=21745 kB, estimate=52881 kB; lsn=0/19820348, redo lsn=0/183AA380
2024-05-11 19:17:18.107 UTC [17446] LOG:  checkpoint starting: time
2024-05-11 19:17:45.133 UTC [17446] LOG:  checkpoint complete: wrote 1961 buffers (12.0%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.970 s, sync=0.008 s, total=27.026 s; sync files=10, longest=0.008 s, average=0.001 s; distance=21860 kB, estimate=49779 kB; lsn=0/1AD2DC10, redo lsn=0/19903420
2024-05-11 19:17:48.136 UTC [17446] LOG:  checkpoint starting: time
2024-05-11 19:18:15.093 UTC [17446] LOG:  checkpoint complete: wrote 2099 buffers (12.8%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.867 s, sync=0.035 s, total=26.958 s; sync files=17, longest=0.018 s, average=0.003 s; distance=21534 kB, estimate=46954 kB; lsn=0/1C29D0B0, redo lsn=0/1AE0AFB8
2024-05-11 19:18:18.096 UTC [17446] LOG:  checkpoint starting: time
2024-05-11 19:18:45.142 UTC [17446] LOG:  checkpoint complete: wrote 1969 buffers (12.0%); 0 WAL file(s) added, 1 removed, 1 recycled; write=26.968 s, sync=0.018 s, total=27.046 s; sync files=9, longest=0.018 s, average=0.002 s; distance=21968 kB, estimate=44456 kB; lsn=0/1D80D090, redo lsn=0/1C37F008
2024-05-11 19:18:48.145 UTC [17446] LOG:  checkpoint starting: time
2024-05-11 19:19:15.118 UTC [17446] LOG:  checkpoint complete: wrote 2101 buffers (12.8%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.873 s, sync=0.037 s, total=26.973 s; sync files=16, longest=0.017 s, average=0.003 s; distance=21876 kB, estimate=42198 kB; lsn=0/1ED062B8, redo lsn=0/1D8DC0E8
2024-05-11 19:19:18.121 UTC [17446] LOG:  checkpoint starting: time
2024-05-11 19:19:45.070 UTC [17446] LOG:  checkpoint complete: wrote 1939 buffers (11.8%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.872 s, sync=0.017 s, total=26.950 s; sync files=9, longest=0.017 s, average=0.002 s; distance=21524 kB, estimate=40130 kB; lsn=0/202A4728, redo lsn=0/1EDE1180
2024-05-11 19:19:48.073 UTC [17446] LOG:  checkpoint starting: time
2024-05-11 19:20:15.137 UTC [17446] LOG:  checkpoint complete: wrote 2110 buffers (12.9%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.970 s, sync=0.022 s, total=27.064 s; sync files=13, longest=0.018 s, average=0.002 s; distance=22110 kB, estimate=38328 kB; lsn=0/2179C528, redo lsn=0/20378BD0
2024-05-11 19:20:18.140 UTC [17446] LOG:  checkpoint starting: time
2024-05-11 19:20:45.102 UTC [17446] LOG:  checkpoint complete: wrote 1930 buffers (11.8%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.872 s, sync=0.017 s, total=26.963 s; sync files=8, longest=0.017 s, average=0.003 s; distance=21558 kB, estimate=36651 kB; lsn=0/22D61968, redo lsn=0/21886750
2024-05-11 19:20:48.103 UTC [17446] LOG:  checkpoint starting: time
2024-05-11 19:21:15.090 UTC [17446] LOG:  checkpoint complete: wrote 2296 buffers (14.0%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.887 s, sync=0.031 s, total=26.987 s; sync files=14, longest=0.016 s, average=0.003 s; distance=22186 kB, estimate=35205 kB; lsn=0/242C5C48, redo lsn=0/22E31220
2024-05-11 19:21:18.093 UTC [17446] LOG:  checkpoint starting: time
2024-05-11 19:21:45.162 UTC [17446] LOG:  checkpoint complete: wrote 1961 buffers (12.0%); 0 WAL file(s) added, 1 removed, 1 recycled; write=26.968 s, sync=0.022 s, total=27.069 s; sync files=8, longest=0.013 s, average=0.003 s; distance=21946 kB, estimate=33879 kB; lsn=0/25890828, redo lsn=0/2439FC98
2024-05-11 19:21:48.165 UTC [17446] LOG:  checkpoint starting: time
2024-05-11 19:22:15.146 UTC [17446] LOG:  checkpoint complete: wrote 2081 buffers (12.7%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.875 s, sync=0.029 s, total=26.982 s; sync files=13, longest=0.019 s, average=0.003 s; distance=22341 kB, estimate=32725 kB; lsn=0/26DF1D68, redo lsn=0/259713B8
2024-05-11 19:22:18.150 UTC [17446] LOG:  checkpoint starting: time
2024-05-11 19:22:45.111 UTC [17446] LOG:  checkpoint complete: wrote 1944 buffers (11.9%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.876 s, sync=0.022 s, total=26.962 s; sync files=9, longest=0.022 s, average=0.003 s; distance=21885 kB, estimate=31641 kB; lsn=0/2835A598, redo lsn=0/26ED0A98
2024-05-11 19:22:48.114 UTC [17446] LOG:  checkpoint starting: time
2024-05-11 19:23:15.055 UTC [17446] LOG:  checkpoint complete: wrote 2308 buffers (14.1%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.878 s, sync=0.019 s, total=26.942 s; sync files=15, longest=0.018 s, average=0.002 s; distance=21898 kB, estimate=30667 kB; lsn=0/2944A638, redo lsn=0/28433668
2024-05-11 19:23:48.089 UTC [17446] LOG:  checkpoint starting: time
2024-05-11 19:24:15.099 UTC [17446] LOG:  checkpoint complete: wrote 1961 buffers (12.0%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.968 s, sync=0.027 s, total=27.011 s; sync files=12, longest=0.019 s, average=0.003 s; distance=16515 kB, estimate=29252 kB; lsn=0/29454468, redo lsn=0/29454430
```
фиксация контрольной точки выполняется через 15 секунд. Проверяю checkpoint_completion_target и всё становится ясно - 30 секунд * на 0,9 = 27 секунд:
```
postgres=# show checkpoint_completion_target;
 checkpoint_completion_target
------------------------------
 0.9
(1 row)
```
 Включаю асинхронность и запускаю нагрузку повторно:
```
postgres@otus-db-pg-vm-1:~$ pgbench -c 50 -j 5 -P 60 -T 600 postgres
pgbench (16.3 (Ubuntu 16.3-1.pgdg22.04+1))
starting vacuum...end.
progress: 60.0 s, 2793.4 tps, lat 17.876 ms stddev 11.650, 0 failed
progress: 120.0 s, 2824.5 tps, lat 17.701 ms stddev 11.460, 0 failed
progress: 180.0 s, 2849.6 tps, lat 17.545 ms stddev 11.223, 0 failed
progress: 240.0 s, 2817.7 tps, lat 17.743 ms stddev 11.757, 0 failed
progress: 300.0 s, 2823.6 tps, lat 17.709 ms stddev 11.781, 0 failed
progress: 360.0 s, 2849.9 tps, lat 17.543 ms stddev 11.044, 0 failed
progress: 420.0 s, 2767.9 tps, lat 18.064 ms stddev 12.043, 0 failed
progress: 480.0 s, 2828.5 tps, lat 17.676 ms stddev 11.585, 0 failed
progress: 540.0 s, 2860.8 tps, lat 17.473 ms stddev 10.907, 0 failed
progress: 600.0 s, 2769.1 tps, lat 18.060 ms stddev 11.772, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 5
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 1691151
number of failed transactions: 0 (0.000%)
latency average = 17.738 ms
latency stddev = 11.529 ms
initial connection time = 66.962 ms
tps = 2818.427154 (without initial connection time)
```
Увеличилась производительность, т.к. в режиме асинхронного подтверждения сервер сообщает об успешном завершении сразу, как только транзакция будет завершена логически, прежде чем сгенерированные записи WAL фактически будут записаны на диск.

Создаю новый кластер с включенной контрольной суммой страниц и запускаю его, подключаюсь к нему:
```
dmitrydergunov95@otus-db-pg-vm-1:~$ sudo pg_createcluster 16 second_cluster -- --data-checksums
Creating new PostgreSQL cluster 16/second_cluster ...
/usr/lib/postgresql/16/bin/initdb -D /var/lib/postgresql/16/second_cluster --auth-local peer --auth-host scram-sha-256 --no-instructions --data-checksums
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locale "en_US.UTF-8".
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are enabled.

fixing permissions on existing directory /var/lib/postgresql/16/second_cluster ... ok
creating subdirectories ... ok
selecting dynamic shared memory implementation ... posix
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting default time zone ... Etc/UTC
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok
Ver Cluster        Port Status Owner    Data directory                        Log file
16  second_cluster 5434 down   postgres /var/lib/postgresql/16/second_cluster /var/log/postgresql/postgresql-16-second_cluster.log

dmitrydergunov95@otus-db-pg-vm-1:~$ sudo pg_ctlcluster 16 second_cluster start
```
Создаю таблицу и вставляю несколько значений:
```
postgres=# create table new_table (id text);
CREATE TABLE
postgres=# insert into new_table values('one');
INSERT 0 1
postgres=# insert into new_table values('two');
INSERT 0 1
postgres=# select pg_relation_filepath('new_table');
 pg_relation_filepath
----------------------
 base/5/16388
(1 row)
```
Остановила кластер. Заменила 16 битов нулями в файле таблицы:
```
dmitrydergunov95@otus-db-pg-vm-1:~$ sudo service postgresql stop
dmitrydergunov95@otus-db-pg-vm-1:~$  pg_lsclusters
Ver Cluster        Port Status Owner    Data directory                        Log file
16  main           5432 down   postgres /var/lib/postgresql/16/main           /var/log/postgresql/postgresql-16-main.log
16  second         5433 down   postgres /var/lib/postgresql/16/second         /var/log/postgresql/postgresql-16-second.log
16  second_cluster 5434 down   postgres /var/lib/postgresql/16/second_cluster /var/log/postgresql/postgresql-16-second_cluster.log
dmitrydergunov95@otus-db-pg-vm-1:~$ sudo dd if=/dev/zero of=/var/lib/postgresql/16/main/base/5/16388 oflag=dsync conv=notrunc bs=1 count=16
16+0 records in
16+0 records out
16 bytes copied, 0.0109461 s, 1.5 kB/s
```
Запускаю заново и смотрю таблицу:
```
dmitrydergunov95@otus-db-pg-vm-1:~$ sudo service postgresql start
dmitrydergunov95@otus-db-pg-vm-1:~$ pg_lsclusters
Ver Cluster        Port Status Owner    Data directory                        Log file
16  main           5432 online postgres /var/lib/postgresql/16/main           /var/log/postgresql/postgresql-16-main.log
16  second         5433 online postgres /var/lib/postgresql/16/second         /var/log/postgresql/postgresql-16-second.log
16  second_cluster 5434 online postgres /var/lib/postgresql/16/second_cluster /var/log/postgresql/postgresql-16-second_cluster.log
dmitrydergunov95@otus-db-pg-vm-1:~$ sudo su postgres
postgres@otus-db-pg-vm-1:/home/dmitrydergunov95$ psql -p 5434
psql (16.3 (Ubuntu 16.3-1.pgdg22.04+1))
Type "help" for help.

postgres=# select * from new_table;
 id
-----
 one
 two
(2 rows)
```
Почему у меня нет ошибки?


***Попытка номер 2:***
Создаю новый кластер, проверяю статус data_checksums, включаю data_checksums и запускаю кластер, подключаюсь к нему, проверяю:
```
postgres=# show data_checksums;
 data_checksums
----------------
 off
(1 row)

postgres=# \q
postgres@test-vm:/home/dmitrydergunov95$ exit
exit
dmitrydergunov95@test-vm:~$ sudo pg_ctlcluster 15 second_cluster stop
dmitrydergunov95@test-vm:~$ pg_lsclusters
Ver Cluster        Port Status Owner    Data directory                        Log file
15  main           5432 online postgres /var/lib/postgresql/15/main           /var/log/postgresql/postgresql-15-main.log
15  second_cluster 5433 down   postgres /var/lib/postgresql/15/second_cluster /var/log/postgresql/postgresql-15-second_cluster.log
dmitrydergunov95@test-vm:~$ sudo su postgres
postgres@test-vm:/home/dmitrydergunov95$ /usr/lib/postgresql/15/bin/pg_checksums --enable /var/lib/postgresql/15/second_cluster
Checksum operation completed
Files scanned:   946
Blocks scanned:  2807
Files written:  778
Blocks written: 2807
pg_checksums: syncing data directory
pg_checksums: updating control file
Checksums enabled in cluster
dmitrydergunov95@test-vm:~$ sudo pg_ctlcluster 15 second_cluster start
dmitrydergunov95@test-vm:~$ pg_lsclusters
Ver Cluster        Port Status Owner    Data directory                        Log file
15  main           5432 online postgres /var/lib/postgresql/15/main           /var/log/postgresql/postgresql-15-main.log
15  second_cluster 5433 online postgres /var/lib/postgresql/15/second_cluster /var/log/postgresql/postgresql-15-second_cluster.log
dmitrydergunov95@test-vm:~$ sudo su postgres
postgres@test-vm:/home/dmitrydergunov95$ psql -p 5433
psql (15.7 (Ubuntu 15.7-1.pgdg22.04+1))
Type "help" for help.

postgres=# show data_checksums;
 data_checksums
----------------
 on
(1 row)
```
Создаю таблицу, заполняю данными:
```
postgres=# create table new_table (id text);
CREATE TABLE
postgres=# insert into new_table values('one');
INSERT 0 1
postgres=# insert into new_table values('two');
INSERT 0 1
postgres=# select pg_relation_filepath('new_table');
 pg_relation_filepath
----------------------
 base/5/16388
(1 row)
```
Останавливаю:
```
dmitrydergunov95@test-vm:~$ sudo service postgresql stop
dmitrydergunov95@test-vm:~$ pg_lsclusters
Ver Cluster        Port Status Owner    Data directory                        Log file
15  main           5432 down   postgres /var/lib/postgresql/15/main           /var/log/postgresql/postgresql-15-main.log
15  second_cluster 5433 down   postgres /var/lib/postgresql/15/second_cluster /var/log/postgresql/postgresql-15-second_cluster.log
dmitrydergunov95@test-vm:~$ sudo dd if=/dev/zero of=/var/lib/postgresql/15/second_cluster/base/5/16388 oflag=dsync conv=notrunc bs=1 count=16
16+0 records in
16+0 records out
16 bytes copied, 0.0393849 s, 0.4 kB/s
dmitrydergunov95@test-vm:~$ sudo service postgresql start
dmitrydergunov95@test-vm:~$ pg_lsclusters
Ver Cluster        Port Status Owner    Data directory                        Log file
15  main           5432 online postgres /var/lib/postgresql/15/main           /var/log/postgresql/postgresql-15-main.log
15  second_cluster 5433 online postgres /var/lib/postgresql/15/second_cluster /var/log/postgresql/postgresql-15-second_cluster.log
dmitrydergunov95@test-vm:~$ sudo su postgres
postgres@test-vm:/home/dmitrydergunov95$ psql -p 5433
psql (15.7 (Ubuntu 15.7-1.pgdg22.04+1))
Type "help" for help.

postgres=# select * from new_table;
 id
-----
 one
 two
(2 rows)

postgres=# show data_checksums;
 data_checksums
----------------
 on
(1 row)

postgres=# show ignore_checksum_failure;
 ignore_checksum_failure
-------------------------
 off
(1 row)
```
Снова не получаю ошибку. Пересмотрела и поняла, что неверно указала кластер. Пробую еще раз:
```
dmitrydergunov95@test-vm:~$ sudo service postgresql stop
dmitrydergunov95@test-vm:~$ pg_lsclusters
Ver Cluster        Port Status Owner    Data directory                        Log file
15  main           5432 down   postgres /var/lib/postgresql/15/main           /var/log/postgresql/postgresql-15-main.log
15  second_cluster 5433 down   postgres /var/lib/postgresql/15/second_cluster /var/log/postgresql/postgresql-15-second_cluster.log
dmitrydergunov95@test-vm:~$ sudo dd if=/dev/zero of=/var/lib/postgresql/15/second_cluster/base/5/16388 oflag=dsync conv=notrunc bs=1 count=16
16+0 records in
16+0 records out
16 bytes copied, 0.0182047 s, 0.9 kB/s
dmitrydergunov95@test-vm:~$ sudo service postgresql start
dmitrydergunov95@test-vm:~$ pg_lsclusters
Ver Cluster        Port Status Owner    Data directory                        Log file
15  main           5432 online postgres /var/lib/postgresql/15/main           /var/log/postgresql/postgresql-15-main.log
15  second_cluster 5433 online postgres /var/lib/postgresql/15/second_cluster /var/log/postgresql/postgresql-15-second_cluster.log
dmitrydergunov95@test-vm:~$ sudo su postgres
postgres@test-vm:/home/dmitrydergunov95$ psql -p 5434
psql: error: connection to server on socket "/var/run/postgresql/.s.PGSQL.5434" failed: No such file or directory
        Is the server running locally and accepting connections on that socket?
postgres@test-vm:/home/dmitrydergunov95$ psql -p 5433
psql (15.7 (Ubuntu 15.7-1.pgdg22.04+1))
Type "help" for help.

postgres=# select * from new_table;
ERROR:  invalid page in block 0 of relation base/5/16388
```
Ура!)
При попытке восстановить страницы после повреждения иногда нужно обойти защиту, обеспечиваемую контрольными суммами. Для этого можно временно установить параметр конфигурации ignore_checksum_failure.
Пробую:
```
postgres=# show ignore_checksum_failure;
 ignore_checksum_failure
-------------------------
 off
(1 row)

postgres=# set ignore_checksum_failure = on;
SET
postgres=# show ignore_checksum_failure;
 ignore_checksum_failure
-------------------------
 on
(1 row)

postgres=# select * from new_table;
ERROR:  invalid page in block 0 of relation base/5/16388
```
Поможет только бэкап? Или я что то сделала не так?


**Попытка 3**
```
postgres=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

postgres=# show data_checksums;
 data_checksums
----------------
 on
(1 row)

postgres=# show ignore_checksum_failure;
 ignore_checksum_failure
-------------------------
 on
(1 row)

postgres=# select * from new_table;
ERROR:  invalid page in block 0 of relation base/5/16393
```
Или перезапустить кластер:
```
dmitrydergunov95@test:~$ sudo pg_ctlcluster 15 second_cluster stop
dmitrydergunov95@test:~$ sudo pg_ctlcluster 15 second_cluster start
dmitrydergunov95@test:~$ sudo su postgres
postgres@test:/home/dmitrydergunov95$ psql -p 5433
psql (15.7 (Ubuntu 15.7-1.pgdg22.04+1))
Type "help" for help.

postgres=# select * from new_table;
ERROR:  invalid page in block 0 of relation base/5/16393
postgres=# show ignore_checksum_failure;
 ignore_checksum_failure
-------------------------
 off
(1 row)

postgres=# show data_checksums;
 data_checksums
----------------
 on
(1 row)
postgres=# select * from new_table;
ERROR:  invalid page in block 0 of relation base/5/16393
```
**Попытка номер .. последняя :) **
```
dmitrydergunov95@otus-db-pg-vm-1:~$ sudo dd if=/dev/zero of=/var/lib/postgresql/15/second_cluster/base/5/16384 oflag=dsync conv=notrunc bs=1 count=5
5+0 records in
5+0 records out
5 bytes copied, 0.00347727 s, 1.4 kB/s
dmitrydergunov95@otus-db-pg-vm-1:~$ sudo service postgresql start
dmitrydergunov95@otus-db-pg-vm-1:~$ sudo su postgres
postgres@otus-db-pg-vm-1:/home/dmitrydergunov95$ psql -p 5433
psql (15.7 (Ubuntu 15.7-1.pgdg22.04+1))
Type "help" for help.

postgres=# select * from new_table;
WARNING:  page verification failed, calculated checksum 26469 but expected 40309
ERROR:  invalid page in block 0 of relation base/5/16384
postgres=# show ignore_checksum_failure;
 ignore_checksum_failure
-------------------------
 off
(1 row)

postgres=# set ignore_checksum_failure = on;
SET
postgres=# select * from new_table;
WARNING:  page verification failed, calculated checksum 26469 but expected 40309
 id
-----
 one
 two
(2 rows)
```
