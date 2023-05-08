# Postgre-DBA-2023-01

Установим в файле /etc/postgresql/15/main/conf.d/main.conf значение параметра checkpoint_timeout равное 30s и log_checkpoints = on
Чтобы применились настройки, перезапустим кластер.
```sh
sudo pg_ctlcluster 15 main restart
```
Проверим, что кластер работает и настройки применились
```sh
ubuntu@ubuntu:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory                Log file
15  main    5432 online postgres /var/lib/postgresql/15/main   /var/log/postgresql/postgresql-15-main.log
15  second  5433 online postgres /var/lib/postgresql/15/second /var/log/postgresql/postgresql-15-second.log
15  third   5434 online postgres /var/lib/postgresql/15/third  /var/log/postgresql/postgresql-15-third.log
```
```sh
psql -U postgres -p 5432 -c "select * from pg_settings where "name" = 'checkpoint_timeout' or "name" = 'log_checkpoints'"
```

Узнаем текущую точку журнала:
```sh
ubuntu@ubuntu:~$ psql -U postgres -p 5432 -c "SELECT pg_current_wal_insert_lsn();"
Password for user postgres:
 pg_current_wal_insert_lsn
---------------------------
 0/4FD12838
(1 row)
```

Выполним инициализацию Pgbench. Иначе relation not found
```sh
ubuntu@ubuntu:~$ pgbench -i -U postgres -p 5432 postgres
Password:
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.06 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.22 s (drop tables 0.01 s, create tables 0.02 s, client-side generate 0.09 s, vacuum 0.04 s, primary keys 0.06 s)
```

Теперь запустим pgbench
```sh
ubuntu@ubuntu:~$ pgbench -c8 -P 60 -T 600 -U postgres -p 5432 postgres
Password:
pgbench (15.2 (Ubuntu 15.2-1.pgdg20.04+1))
starting vacuum...end.
progress: 60.0 s, 797.8 tps, lat 10.016 ms stddev 2.209, 0 failed
progress: 120.0 s, 408.6 tps, lat 19.571 ms stddev 13.261, 0 failed
progress: 180.0 s, 215.2 tps, lat 37.171 ms stddev 8.010, 0 failed
progress: 240.0 s, 212.6 tps, lat 37.627 ms stddev 8.190, 0 failed
progress: 300.0 s, 212.3 tps, lat 37.683 ms stddev 8.273, 0 failed
progress: 360.0 s, 212.2 tps, lat 37.690 ms stddev 7.917, 0 failed
progress: 420.0 s, 210.0 tps, lat 38.104 ms stddev 9.144, 0 failed
progress: 480.0 s, 205.2 tps, lat 38.986 ms stddev 8.545, 0 failed
progress: 540.0 s, 214.5 tps, lat 37.284 ms stddev 8.165, 0 failed
progress: 600.0 s, 209.8 tps, lat 38.120 ms stddev 8.453, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 173910
number of failed transactions: 0 (0.000%)
latency average = 27.597 ms
latency stddev = 14.905 ms
initial connection time = 52.030 ms
tps = 289.853852 (without initial connection time)
```
Среднее значение tps = 289

В логах видим, что контрольные точки создаются по расписанию и завершаются на 27ой секунде, потому что в конфиге checkpoint_completion_target = 0.9

```sh
	Строка  69: 2023-05-08 09:36:27.123 UTC [1767] LOG:  checkpoint starting: time
	Строка  71: 2023-05-08 09:36:57.122 UTC [1767] LOG:  checkpoint starting: time
	Строка  73: 2023-05-08 09:37:27.165 UTC [1767] LOG:  checkpoint starting: time
	Строка  75: 2023-05-08 09:37:57.161 UTC [1767] LOG:  checkpoint starting: time
	Строка  77: 2023-05-08 09:38:27.163 UTC [1767] LOG:  checkpoint starting: time
	Строка  79: 2023-05-08 09:38:57.163 UTC [1767] LOG:  checkpoint starting: time
	Строка  81: 2023-05-08 09:39:27.191 UTC [1767] LOG:  checkpoint starting: time
	Строка  83: 2023-05-08 09:39:57.094 UTC [1767] LOG:  checkpoint starting: time
	Строка  85: 2023-05-08 09:40:27.107 UTC [1767] LOG:  checkpoint starting: time
	Строка  87: 2023-05-08 09:40:57.115 UTC [1767] LOG:  checkpoint starting: time
	Строка  89: 2023-05-08 09:41:27.111 UTC [1767] LOG:  checkpoint starting: time
	Строка  91: 2023-05-08 09:42:57.100 UTC [1767] LOG:  checkpoint starting: time
	Строка  97: 2023-05-08 09:49:27.354 UTC [1767] LOG:  checkpoint starting: time
	Строка  99: 2023-05-08 09:49:57.080 UTC [1767] LOG:  checkpoint starting: time
	Строка 101: 2023-05-08 09:50:27.223 UTC [1767] LOG:  checkpoint starting: time
	Строка 103: 2023-05-08 09:50:57.130 UTC [1767] LOG:  checkpoint starting: time
	Строка 105: 2023-05-08 09:51:27.147 UTC [1767] LOG:  checkpoint starting: time
	Строка 107: 2023-05-08 09:51:57.123 UTC [1767] LOG:  checkpoint starting: time
	Строка 109: 2023-05-08 09:52:27.185 UTC [1767] LOG:  checkpoint starting: time
	Строка 111: 2023-05-08 09:52:57.131 UTC [1767] LOG:  checkpoint starting: time
	Строка 113: 2023-05-08 09:53:27.151 UTC [1767] LOG:  checkpoint starting: time
```

