Котрольная точка каждые 30 сек:
```
postgres=# alter system set checkpoint_timeout = '30s';
ALTER SYSTEM
```
Нагрузка:
```
postgres@otus-db-pg-vm-1:~$ pgbench -c 50 -j 5 -P 60 -T 600 postgres
pgbench (16.2 (Ubuntu 16.2-1.pgdg22.04+1))
starting vacuum...end.
progress: 60.0 s, 418.3 tps, lat 119.148 ms stddev 122.321, 0 failed
progress: 120.0 s, 452.1 tps, lat 110.544 ms stddev 99.935, 0 failed
progress: 180.0 s, 401.0 tps, lat 124.750 ms stddev 117.091, 0 failed
progress: 240.0 s, 434.0 tps, lat 115.196 ms stddev 114.636, 0 failed
progress: 300.0 s, 439.8 tps, lat 113.768 ms stddev 111.984, 0 failed
progress: 360.0 s, 442.2 tps, lat 113.000 ms stddev 103.692, 0 failed
progress: 420.0 s, 437.2 tps, lat 114.348 ms stddev 105.875, 0 failed
progress: 480.0 s, 367.0 tps, lat 136.242 ms stddev 139.791, 0 failed
progress: 540.0 s, 414.9 tps, lat 120.588 ms stddev 107.483, 0 failed
progress: 600.0 s, 403.4 tps, lat 124.001 ms stddev 122.368, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 5
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 252640
number of failed transactions: 0 (0.000%)
latency average = 118.745 ms
latency stddev = 114.669 ms
initial connection time = 62.633 ms
tps = 421.019384 (without initial connection time)
```

