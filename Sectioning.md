# Postgre-DBA-2023-01

Создадим таблицу с секциями, взяв за основу boarding_passes

```sh
CREATE TABLE bookings.boarding_passes_p (
	ticket_no bpchar(13) NOT NULL,
	flight_id int4 NOT NULL,
	boarding_no int4 NOT NULL,
	seat_no varchar(4) NOT NULL,
	CONSTRAINT boarding_passes_p_flight_id_boarding_no_key UNIQUE (flight_id, boarding_no),
	CONSTRAINT boarding_passes_p_flight_id_seat_no_key UNIQUE (flight_id, seat_no),
	CONSTRAINT boarding_passes_p_pkey PRIMARY KEY (ticket_no, flight_id),
	CONSTRAINT boarding_passes_p_ticket_no_fkey FOREIGN KEY (ticket_no,flight_id) REFERENCES bookings.ticket_flights(ticket_no,flight_id)
) PARTITION BY RANGE (flight_id);
```
Создадим одну секцию на каждые 5000 рейсов
```sh
CREATE TABLE bookings.boarding_passes_p_0 PARTITION OF bookings.boarding_passes_p FOR VALUES FROM (0) TO (5000);
CREATE TABLE bookings.boarding_passes_p_1 PARTITION OF bookings.boarding_passes_p FOR VALUES FROM (5000) TO (10000);
CREATE TABLE bookings.boarding_passes_p_2 PARTITION OF bookings.boarding_passes_p FOR VALUES FROM (10000) TO (15000);
CREATE TABLE bookings.boarding_passes_p_3 PARTITION OF bookings.boarding_passes_p FOR VALUES FROM (15000) TO (20000);
CREATE TABLE bookings.boarding_passes_p_4 PARTITION OF bookings.boarding_passes_p FOR VALUES FROM (20000) TO (25000);
CREATE TABLE bookings.boarding_passes_p_5 PARTITION OF bookings.boarding_passes_p FOR VALUES FROM (25000) TO (30000);
CREATE TABLE bookings.boarding_passes_p_6 PARTITION OF bookings.boarding_passes_p FOR VALUES FROM (30000) TO (35000);
CREATE TABLE bookings.boarding_passes_p_7 PARTITION OF bookings.boarding_passes_p FOR VALUES FROM (35000) TO (40000);
CREATE TABLE bookings.boarding_passes_p_8 PARTITION OF bookings.boarding_passes_p FOR VALUES FROM (40000) TO (45000);
CREATE TABLE bookings.boarding_passes_p_9 PARTITION OF bookings.boarding_passes_p FOR VALUES FROM (45000) TO (50000);
CREATE TABLE bookings.boarding_passes_p_10 PARTITION OF bookings.boarding_passes_p FOR VALUES FROM (50000) TO (55000);
CREATE TABLE bookings.boarding_passes_p_11 PARTITION OF bookings.boarding_passes_p FOR VALUES FROM (55000) TO (60000);
CREATE TABLE bookings.boarding_passes_p_12 PARTITION OF bookings.boarding_passes_p FOR VALUES FROM (60000) TO (65000);
CREATE TABLE bookings.boarding_passes_p_13 PARTITION OF bookings.boarding_passes_p FOR VALUES FROM (65000) TO (70000);
```
Также добавим default секцию, чтобы не терять данные, если какую-то секцию не учли
```sh
CREATE TABLE bookings.boarding_passes_p_d PARTITION OF bookings.boarding_passes_p DEFAULT;
```
Заполним созданную таблицу данными из существующей
```sh
insert into boarding_passes_p select * from bookings.boarding_passes;
```
Проверка

Запрос по оригинальной таблице
```sh
explain analyze 
select * from boarding_passes bpp
where flight_id > 11000 and flight_id < 22000

Index Scan using boarding_passes_flight_id_seat_no_key on boarding_passes bpp  (cost=0.43..24717.46 rows=408298 width=25) (actual time=0.019..85.469 rows=411363 loops=1)
  Index Cond: ((flight_id > 11000) AND (flight_id < 22000))
Planning Time: 0.081 ms
Execution Time: 101.005 ms
```

Запрос по секционированной таблице
```sh
explain analyze
select * from boarding_passes_p bppd
where flight_id > 11000 and flight_id < 22000

Append  (cost=0.42..14932.12 rows=412336 width=25) (actual time=0.019..100.525 rows=411363 loops=1)
  ->  Index Scan using boarding_passes_p_2_flight_id_boarding_no_key on boarding_passes_p_2 bppd_1  (cost=0.42..5199.46 rows=132301 width=25) (actual time=0.019..25.815 rows=132366 loops=1)
        Index Cond: ((flight_id > 11000) AND (flight_id < 22000))
  ->  Seq Scan on boarding_passes_p_3 bppd_2  (cost=0.00..4743.49 rows=212166 width=25) (actual time=0.014..26.948 rows=212166 loops=1)
        Filter: ((flight_id > 11000) AND (flight_id < 22000))
  ->  Index Scan using boarding_passes_p_4_flight_id_seat_no_key on boarding_passes_p_4 bppd_3  (cost=0.42..2927.49 rows=67869 width=25) (actual time=0.023..15.298 rows=66831 loops=1)
        Index Cond: ((flight_id > 11000) AND (flight_id < 22000))
Planning Time: 0.254 ms
Execution Time: 116.014 ms
```

Преимущества в данном случае не заметил. Судя по времени выполнения запроса, секционирование проигрывает в скорости.
