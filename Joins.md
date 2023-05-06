# Postgre-DBA-2023-01

Прямое соединение

Выводит город,аэропорт и количество вылетов
```sh
select city,
    airport_name,
    count(status)
from airports_data a,
    flights f
where a.airport_code = f.departure_airport
group by city,
    airport_name;
```
Левосторонне соединение

Кол-во вылетов из аэропорта за всё время
```sh
select airport_name,
    count(f.flight_id)
from airports_data a
    left join flights f on a.airport_code = f.departure_airport
group by 1
order by 2 desc;
```
Кросс соединение таблиц

Показывает куда можно улететь из каждого аэропорта
```sh
select a.airport_name,
    a2.airport_name
from airports_data a
    cross join airports a2
where a.airport_code <> a2.airport_code;
```
Полное соединение

Показывает сколько мест в конкретной модели самолёта
```sh
select a.model,
    count(s.seat_no)
from seats s
    full join aircrafts a on a.aircraft_code = s.aircraft_code
group by 1;
```
