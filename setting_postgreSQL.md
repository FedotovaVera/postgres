Развернула ВМ и поставила пг.
на https://pgconfigurator.cybertec.at/ смотрю рекомендации по настройке
Но прежде чем вносить изменения , я нагружу кластер:
```
postgres@test:~$ pgbench -c 100 -P 10 -T 60 -j 100 -U postgres
pgbench (15.7 (Ubuntu 15.7-1.pgdg22.04+1))
starting vacuum...end.
progress: 10.0 s, 389.1 tps, lat 249.776 ms stddev 223.047, 0 failed
progress: 20.0 s, 433.5 tps, lat 231.237 ms stddev 285.728, 0 failed
progress: 30.0 s, 499.6 tps, lat 194.420 ms stddev 207.483, 0 failed
progress: 40.0 s, 261.2 tps, lat 389.368 ms stddev 324.972, 0 failed
progress: 50.0 s, 335.8 tps, lat 300.173 ms stddev 321.966, 0 failed
progress: 60.0 s, 504.1 tps, lat 197.031 ms stddev 173.964, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 100
number of threads: 100
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 24332
number of failed transactions: 0 (0.000%)
latency average = 247.492 ms
latency stddev = 260.106 ms
initial connection time = 126.460 ms
tps = 402.563735 (without initial connection time)
```
Затем установила рекомендованные настройки в postgresql.conf
И снова запустила нагрузку:
```
postgres@test:~$ psql
psql (15.7 (Ubuntu 15.7-1.pgdg22.04+1))
Type "help" for help.

postgres=# select current_setting('config_file');
             current_setting
-----------------------------------------
 /etc/postgresql/15/main/postgresql.conf
(1 row)

postgres=# \q
postgres@test:~$ nano /etc/postgresql/15/main/postgresql.conf
postgres@test:~$ exit
logout
dmitrydergunov95@test:~$ sudo service postgresql stop
dmitrydergunov95@test:~$  pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 down   postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
dmitrydergunov95@test:~$  sudo service postgresql start
dmitrydergunov95@test:~$ sudo -i -u postgres
postgres@test:~$ pgbench -c 100 -P 10 -T 60 -j 100 -U postgres
pgbench (15.7 (Ubuntu 15.7-1.pgdg22.04+1))
starting vacuum...end.
progress: 10.0 s, 353.7 tps, lat 275.088 ms stddev 298.620, 0 failed
progress: 20.0 s, 404.7 tps, lat 244.725 ms stddev 208.376, 0 failed
progress: 30.0 s, 382.5 tps, lat 263.097 ms stddev 225.110, 0 failed
progress: 40.0 s, 374.3 tps, lat 268.680 ms stddev 277.271, 0 failed
progress: 50.0 s, 518.1 tps, lat 193.268 ms stddev 166.978, 0 failed
progress: 60.0 s, 568.2 tps, lat 174.954 ms stddev 145.320, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 100
number of threads: 100
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 26114
number of failed transactions: 0 (0.000%)
latency average = 229.872 ms
latency stddev = 221.642 ms
initial connection time = 128.022 ms
tps = 434.061599 (without initial connection time)
```
Не заметила особых изменений.
Я что-то сделала не так? 
