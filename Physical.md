# Postgre-DBA-2023-01

перенесите содержимое /var/lib/postgres/14 в /mnt/data - mv /var/lib/postgresql/14 /mnt/data
попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 14 main start
напишите получилось или нет и почему

```sh
Не получилось, потому что файлы данных были перемещены, а постгрес не перенастроен
```
задание: найти конфигурационный параметр в файлах раположенных в /etc/postgresql/14/main который надо поменять и поменяйте его
напишите что и почему поменяли
```sh
Необходимо поменять data_directory в файле /etc/postgresql/14/main/postgresql.conf

```
Попытайтесь запустить кластер.
Напишите получилось или нет и почему.
```sh
ubuntu@ubuntu:~$ sudo -u postgres pg_ctlcluster 14 main start
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
  sudo systemctl start postgresql@14-main
ubuntu@ubuntu:~$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory    Log file
14  main    5432 online postgres /mnt/data/14/main /var/log/postgresql/postgresql-14-main.log
```

зайдите через через psql и проверьте содержимое ранее созданной таблицы
```sh
ubuntu@ubuntu:~$ sudo -u postgres psql
psql (14.7 (Ubuntu 14.7-1.pgdg20.04+1))
Type "help" for help.

postgres=# select * from test;
 c1
----
 1
(1 row)

postgres=# \q
```