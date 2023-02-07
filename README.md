# Postgre-DBA-2023-01

- создать новый проект в Google Cloud Platform, например postgres2022-, где yyyymm год, месяц вашего рождения (имя проекта должно быть уникально на уровне GCP), Яндекс облако или на любых ВМ, докере.

- Далее создать инстанс виртуальной машины с дефолтными параметрами
- Добавить свой ssh ключ в metadata ВМ
- Зайти удаленным ssh (первая сессия), **не забывайте про ssh-add**
- Поставить PostgreSQL
- Зайти вторым ssh (вторая сессия)
- Запустить везде psql из под пользователя postgres


Выключить auto commit
```sh
\SET AUTOCOMMIT OFF
```

Сделать в первой сессии новую таблицу и наполнить ее данными 
```sh
create table persons(id serial, first_name text, second_name text); insert into persons(first_name, second_name) values('ivan', 'ivanov'); insert into persons(first_name, second_name) values('petr', 'petrov'); commit;
```
Посмотреть текущий уровень изоляции: 
```sh
show transaction isolation level
```
Начать новую транзакцию в обоих сессиях с дефолтным (не меняя) уровнем изоляции

В первой сессии добавить новую запись 
```sh
insert into persons(first_name, second_name) values('sergey', 'sergeev');
```
Сделать во второй сессии
```sh 
select * from persons;
``` 

|  Видите ли вы новую запись и если да то почему? |  Нет, потому что autocommit выключен и транзакция ещё не завершена. |


Завершить первую транзакцию
```sh
commit;
```
Cделать во второй сессии
```sh 
select * from persons
```
 
Видите ли вы новую запись и если да то почему?

Да, потому что первая транзакция завершена успешно.


Завершите транзакцию во второй сессии

Начать новые но уже repeatable read транзации - 
```sh
set transaction isolation level repeatable read;
```
В первой сессии добавить новую запись 
```sh
insert into persons(first_name, second_name) values('sveta', 'svetova');
```
Сделать во второй сессии
```sh
select * from persons;
```
 
Видите ли вы новую запись и если да то почему?

Нет, не вижу

Завершить первую транзакцию 
```sh
commit;
```
Сделать во второй сессии
```sh
select * from persons
```
 
Видите ли вы новую запись и если да то почему?

Да, вижу.

Завершить вторую транзакцию

Сделать во второй сессии 
```sh
select * from persons;
```
 
видите ли вы новую запись и если да то почему? 



ДЗ сдаем в виде миниотчета в markdown в гите