# Postgre-DBA-2023-01
2 зайдите в созданный кластер под пользователем postgres
3 создайте новую базу данных testdb
4 зайдите в созданную базу данных под пользователем postgres
5 создайте новую схему testnm
```sh
ubuntu@ubuntu:~$ sudo -u postgres psql
psql (14.7 (Ubuntu 14.7-1.pgdg20.04+1))
Type "help" for help.

postgres=# \?
postgres=# create database testdb;
CREATE DATABASE
postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
testdb=# create schema testnm;
CREATE SCHEMA
testdb=#

```

создайте новую таблицу t1 с одной колонкой c1 типа integer

вставьте строку со значением c1=1
```sh
testdb=# create table testnm.t1 (c1 int);
CREATE TABLE
testdb=# insert into testnm.t1 values (1);
INSERT 0 1
testdb=# select * from  testnm.t1;
 c1
----
  1
(1 row)

```

создайте новую роль readonly
```sh
testdb=# create role readonly with login;
CREATE ROLE

testdb=# \du
                                   List of roles
 Role name |                         Attributes                         | Member of
-----------+------------------------------------------------------------+-----------
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
 readonly  |                                                            | {}

```

9 дайте новой роли право на подключение к базе данных testdb
10 дайте новой роли право на использование схемы testnm
11 дайте новой роли право на select для всех таблиц схемы testnm
```sh
testdb=# grant connect on database testdb to readonly;
GRANT
testdb=# grant usage on schema testnm to readonly;
GRANT
testdb=# grant select on all tables in schema testnm to readonly;
GRANT
```

12 создайте пользователя testread с паролем test123

13 дайте роль readonly пользователю testread
```sh
testdb=# create user testread with password 'test123';
CREATE ROLE
testdb=# grant readonly to testread;
GRANT ROLE
testdb=# \du
                                    List of roles
 Role name |                         Attributes                         | Member of
-----------+------------------------------------------------------------+------------
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
 readonly  |                                                            | {}
 testread  |                                                            | {readonly}

```
14 зайдите под пользователем testread в базу данных testdb

15 сделайте select * from t1;

16 получилось? (могло если вы делали сами не по шпаргалке и не упустили один существенный момент про который позже)
```sh
Получилось, но не сразу. Забыл, что надо править pg_hba, потому что там  было указано peer для локального подключения.Поменял на scram-sha-256.


ubuntu@ubuntu:~$ psql -d testdb -U testread -h 127.0.0.1
Password for user testread:
psql (14.7 (Ubuntu 14.7-1.pgdg20.04+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

testdb=> select * from testnm.t1;
 c1
----
  1
(1 row)

```
```sh
Поскольку я изначально создал таблицу в правильной схеме, то пункты 16-33 не актуальны. Подглядел в шпаргалке, что имелось в виду в этих пунктах.
```

34 теперь попробуйте выполнить команду create table t2(c1 integer); insert into t2 values (2);
```sh
testdb=> create table t2(c1 integer); insert into t2 values (2);
CREATE TABLE
INSERT 0 1
testdb=> \dt
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | t2   | table | testread
(1 row)

```

37 если вы справились сами то расскажите что сделали и почему, если смотрели шпаргалку - объясните что сделали и почему выполнив указанные в ней команды
```sh
Честно подглядел в шпаргалке, потому что не знал про дефолтное применение роли Public.


revoke CREATE on SCHEMA public FROM public; - чтобы роль public не могла создавать новые объекты в схеме public
revoke all on DATABASE testdb FROM public;  - чтобы роль public не могла выполнять select,update,delete и т.д. в базе testdb
```