# Postgre-DBA-2023-01

1. Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд
```sh
log_lock_waits = on
deadlock_timeout = 200ms
``` 
  Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.
  
Первая сессия
```sh
postgres=# create database homework;
CREATE DATABASE
postgres=# \c homework

homework=# CREATE TABLE accounts(
homework(#   acc_no integer PRIMARY KEY,
homework(#   amount numeric
homework(# );
```
Вторая сессия
```sh
Begin;
```
Третья сессия
```sh
homework=*# CREATE INDEX ON accounts(acc_no);
```
Вывод из postgresql-15-main.log
```sh
2023-04-09 08:33:48.021 UTC [3465] postgres@homework LOG:  process 3465 still waiting for ShareLock on relation 16389 of database 16388 after 200.681 ms
2023-04-09 08:33:48.021 UTC [3465] postgres@homework DETAIL:  Process holding the lock: 3409. Wait queue: 3465.
2023-04-09 08:33:48.021 UTC [3465] postgres@homework STATEMENT:  CREATE INDEX ON accounts(acc_no);
2023-04-09 08:37:22.448 UTC [3465] postgres@homework LOG:  process 3465 acquired ShareLock on relation 16389 of database 16388 after 214628.165 ms
2023-04-09 08:37:22.448 UTC [3465] postgres@homework STATEMENT:  CREATE INDEX ON accounts(acc_no);
```

2.Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. 

Вывод первой транзакции:
```sh
homework=*# SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted
FROM pg_locks WHERE pid = 3403;
   locktype    |      relation       | virtxid | xid |       mode       | granted
---------------+---------------------+---------+-----+------------------+---------
 relation      | pg_locks            |         |     | AccessShareLock  | t
 relation      | accounts_acc_no_idx |         |     | RowExclusiveLock | t
 relation      | accounts_pkey       |         |     | RowExclusiveLock | t
 relation      | accounts            |         |     | RowExclusiveLock | t
 virtualxid    |                     | 4/21    |     | ExclusiveLock    | t
 transactionid |                     |         | 736 | ExclusiveLock    | t
```
Вывод второй транзакции:
```sh
homework=*# SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted
FROM pg_locks WHERE pid = 3409;
   locktype    |      relation       | virtxid | xid |       mode       | granted
---------------+---------------------+---------+-----+------------------+---------
 relation      | accounts_acc_no_idx |         |     | RowExclusiveLock | t
 relation      | accounts_pkey       |         |     | RowExclusiveLock | t
 relation      | accounts            |         |     | RowExclusiveLock | t
 virtualxid    |                     | 3/26    |     | ExclusiveLock    | t
 transactionid |                     |         | 736 | ShareLock        | f
 transactionid |                     |         | 737 | ExclusiveLock    | t
 tuple         | accounts            |         |     | ExclusiveLock    | t
```
Вывод третьей транзакции:
```sh
homework=*# SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted
homework-*# FROM pg_locks WHERE pid = 3465;
   locktype    |      relation       | virtxid | xid |       mode       | granted
---------------+---------------------+---------+-----+------------------+---------
 relation      | accounts_acc_no_idx |         |     | RowExclusiveLock | t
 relation      | accounts_pkey       |         |     | RowExclusiveLock | t
 relation      | accounts            |         |     | RowExclusiveLock | t
 virtualxid    |                     | 5/26    |     | ExclusiveLock    | t
 transactionid |                     |         | 738 | ExclusiveLock    | t
 tuple         | accounts            |         |     | ExclusiveLock    | f
```
Третья транзакция ничего не пишет про Sharelock от второй, пока не сделать commit в первой транзакции. (Как в выводе второй транзакции xid 736 (первая транзакция) ShareLock)

Посмотреть все блокировки в базе homework:
```sh
homework=*# select
  lock.locktype,
  lock.relation::regclass,
  lock.mode,
  lock.transactionid as tid,
  lock.virtualtransaction as vtid,
  lock.pid,
  lock.granted
from pg_catalog.pg_locks lock
  left join pg_catalog.pg_database db
    on db.oid = lock.database
where (db.datname = 'homework' or db.datname is null)                                                                     and not lock.pid = pg_backend_pid()
order by lock.pid;
   locktype    |      relation       |       mode       | tid | vtid | pid  | granted
---------------+---------------------+------------------+-----+------+------+---------
 virtualxid    |                     | ExclusiveLock    |     | 4/24 | 3403 | t
 transactionid |                     | ExclusiveLock    | 739 | 4/24 | 3403 | t
 tuple         | accounts            | ExclusiveLock    |     | 4/24 | 3403 | t
 transactionid |                     | ShareLock        | 737 | 4/24 | 3403 | f
 relation      | accounts_acc_no_idx | RowExclusiveLock |     | 4/24 | 3403 | t
 relation      | accounts_pkey       | RowExclusiveLock |     | 4/24 | 3403 | t
 relation      | accounts            | RowExclusiveLock |     | 4/24 | 3403 | t
 transactionid |                     | ShareLock        | 737 | 5/26 | 3465 | f
 relation      | accounts_pkey       | RowExclusiveLock |     | 5/26 | 3465 | t
 relation      | accounts            | RowExclusiveLock |     | 5/26 | 3465 | t
 virtualxid    |                     | ExclusiveLock    |     | 5/26 | 3465 | t
 transactionid |                     | ExclusiveLock    | 738 | 5/26 | 3465 | t
 relation      | accounts_acc_no_idx | RowExclusiveLock |     | 5/26 | 3465 | t;
```

3.Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?
Вывод из postgresql-15-main.log:
```sh
2023-04-09 08:54:40.508 UTC [3409] postgres@homework LOG:  process 3409 still waiting for ShareLock on transaction 736 after 200.598 ms
2023-04-09 08:54:40.508 UTC [3409] postgres@homework DETAIL:  Process holding the lock: 3403. Wait queue: 3409.
2023-04-09 08:54:40.508 UTC [3409] postgres@homework CONTEXT:  while updating tuple (0,4) in relation "accounts"
2023-04-09 08:54:40.508 UTC [3409] postgres@homework STATEMENT:  update accounts
        set amount=1112.00
        where acc_no=1;
2023-04-09 08:55:46.891 UTC [3465] postgres@homework LOG:  process 3465 still waiting for ExclusiveLock on tuple (0,4) of relation 16389 of database 16388 after 200.245 ms
2023-04-09 08:55:46.891 UTC [3465] postgres@homework DETAIL:  Process holding the lock: 3409. Wait queue: 3465.
2023-04-09 08:55:46.891 UTC [3465] postgres@homework STATEMENT:  update accounts
        set amount=1113.00
        where acc_no=1;
```
Да, в логе видно какая транзация заблокировала и даже видно каким запросом.

4. Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?

У меня не получилось. Первая транзакция заблокировала вторую. Но после commit`a первой всё ок.
