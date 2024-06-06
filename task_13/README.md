# Задание №13 "Секционирование"

# Секционировать большую таблицу из демо базы flights.

## 0. Подготовка рабочей среды
Скачиваю демо базу данных demo-big.zip с [источника](https://postgrespro.ru/education/demodb) и делаю ее установку локально в свой докер контейнер pg-cluster:

```
[~]$ docker cp Downloads/demo-big/demo-big-20170815.sql pg-cluster:/demo-big-20170815.sql
Successfully copied 931MB to pg-cluster:/demo-big-20170815.sql
[~]$ docker exec -it pg-cluster psql -U postgres -c "CREATE DATABASE flightsdb;"
CREATE DATABASE
[~]$ docker exec -it pg-cluster psql -U postgres -d flightsdb -f /demo-big-20170815.sql
SET
SET
SET
SET
SET
SET
SET
SET
psql:/demo-big-20170815.sql:17: ERROR:  database "demo" does not exist
CREATE DATABASE
You are now connected to database "demo" as user "postgres".
SET
SET
SET
SET
SET
SET
SET
SET
CREATE SCHEMA
COMMENT
CREATE EXTENSION
COMMENT
SET
CREATE FUNCTION
CREATE FUNCTION
COMMENT
SET
SET
CREATE TABLE
COMMENT
COMMENT
COMMENT
COMMENT
CREATE VIEW
COMMENT
COMMENT
COMMENT
COMMENT
CREATE TABLE
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
CREATE VIEW
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
CREATE TABLE
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
CREATE TABLE
COMMENT
COMMENT
COMMENT
COMMENT
CREATE TABLE
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
CREATE SEQUENCE
ALTER SEQUENCE
CREATE VIEW
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
CREATE VIEW
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
CREATE TABLE
COMMENT
COMMENT
COMMENT
COMMENT
CREATE TABLE
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
CREATE TABLE
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
ALTER TABLE
COPY 9
COPY 104
COPY 7925812
COPY 2111110
COPY 214867
COPY 1339
COPY 8391852
COPY 2949857
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER DATABASE
ALTER DATABASE
[~]$ docker exec -it pg-cluster psql -U postgres -d flightsdb
psql (15.6 (Debian 15.6-1.pgdg120+2))
Type "help" for help.

flightsdb=#
[~]$ docker exec -it pg-cluster psql -U postgres -d flightsdb
psql (15.6 (Debian 15.6-1.pgdg120+2))
Type "help" for help.

flightsdb=# \dt
Did not find any relations.
flightsdb=# exit

```
Понимаю что база создалась под другим именем, а именно "demo":
```
[~]$ docker exec -it pg-cluster psql -U postgres -d demo
psql (15.6 (Debian 15.6-1.pgdg120+2))
Type "help" for help.

demo=# \dt
               List of relations
  Schema  |      Name       | Type  |  Owner
----------+-----------------+-------+----------
 bookings | aircrafts_data  | table | postgres
 bookings | airports_data   | table | postgres
 bookings | boarding_passes | table | postgres
 bookings | bookings        | table | postgres
 bookings | flights         | table | postgres
 bookings | seats           | table | postgres
 bookings | ticket_flights  | table | postgres
 bookings | tickets         | table | postgres
(8 rows)


```


## 1. Выбор самой крупной таблицы
Теперь через запрос проверяю размеры всех таблиц:
```
demo=# SELECT
    table_name,
    pg_size_pretty(pg_total_relation_size(quote_ident(table_name))) AS size
FROM
    information_schema.tables
WHERE
    table_schema = 'bookings'
ORDER BY
    pg_total_relation_size(quote_ident(table_name)) DESC;
   table_name    |  size
-----------------+---------
 boarding_passes | 1102 MB
 ticket_flights  | 871 MB
 tickets         | 475 MB
 bookings        | 150 MB
 flights         | 32 MB
 seats           | 144 kB
 airports_data   | 72 kB
 aircrafts_data  | 32 kB
 aircrafts       | 0 bytes
 routes          | 0 bytes
 flights_v       | 0 bytes
 airports        | 0 bytes
(12 rows)

```
где самая большая - boarding_passes - буду секционировать ее.

## 2. Секционирование
Теперь, чтобы секционировать эту таблицу,буду использовать диапазонное секционирование (range partitioning) по значению колонки boarding_no.
Проверяю структуру таблицы:
```
demo=# \d+ boarding_passes
                                                  Table "bookings.boarding_passes"
   Column    |         Type         | Collation | Nullable | Default | Storage  | Compression | Stats target |     Description
-------------+----------------------+-----------+----------+---------+----------+-------------+--------------+----------------------
 ticket_no   | character(13)        |           | not null |         | extended |             |              | Ticket number
 flight_id   | integer              |           | not null |         | plain    |             |              | Flight ID
 boarding_no | integer              |           | not null |         | plain    |             |              | Boarding pass number
 seat_no     | character varying(4) |           | not null |         | extended |             |              | Seat number
Indexes:
    "boarding_passes_pkey" PRIMARY KEY, btree (ticket_no, flight_id)
    "boarding_passes_flight_id_boarding_no_key" UNIQUE CONSTRAINT, btree (flight_id, boarding_no)
    "boarding_passes_flight_id_seat_no_key" UNIQUE CONSTRAINT, btree (flight_id, seat_no)
Foreign-key constraints:
    "boarding_passes_ticket_no_fkey" FOREIGN KEY (ticket_no, flight_id) REFERENCES ticket_flights(ticket_no, flight_id)
Access method: heap
```
и проверяю иные данные о данной таблице:
```
demo=# \d+ bookings.boarding_passes
                                                  Table "bookings.boarding_passes"
   Column    |         Type         | Collation | Nullable | Default | Storage  | Compression | Stats target |     Description
-------------+----------------------+-----------+----------+---------+----------+-------------+--------------+----------------------
 ticket_no   | character(13)        |           | not null |         | extended |             |              | Ticket number
 flight_id   | integer              |           | not null |         | plain    |             |              | Flight ID
 boarding_no | integer              |           | not null |         | plain    |             |              | Boarding pass number
 seat_no     | character varying(4) |           | not null |         | extended |             |              | Seat number
Indexes:
    "boarding_passes_pkey" PRIMARY KEY, btree (ticket_no, flight_id)
    "boarding_passes_flight_id_boarding_no_key" UNIQUE CONSTRAINT, btree (flight_id, boarding_no)
    "boarding_passes_flight_id_seat_no_key" UNIQUE CONSTRAINT, btree (flight_id, seat_no)
Foreign-key constraints:
    "boarding_passes_ticket_no_fkey" FOREIGN KEY (ticket_no, flight_id) REFERENCES ticket_flights(ticket_no, flight_id)
Access method: heap

demo=# SELECT COUNT(*) FROM bookings.boarding_passes;
  count
---------
 7925812
(1 row)

demo=# SELECT pg_size_pretty(pg_total_relation_size('bookings.boarding_passes')) AS size;
  size
---------
 1102 MB
(1 row)

```
Создаю новую секционированную таблицу:
```
demo=# CREATE TABLE bookings.boarding_passes_partitioned (
    ticket_no VARCHAR(13),
    flight_id INTEGER,
    boarding_no INTEGER,
    seat_no VARCHAR(4)
) PARTITION BY RANGE (boarding_no);
CREATE TABLE
```
Я хочу создать партиции по диапазону номеров посадочных талонов (boarding_no), проверяю максимальное значение которое бывает в такой колонке:
```
demo=# SELECT MAX(boarding_no) FROM bookings.boarding_passes;
 max
-----
 381
(1 row)

```

381 - буду строить диапазоны по 100 штук:
```
demo=# CREATE TABLE bookings.boarding_passes_part1 PARTITION OF bookings.boarding_passes_partitioned
FOR VALUES FROM (1) TO (100);
CREATE TABLE
demo=# CREATE TABLE bookings.boarding_passes_part2 PARTITION OF bookings.boarding_passes_partitioned
FOR VALUES FROM (100) TO (200);
CREATE TABLE
demo=# CREATE TABLE bookings.boarding_passes_part3 PARTITION OF bookings.boarding_passes_partitioned
FOR VALUES FROM (200) TO (300);
CREATE TABLE
demo=# CREATE TABLE bookings.boarding_passes_part4 PARTITION OF bookings.boarding_passes_partitioned
FOR VALUES FROM (300) TO (400);
CREATE TABLE
```

Теперь делаю миграцию данных из старой таблицы в новую секционированную таблицу:
```
demo=# INSERT INTO bookings.boarding_passes_partitioned (ticket_no, flight_id, boarding_no, seat_no)
SELECT ticket_no, flight_id, boarding_no, seat_no FROM bookings.boarding_passes;
INSERT 0 7925812
demo=# DROP TABLE bookings.boarding_passes;
DROP TABLE
demo=# ALTER TABLE bookings.boarding_passes_partitioned RENAME TO boarding_passes;
ALTER TABLE
demo=# SELECT * FROM bookings.boarding_passes LIMIT 10;
   ticket_no   | flight_id | boarding_no | seat_no
---------------+-----------+-------------+---------
 0005435189093 |    198393 |           1 | 27G
 0005435189119 |    198393 |           2 | 2D
 0005435189096 |    198393 |           3 | 18E
 0005435189117 |    198393 |           4 | 31B
 0005432208788 |    198393 |           5 | 28C
 0005435189151 |    198393 |           6 | 32A
 0005433655456 |    198393 |           7 | 31J
 0005435189129 |    198393 |           8 | 30C
 0005435629876 |    198393 |           9 | 30E
 0005435189100 |    198393 |          10 | 30F
(10 rows)

```
По сути пересоздал оригинальную таблицу boarding_passes, проверил, данные выводятся.
Теперь проверяю размер каждой партиции:
```
demo=# SELECT
    tablename,
    pg_size_pretty(pg_total_relation_size(quote_ident(tablename))) AS size
FROM
    pg_tables
WHERE
    schemaname = 'bookings'
    AND tablename LIKE 'boarding_passes_part%';
       tablename       |  size
-----------------------+---------
 boarding_passes_part1 | 379 MB
 boarding_passes_part2 | 54 MB
 boarding_passes_part3 | 19 MB
 boarding_passes_part4 | 3928 kB
(4 rows)

```
и проверяю иные параметры:
```
demo=# \d+ bookings.boarding_passes
                                        Partitioned table "bookings.boarding_passes"
   Column    |         Type          | Collation | Nullable | Default | Storage  | Compression | Stats target | Description
-------------+-----------------------+-----------+----------+---------+----------+-------------+--------------+-------------
 ticket_no   | character varying(13) |           |          |         | extended |             |              |
 flight_id   | integer               |           |          |         | plain    |             |              |
 boarding_no | integer               |           |          |         | plain    |             |              |
 seat_no     | character varying(4)  |           |          |         | extended |             |              |
Partition key: RANGE (boarding_no)
Partitions: boarding_passes_part1 FOR VALUES FROM (1) TO (100),
            boarding_passes_part2 FOR VALUES FROM (100) TO (200),
            boarding_passes_part3 FOR VALUES FROM (200) TO (300),
            boarding_passes_part4 FOR VALUES FROM (300) TO (400)

demo=# SELECT COUNT(*) FROM bookings.boarding_passes;
  count
---------
 7925812
(1 row)

```
вижу что данные в норме.

Таким образом:
- Количество строк в новой таблице совпадает с количеством строк в оригинальной таблице.
- Общий размер данных уменьшился, что может быть связано с различиями в управлении хранением и индексами.
- Партиции распределены по размеру в зависимости от диапазонов boarding_no