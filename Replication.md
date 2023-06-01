# Postgre-DBA-2023-01

Создадим 3 кластера 15 версии

```sh
sudo pg_createcluster 15 main
sudo pg_createcluster 15 second
sudo pg_createcluster 15 third
```
Создадим файл  repl.conf, укажем listen_address и wal_level и поместим файл в папки /conf.d каждого кластера
```sh
nano repl.conf
listen_addresses = '*'
wal_level = logical
sudo cp repl.conf /conf.d  3х кластеров 
```
Разрешим подключение для репликации
```sh
echo "host all replicator $(hostname -I | awk '{print $1}')/32 scram-sha-256" | sudo tee -a /etc/postgresql/15/main/pg_hba.conf > /dev/null
echo "host all replicator $(hostname -I | awk '{print $1}')/32 scram-sha-256" | sudo tee -a /etc/postgresql/15/second/pg_hba.conf > /dev/null
echo "host all replicator $(hostname -I | awk '{print $1}')/32 scram-sha-256" | sudo tee -a /etc/postgresql/15/third/pg_hba.conf > /dev/null
```
Включим кластеры
```sh
sudo pg_ctlcluster 15 main start
sudo pg_ctlcluster 15 second start
sudo pg_ctlcluster 15 third start
```
Создадим базы,таблицы и пользователей для репликации
```sh
sudo -u postgres psql -p 5432 -c "CREATE DATABASE d1;"
sudo -u postgres psql -p 5433 -c "CREATE DATABASE d2;"
sudo -u postgres psql -p 5434 -c "CREATE DATABASE d3;"
sudo -u postgres psql -p 5432 -d d1 -c "create table d1t1 (id int, txt text); create table d2t2(id int, txt text);"
sudo -u postgres psql -p 5433 -d d2 -c "create table d1t1 (id int, txt text); create table d2t2(id int, txt text);"
sudo -u postgres psql -p 5434 -d d3 -c "create table d1t1 (id int, txt text); create table d2t2(id int, txt text);"


sudo -u postgres psql -p 5432 -c "CREATE USER replicator WITH SUPERUSER PASSWORD 'password';"
sudo -u postgres psql -p 5433 -c "CREATE USER replicator WITH SUPERUSER PASSWORD 'password';"
sudo -u postgres psql -p 5434 -c "CREATE USER replicator WITH SUPERUSER PASSWORD 'password';"
```

Теперь создадим публикации и подписки
```sh
--main
CREATE PUBLICATION pub_d1t1 for table d1t1;
--second
CREATE PUBLICATION pub_d2t2 for table d2t2;

--main
CREATE SUBSCRIPTION second_d2t2 CONNECTION 'host=192.168.1.91 port=5433 user=replicator password=password dbname=d2' publication pub_d2t2 with (copy_data = true);
--second
CREATE SUBSCRIPTION main_d1t1 CONNECTION 'host=192.168.1.91 port=5432 user=replicator password=password dbname=d1' publication pub_d1t1 with (copy_data = true);
--third
CREATE SUBSCRIPTION main_c1t1 CONNECTION 'host=192.168.1.91 port=5432 user=replicator password=password dbname=d1' publication pub_d1t1 with (copy_data = true);
CREATE SUBSCRIPTION second_c2t2 CONNECTION 'host=192.168.1.91 port=5433 user=replicator password=password dbname=d2' publication pub_d2t2 with (copy_data = true);
```

Проверим репликацию
```sh
--main
INSERT INTO d1t1 (id, txt) VALUES (1, 'asd'), (2, 'dsa');
--second
INSERT INTO d2t2 (id, txt) VALUES (3, 'qwe'), (4, 'ewq');

d1=# SELECT id, txt from d2t2;
 id |  txt
----+-------
  3 | qwe
  4 | ewq
(2 rows)
d2=# SELECT id, txt from d1t1;
 id | txt
----+-----
  1 | asd
  2 | dsa
(2 rows)

d3=# select * from d1t1;
 id | txt
----+-----
  1 | asd
  2 | dsa
(2 rows)
d3=# select * from d2t2;
 id |  txt
----+-------
  3 | qwe
  4 | ewq
```
- Физическая репликация

Исправим pg_hba и wal_level
```sh
echo "host replication replicator $(hostname -I | awk '{print $1}')/32 scram-sha-256" | sudo tee -a /etc/postgresql/15/checksum/pg_hba.conf > /dev/null
sudo cp repl.conf /etc/postgresql/15/checksum/conf.d/
```

Удалим данные из кластера checksum и восстановим его из кластера third
```sh
sudo rm -rf /var/lib/postgresql/15/checksum/
sudo -u postgres pg_basebackup -p 5434 -R -D /var/lib/postgresql/15/checksum
```

Проверим, что кластер работает
```sh
sudo -u postgres psql -p 5435

postgres=# \c d3
You are now connected to database "d3" as user "postgres".
d3=# \dt
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | d1t1 | table | postgres
 public | d2t2 | table | postgres
(2 rows)
```

Проверим репликацию
```sh
--создадим таблицу test в кластере third
ubuntu@ubuntu:~$ sudo -u postgres psql -p 5434
psql (15.2 (Ubuntu 15.2-1.pgdg20.04+1))
Type "help" for help.

postgres=# \c d3
You are now connected to database "d3" as user "postgres".
d3=# create table test (id int);
CREATE TABLE
d3=# \q
could not save history to file "/home/postgres/.psql_history": No such file or directory

--проверим, что эта таблица появилась на кластере checksum
ubuntu@ubuntu:~$ sudo -u postgres psql -p 5435
psql (15.2 (Ubuntu 15.2-1.pgdg20.04+1))
Type "help" for help.

postgres=# \c d3
You are now connected to database "d3" as user "postgres".
d3=# \dt
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | d1t1 | table | postgres
 public | d2t2 | table | postgres
 public | test | table | postgres
(3 rows)
```