Проверим, сколько контрольных точеку выполнилось за время теста 

```sh
ubuntu@ubuntu:~$ psql -U postgres -p 5432
Password for user postgres:
psql (15.2 (Ubuntu 15.2-1.pgdg20.04+1))
Type "help" for help.

postgres=# SELECT * FROM pg_stat_bgwriter \gx
-[ RECORD 1 ]---------+------------------------------
checkpoints_timed     | 23
checkpoints_req       | 0
checkpoint_write_time | 536707
checkpoint_sync_time  | 872
buffers_checkpoint    | 33079
buffers_clean         | 0
maxwritten_clean      | 0
buffers_backend       | 1190
buffers_backend_fsync | 0
buffers_alloc         | 1185
stats_reset           | 2023-05-08 09:31:23.667801+00
```

Чтобы узнать средний размер контрольной точки, получим значение последней точки и высчитаем общий размер. Потом поделим на кол-во контрольных точек

```sh
postgres=# SELECT pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn
---------------------------
 0/66C1CB10
(1 row)

postgres=# select pg_size_pretty('0/66C1CB10'::pg_lsn - '0/4FD12838'::pg_lsn);
 pg_size_pretty
----------------
 367 MB
(1 row)

367/23 = 15.6 MB
```

Проверка кол-ва TPS при асинхронном коммите

```sh
postgres=# ALTER SYSTEM SET synchronous_commit TO off;

ubuntu@ubuntu:~$ pgbench -c8 -P 60 -T 600 -U postgres -p 5432 postgres
Password:
pgbench (15.2 (Ubuntu 15.2-1.pgdg20.04+1))
starting vacuum...end.
progress: 60.0 s, 2770.7 tps, lat 2.884 ms stddev 0.526, 0 failed
progress: 120.0 s, 2798.7 tps, lat 2.858 ms stddev 0.425, 0 failed
progress: 180.0 s, 2813.1 tps, lat 2.843 ms stddev 0.402, 0 failed
progress: 240.0 s, 2768.9 tps, lat 2.889 ms stddev 0.394, 0 failed
progress: 300.0 s, 2540.8 tps, lat 3.148 ms stddev 1.215, 0 failed
progress: 360.0 s, 2739.8 tps, lat 2.919 ms stddev 0.414, 0 failed
progress: 420.0 s, 2747.7 tps, lat 2.911 ms stddev 0.413, 0 failed
progress: 480.0 s, 2804.7 tps, lat 2.852 ms stddev 0.403, 0 failed
progress: 540.0 s, 2818.0 tps, lat 2.838 ms stddev 0.543, 0 failed
progress: 600.0 s, 2732.4 tps, lat 2.927 ms stddev 0.414, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 1652091
number of failed transactions: 0 (0.000%)
latency average = 2.905 ms
latency stddev = 0.565 ms
initial connection time = 52.650 ms
tps = 2753.658843 (without initial connection time)
```
Количество tps вырастает, потому что postgres не ждёт записи на диск.


Создайте новый кластер с включенной контрольной суммой страниц

```sh
sudo pg_createcluster 15 checksum -- --data-checksums
sudo pg_ctlcluster 15 checksum start
```

Создадим и заполним таблицу

```sh
ubuntu@ubuntu:~$ sudo psql -p 5435 -U postgres
psql (15.2 (Ubuntu 15.2-1.pgdg20.04+1))
Type "help" for help.

postgres=# create table qwe (c int);
ert into qwe values (1),(2),(3),(4),(5);
 SELECT pg_relCREATE TABLE
postgres=#  insert into qwe values (1),(2),(3),(4),(5);
ation_filepath('qwe');INSERT 0 5

```
Найдём файл, в котором хранится таблица

```sh
postgres=#  SELECT pg_relation_filepath('qwe');
 pg_relation_filepath
----------------------
 base/5/16388
(1 row)
```

Остановим кластер и изменим файл

```sh
postgres=# \q
ubuntu@ubuntu:~$ sudo pg_ctlcluster 15 checksum stop
ubuntu@ubuntu:~$ sudo dd if=/dev/zero of=/var/lib/postgresql/15/checksum/base/5/16388 oflag=dsync conv=notrunc bs=1024 count=8
8+0 records in
8+0 records out
8192 bytes (8.2 kB, 8.0 KiB) copied, 0.00800721 s, 1.0 MB/s
```

Запустим кластер и выполним select на нашей таблице

```sh
postgres=# select * from qwe;
 c
---
(0 rows)
```
Ошибки нет, но в таблице, почему-то нет данных.
