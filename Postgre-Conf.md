# Postgre-DBA-2023-01

Конфигурация ВМ:
CPU - 1
RAM - 4 GB
SSD

Установлен кластер postgresql 15

Изменения в postgresql.conf:
```sh
max_connections = 100
shared_buffers = 1GB
work_mem = 5242kB
effective_io_concurrency = 200
synchronous_commit = off
checkpoint_timeout = 15min
checkpoint_completion_target = 0.9
max_wal_size = 4GB
min_wal_size = 1GB
random_page_cost = 1.1
default_statistics_target = 100
effective_cache_size = 3GB
maintenance_work_mem = 256MB
wal_buffers = 16MB
```
Первый запуск pgbench. 
Пришлось сначала запустить инициализацию, иначе выдавал ошибку:
pgbench: error: could not count number of branches: ERROR:  relation "pgbench_branches" does not exist
```sh
ubuntu@ubuntu:~$ pgbench -i -U postgres postgres
Password:
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
done in 0.20 s (drop tables 0.00 s, create tables 0.00 s, client-side generate 0.11 s, vacuum 0.03 s, primary keys 0.05 s).
```
Запуск с выключенным вакуумом и synchronous_commit = off
```sh
ubuntu@ubuntu:~$ pgbench -c10 -P 60 -T 300 -n -U postgres postgres
Password:
pgbench (15.2 (Ubuntu 15.2-1.pgdg20.04+1))
progress: 60.0 s, 2321.2 tps, lat 4.303 ms stddev 0.880, 0 failed
progress: 120.0 s, 2285.7 tps, lat 4.374 ms stddev 1.218, 0 failed
progress: 180.0 s, 2310.2 tps, lat 4.328 ms stddev 0.781, 0 failed
progress: 240.0 s, 2287.0 tps, lat 4.372 ms stddev 1.274, 0 failed
progress: 300.0 s, 2188.2 tps, lat 4.569 ms stddev 1.184, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 1
maximum number of tries: 1
duration: 300 s
number of transactions actually processed: 683548
number of failed transactions: 0 (0.000%)
latency average = 4.387 ms
latency stddev = 1.088 ms
initial connection time = 65.226 ms
tps = 2278.823542 (without initial connection time)
```

Запуск с включенным вакуумом и synchronous_commit = off
```sh
ubuntu@ubuntu:~$ pgbench -c10 -P 60 -T 300 -U postgres postgres
Password:
pgbench (15.2 (Ubuntu 15.2-1.pgdg20.04+1))
starting vacuum...end.
progress: 60.0 s, 2092.3 tps, lat 4.773 ms stddev 1.319, 0 failed
progress: 120.0 s, 2046.9 tps, lat 4.885 ms stddev 1.712, 0 failed
progress: 180.0 s, 2187.9 tps, lat 4.570 ms stddev 0.748, 0 failed
progress: 240.0 s, 2212.5 tps, lat 4.519 ms stddev 1.165, 0 failed
progress: 300.0 s, 2246.2 tps, lat 4.451 ms stddev 0.702, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 1
maximum number of tries: 1
duration: 300 s
number of transactions actually processed: 647154
number of failed transactions: 0 (0.000%)
latency average = 4.634 ms
latency stddev = 1.190 ms
initial connection time = 65.946 ms
tps = 2157.514206 (without initial connection time)
```

Запуск с выключенным вакуумом и synchronous_commit = on 
```sh
ubuntu@ubuntu:~$ pgbench -c10 -P 60 -T 300 -n -U postgres postgres
Password:
pgbench (15.2 (Ubuntu 15.2-1.pgdg20.04+1))
progress: 60.0 s, 708.4 tps, lat 14.097 ms stddev 3.585, 0 failed
progress: 120.0 s, 461.5 tps, lat 21.664 ms stddev 13.477, 0 failed
progress: 180.0 s, 245.4 tps, lat 40.750 ms stddev 10.692, 0 failed
progress: 240.0 s, 246.5 tps, lat 40.565 ms stddev 10.491, 0 failed
progress: 300.0 s, 245.7 tps, lat 40.705 ms stddev 10.559, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 1
maximum number of tries: 1
duration: 300 s
number of transactions actually processed: 114458
number of failed transactions: 0 (0.000%)
latency average = 26.206 ms
latency stddev = 15.246 ms
initial connection time = 69.288 ms
tps = 381.543278 (without initial connection time)
```
Запуск с включенным вакуумом и synchronous_commit = on 
```sh
ubuntu@ubuntu:~$ pgbench -c10 -P 60 -T 300 -U postgres postgres
Password:
pgbench (15.2 (Ubuntu 15.2-1.pgdg20.04+1))
starting vacuum...end.
progress: 60.0 s, 697.8 tps, lat 14.312 ms stddev 3.546, 0 failed
progress: 120.0 s, 450.3 tps, lat 22.203 ms stddev 14.198, 0 failed
progress: 180.0 s, 238.3 tps, lat 41.956 ms stddev 10.745, 0 failed
progress: 240.0 s, 239.8 tps, lat 41.694 ms stddev 11.104, 0 failed
progress: 300.0 s, 243.1 tps, lat 41.147 ms stddev 10.724, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 1
maximum number of tries: 1
duration: 300 s
number of transactions actually processed: 112168
number of failed transactions: 0 (0.000%)
latency average = 26.740 ms
latency stddev = 15.702 ms
initial connection time = 66.836 ms
tps = 373.922270 (without initial connection time)
```
