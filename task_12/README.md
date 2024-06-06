# Задание №12 "Работа с join'ами, статистикой"

## 0. Подготовка рабочей среды
Буду использовать базу данных созданную для [ДЗ 11](https://github.com/Alejaldo/otus_psql/tree/master/task_11) локально, поэтому также запускаю свой докер контейнер с базой `chinook_auto_increment`:
```
[~]$ docker start pg-cluster
pg-cluster
[~]$ docker exec -it pg-cluster psql -U postgres -d chinook_auto_increment
psql (15.6 (Debian 15.6-1.pgdg120+2))
Type "help" for help.

chinook_auto_increment=#

```
## 1. Реализовать прямое соединение двух или более таблиц
Прямое соединение (INNER JOIN) объединяет строки из двух или более таблиц на основе условия соединения, где только совпадающие строки обеих таблиц включаются в результат.\
Соединю таблицы invoice и customer по столбцу customer_id:
```
chinook_auto_increment=# SELECT
    c.customer_id,
    c.first_name,
    c.last_name,
    i.invoice_id,
    i.total
FROM
    customer c
INNER JOIN
    invoice i ON c.customer_id = i.customer_id;

 customer_id | first_name |  last_name   | invoice_id | total
-------------+------------+--------------+------------+-------
           2 | Leonie     | Köhler       |          1 |  1.98
           4 | Bjørn      | Hansen       |          2 |  3.96
           8 | Daan       | Peeters      |          3 |  5.94
          14 | Mark       | Philips      |          4 |  8.91
          23 | John       | Gordon       |          5 | 13.86
          37 | Fynn       | Zimmermann   |          6 |  0.99
          38 | Niklas     | Schröder     |          7 |  1.98
          40 | Dominique  | Lefebvre     |          8 |  1.98
          42 | Wyatt      | Girard       |          9 |  3.96
          46 | Hugh       | O'Reilly     |         10 |  5.94
          52 | Emma       | Jones        |         11 |  8.91
           2 | Leonie     | Köhler       |         12 | 13.86
          16 | Frank      | Harris       |         13 |  0.99
          17 | Jack       | Smith        |         14 |  1.98
          19 | Tim        | Goyer        |         15 |  1.98
          21 | Kathy      | Chase        |         16 |  3.96
          25 | Victor     | Stevens      |         17 |  5.94
          31 | Martha     | Silk         |         18 |  8.91
          40 | Dominique  | Lefebvre     |         19 | 13.86
          54 | Steve      | Murray       |         20 |  0.99

          ...
          56 | Diego      | Gutiérrez    |        403 |  8.91
           6 | Helena     | Holý         |        404 | 25.86
          20 | Dan        | Miller       |        405 |  0.99
          21 | Kathy      | Chase        |        406 |  1.98
          23 | John       | Gordon       |        407 |  1.98
          25 | Victor     | Stevens      |        408 |  3.96
          29 | Robert     | Brown        |        409 |  5.94
          35 | Madalena   | Sampaio      |        410 |  8.91
          44 | Terhi      | Hämäläinen   |        411 | 13.86
          58 | Manoj      | Pareek       |        412 |  1.99
(412 rows)
```
где:
- `SELECT c.customer_id, c.first_name, c.last_name, i.invoice_id, i.total`: Выбираются столбцы customer_id, first_name, last_name из таблицы customer, а также invoice_id и total из таблицы invoice.
- `FROM customer c`: Основная таблица для запроса - customer, обозначенная как c.
- `INNER JOIN invoice i ON c.customer_id = i.customer_id`: Выполняется внутреннее соединение таблицы customer с таблицей invoice по условию совпадения customer_id.

Анализ через команду EXPLAIN:
```
chinook_auto_increment=# EXPLAIN ANALYZE
SELECT
    c.customer_id,
    c.first_name,
    c.last_name,
    i.invoice_id,
    i.total
FROM
    customer c
INNER JOIN
    invoice i ON c.customer_id = i.customer_id;
                                                    QUERY PLAN
-------------------------------------------------------------------------------------------------------------------
 Hash Join  (cost=3.33..14.61 rows=412 width=28) (actual time=0.031..0.138 rows=412 loops=1)
   Hash Cond: (i.customer_id = c.customer_id)
   ->  Seq Scan on invoice i  (cost=0.00..10.12 rows=412 width=14) (actual time=0.005..0.029 rows=412 loops=1)
   ->  Hash  (cost=2.59..2.59 rows=59 width=18) (actual time=0.019..0.020 rows=59 loops=1)
         Buckets: 1024  Batches: 1  Memory Usage: 11kB
         ->  Seq Scan on customer c  (cost=0.00..2.59 rows=59 width=18) (actual time=0.003..0.009 rows=59 loops=1)
 Planning Time: 0.210 ms
 Execution Time: 0.161 ms
(8 rows)
```
## 2. Реализовать левостороннее (или правостороннее) соединение двух или более таблиц
Левостороннее соединение (LEFT JOIN) возвращает все строки из левой таблицы и совпадающие строки из правой таблицы. Если совпадений нет, результат содержит NULL для столбцов из правой таблицы.\
Соединю таблицы customer и invoice с помощью левостороннего соединения по столбцу customer_id:
```
chinook_auto_increment=# SELECT
    ar.artist_id,
    ar.name AS artist_name,
    al.album_id,
    al.title AS album_title
FROM
    artist ar
LEFT JOIN
    album al ON ar.artist_id = al.artist_id;


 artist_id |                                      artist_name                                      | album_id |                                           album_title

-----------+---------------------------------------------------------------------------------------+----------+--------------------------------------------------------------------------------------------
-----
         1 | AC/DC                                                                                 |        1 | For Those About To Rock We Salute You
         2 | Accept                                                                                |        2 | Balls to the Wall
         2 | Accept                                                                                |        3 | Restless and Wild
         1 | AC/DC                                                                                 |        4 | Let There Be Rock
         3 | Aerosmith                                                                             |        5 | Big Ones
         4 | Alanis Morissette                                                                     |        6 | Jagged Little Pill
         5 | Alice In Chains                                                                       |        7 | Facelift
         ...
        35 | Pedro Luís & A Parede                                                                 |          |
       173 | Matisyahu                                                                             |          |
       188 | Mundo Livre S/A                                                                       |          |
        63 | Santana Feat. Lauryn Hill & Cee-Lo                                                    |          |
       183 | Gustavo & Andres Veiga & Salazar                                                      |          |
(418 rows)

```
где:
- `SELECT ar.artist_id, ar.name AS artist_name, al.album_id, al.title AS album_title`: Выбираю столбцы artist_id и name из таблицы artist, а также album_id и title из таблицы album.
- `FROM artist ar`: Основная таблица для запроса - artist, обозначенная как ar.
- `LEFT JOIN album al ON ar.artist_id = al.artist_id`: Выполняю левостороннее соединение таблицы artist с таблицей album по условию совпадения artist_id.


EXPLAIN анализ:
```
chinook_auto_increment=# EXPLAIN ANALYZE
SELECT
    ar.artist_id,
    ar.name AS artist_name,
    al.album_id,
    al.title AS album_title
FROM
    artist ar
LEFT JOIN
    album al ON ar.artist_id = al.artist_id;
                                                     QUERY PLAN
--------------------------------------------------------------------------------------------------------------------
 Hash Right Join  (cost=8.19..15.58 rows=347 width=52) (actual time=0.063..0.187 rows=418 loops=1)
   Hash Cond: (al.artist_id = ar.artist_id)
   ->  Seq Scan on album al  (cost=0.00..6.47 rows=347 width=31) (actual time=0.004..0.024 rows=347 loops=1)
   ->  Hash  (cost=4.75..4.75 rows=275 width=25) (actual time=0.054..0.054 rows=275 loops=1)
         Buckets: 1024  Batches: 1  Memory Usage: 24kB
         ->  Seq Scan on artist ar  (cost=0.00..4.75 rows=275 width=25) (actual time=0.008..0.024 rows=275 loops=1)
 Planning Time: 0.181 ms
 Execution Time: 0.213 ms
(8 rows)
```
## 3. Реализовать кросс соединение двух или более таблиц
Кросс соединение (CROSS JOIN) возвращает декартово произведение строк двух таблиц. Это означает, что каждая строка из первой таблицы будет соединена с каждой строкой из второй таблицы.\
```
chinook_auto_increment=# SELECT
    c.customer_id,
    c.first_name,
    c.last_name,
    i.invoice_id,
    i.total
FROM
    customer c
CROSS JOIN
    invoice i;

 customer_id | first_name |  last_name   | invoice_id | total
-------------+------------+--------------+------------+-------
           1 | Luís       | Gonçalves    |          1 |  1.98
           2 | Leonie     | Köhler       |          1 |  1.98
           3 | François   | Tremblay     |          1 |  1.98
           4 | Bjørn      | Hansen       |          1 |  1.98
           5 | František  | Wichterlová  |          1 |  1.98
           6 | Helena     | Holý         |          1 |  1.98
           7 | Astrid     | Gruber       |          1 |  1.98
           8 | Daan       | Peeters      |          1 |  1.98
           9 | Kara       | Nielsen      |          1 |  1.98
          10 | Eduardo    | Martins      |          1 |  1.98
          11 | Alexandre  | Rocha        |          1 |  1.98
          12 | Roberto    | Almeida      |          1 |  1.98
          13 | Fernanda   | Ramos        |          1 |  1.98
          14 | Mark       | Philips      |          1 |  1.98
          15 | Jennifer   | Peterson     |          1 |  1.98
          16 | Frank      | Harris       |          1 |  1.98
          17 | Jack       | Smith        |          1 |  1.98
          18 | Michelle   | Brooks       |          1 |  1.98
          19 | Tim        | Goyer        |          1 |  1.98
          20 | Dan        | Miller       |          1 |  1.98
          21 | Kathy      | Chase        |          1 |  1.98
          22 | Heather    | Leacock      |          1 |  1.98
          23 | John       | Gordon       |          1 |  1.98
          24 | Frank      | Ralston      |          1 |  1.98
          25 | Victor     | Stevens      |          1 |  1.98
...
          53 | Phil       | Hughes       |        412 |  1.99
          54 | Steve      | Murray       |        412 |  1.99
          55 | Mark       | Taylor       |        412 |  1.99
          56 | Diego      | Gutiérrez    |        412 |  1.99
          57 | Luis       | Rojas        |        412 |  1.99
          58 | Manoj      | Pareek       |        412 |  1.99
          59 | Puja       | Srivastava   |        412 |  1.99
(24308 rows)

chinook_auto_increment=# EXPLAIN ANALYZE
SELECT
    c.customer_id,
    c.first_name,
    c.last_name,
    i.invoice_id,
    i.total
FROM
    customer c
CROSS JOIN
    invoice i;
                                                    QUERY PLAN
-------------------------------------------------------------------------------------------------------------------
 Nested Loop  (cost=0.00..316.71 rows=24308 width=28) (actual time=0.014..3.679 rows=24308 loops=1)
   ->  Seq Scan on invoice i  (cost=0.00..10.12 rows=412 width=10) (actual time=0.006..0.046 rows=412 loops=1)
   ->  Materialize  (cost=0.00..2.88 rows=59 width=18) (actual time=0.000..0.002 rows=59 loops=412)
         ->  Seq Scan on customer c  (cost=0.00..2.59 rows=59 width=18) (actual time=0.004..0.019 rows=59 loops=1)
 Planning Time: 0.092 ms
 Execution Time: 4.292 ms
(6 rows)

```
здесь `CROSS JOIN invoice i` выполняет кросс соединение таблицы customer с таблицей invoice.\

```
chinook_auto_increment=# SELECT COUNT(*) FROM customer;
 count
-------
    59
(1 row)

chinook_auto_increment=# SELECT COUNT(*) FROM invoice;
 count
-------
   412
(1 row)

chinook_auto_increment=# SELECT
    (SELECT COUNT(*) FROM customer) * (SELECT COUNT(*) FROM invoice) AS total_product;
 total_product
---------------
         24308
(1 row)
```
Перемножив количество строк таблиц customer и invoice вижу что 24308, как и в CROSS JOIN таблице, что и ожидалось.
## 4. Реализовать полное соединение двух или более таблиц
Полное соединение (FULL JOIN) объединяет строки из двух таблиц, возвращая все строки, где есть совпадения в одной из таблиц, а также строки из обеих таблиц, где нет совпадений. Если совпадений нет, результат содержит NULL для столбцов из другой таблицы.


```
chinook_auto_increment=# SELECT
    c.customer_id,
    c.first_name,
    c.last_name,
    i.invoice_id,
    i.total
FROM
    customer c
FULL JOIN
    invoice i ON c.customer_id = i.customer_id;

 customer_id | first_name |  last_name   | invoice_id | total
-------------+------------+--------------+------------+-------
           2 | Leonie     | Köhler       |          1 |  1.98
           4 | Bjørn      | Hansen       |          2 |  3.96
           8 | Daan       | Peeters      |          3 |  5.94
          14 | Mark       | Philips      |          4 |  8.91
          23 | John       | Gordon       |          5 | 13.86
          37 | Fynn       | Zimmermann   |          6 |  0.99
          38 | Niklas     | Schröder     |          7 |  1.98
          40 | Dominique  | Lefebvre     |          8 |  1.98
          42 | Wyatt      | Girard       |          9 |  3.96
          46 | Hugh       | O'Reilly     |         10 |  5.94
          52 | Emma       | Jones        |         11 |  8.91
           2 | Leonie     | Köhler       |         12 | 13.86
          16 | Frank      | Harris       |         13 |  0.99
          17 | Jack       | Smith        |         14 |  1.98
          19 | Tim        | Goyer        |         15 |  1.98
          21 | Kathy      | Chase        |         16 |  3.96
          25 | Victor     | Stevens      |         17 |  5.94
          31 | Martha     | Silk         |         18 |  8.91
          40 | Dominique  | Lefebvre     |         19 | 13.86
          54 | Steve      | Murray       |         20 |  0.99
          ...
          20 | Dan        | Miller       |        405 |  0.99
          21 | Kathy      | Chase        |        406 |  1.98
          23 | John       | Gordon       |        407 |  1.98
          25 | Victor     | Stevens      |        408 |  3.96
          29 | Robert     | Brown        |        409 |  5.94
          35 | Madalena   | Sampaio      |        410 |  8.91
          44 | Terhi      | Hämäläinen   |        411 | 13.86
          58 | Manoj      | Pareek       |        412 |  1.99
(412 rows)

chinook_auto_increment=# EXPLAIN ANALYZE
SELECT
    c.customer_id,
    c.first_name,
    c.last_name,
    i.invoice_id,
    i.total
FROM
    customer c
FULL JOIN
    invoice i ON c.customer_id = i.customer_id;
                                                    QUERY PLAN
-------------------------------------------------------------------------------------------------------------------
 Hash Full Join  (cost=3.33..14.61 rows=412 width=28) (actual time=0.032..0.142 rows=412 loops=1)
   Hash Cond: (i.customer_id = c.customer_id)
   ->  Seq Scan on invoice i  (cost=0.00..10.12 rows=412 width=14) (actual time=0.002..0.027 rows=412 loops=1)
   ->  Hash  (cost=2.59..2.59 rows=59 width=18) (actual time=0.024..0.025 rows=59 loops=1)
         Buckets: 1024  Batches: 1  Memory Usage: 11kB
         ->  Seq Scan on customer c  (cost=0.00..2.59 rows=59 width=18) (actual time=0.008..0.015 rows=59 loops=1)
 Planning Time: 0.089 ms
 Execution Time: 0.166 ms
(8 rows)
```
где `FULL JOIN invoice i ON c.customer_id = i.customer_id` выполняет полное соединение таблицы customer с таблицей invoice по условию совпадения customer_id.
## 5. Реализовать запрос, в котором будут использованы разные типы соединений
Для демонстрации запроса, использующего разные типы соединений, создам запрос, который включает INNER JOIN, LEFT JOIN и FULL JOIN и буду использовать таблицы customer, invoice, invoice_line и track.
```
chinook_auto_increment=# SELECT
    c.customer_id,
    c.first_name,
    c.last_name,
    i.invoice_id,
    i.total,
    il.invoice_line_id,
    t.track_id,
    t.name AS track_name
FROM
    customer c
INNER JOIN
    invoice i ON c.customer_id = i.customer_id
LEFT JOIN
    invoice_line il ON i.invoice_id = il.invoice_id
FULL JOIN
    track t ON il.track_id = t.track_id;

 customer_id | first_name |  last_name   | invoice_id | total | invoice_line_id | track_id |                                                         track_name
-------------+------------+--------------+------------+-------+-----------------+----------+-----------------------------------------------------------------------------------------------------------------------------
           2 | Leonie     | Köhler       |          1 |  1.98 |               1 |        2 | Balls to the Wall
           2 | Leonie     | Köhler       |          1 |  1.98 |               2 |        4 | Restless and Wild
           4 | Bjørn      | Hansen       |          2 |  3.96 |               3 |        6 | Put The Finger On You
           4 | Bjørn      | Hansen       |          2 |  3.96 |               4 |        8 | Inject The Venom
           4 | Bjørn      | Hansen       |          2 |  3.96 |               5 |       10 | Evil Walks
           4 | Bjørn      | Hansen       |          2 |  3.96 |               6 |       12 | Breaking The Rules
           8 | Daan       | Peeters      |          3 |  5.94 |               7 |       16 | Dog Eat Dog
           8 | Daan       | Peeters      |          3 |  5.94 |               8 |       20 | Overdose
           8 | Daan       | Peeters      |          3 |  5.94 |               9 |       24 | Love In An Elevator
           8 | Daan       | Peeters      |          3 |  5.94 |              10 |       28 | Janie's Got A Gun
           8 | Daan       | Peeters      |          3 |  5.94 |              11 |       32 | Deuces Are Wild
           8 | Daan       | Peeters      |          3 |  5.94 |              12 |       36 | Angel
          14 | Mark       | Philips      |          4 |  8.91 |              13 |       42 | Right Through You
          14 | Mark       | Philips      |          4 |  8.91 |              14 |       48 | Not The Doctor
          14 | Mark       | Philips      |          4 |  8.91 |              15 |       54 | Bleed The Freak
          14 | Mark       | Philips      |          4 |  8.91 |              16 |       60 | Confusion
          14 | Mark       | Philips      |          4 |  8.91 |              17 |       66 | Por Causa De Você
          14 | Mark       | Philips      |          4 |  8.91 |              18 |       72 | Angela
          14 | Mark       | Philips      |          4 |  8.91 |              19 |       78 | Master Of Puppets
...
             |            |              |            |       |                 |      908 | I Can't Stand It
             |            |              |            |       |                 |     3345 | The Other Woman
             |            |              |            |       |                 |      538 | Desculpe Babe
             |            |              |            |       |                 |     2460 | Preto Damião
             |            |              |            |       |                 |     3448 | Lamentations of Jeremiah, First Set \ Incipit Lamentatio
             |            |              |            |       |                 |     2795 | Bichos Escrotos (Vinheta)
             |            |              |            |       |                 |     2502 | The Everlasting Gaze
             |            |              |            |       |                 |     2824 | Torn
(3759 rows)
```
где вижу:
- Полные строки данных: Строки, где имеются полные данные от клиента до трека, указывают на успешное выполнение всех JOIN операций. В этих строках INNER JOIN, LEFT JOIN и FULL JOIN нашли соответствия во всех таблицах.
- Строки с NULL значениями в столбцах invoice_line_id и track_id: Это результат выполнения LEFT JOIN. Строки, у которых нет совпадений в таблице invoice_line, содержат NULL в столбцах invoice_line_id и track_id.
- Строки с NULL значениями в столбцах customer_id, first_name, last_name, invoice_id и total: Это результат выполнения FULL JOIN. Строки, у которых нет совпадений в таблице invoice и customer, содержат NULL в этих столбцах.

```
chinook_auto_increment=# EXPLAIN ANALYZE
SELECT
    c.customer_id,
    c.first_name,
    c.last_name,
    i.invoice_id,
    i.total,
    il.invoice_line_id,
    t.track_id,
    t.name AS track_name
FROM
    customer c
INNER JOIN
    invoice i ON c.customer_id = i.customer_id
LEFT JOIN
    invoice_line il ON i.invoice_id = il.invoice_id
FULL JOIN
    track t ON il.track_id = t.track_id;
                                                            QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------
 Hash Full Join  (cost=142.41..197.94 rows=3503 width=52) (actual time=1.616..4.384 rows=3759 loops=1)
   Hash Cond: (il.track_id = t.track_id)
   ->  Hash Join  (cost=18.60..68.24 rows=2240 width=36) (actual time=0.237..1.894 rows=2240 loops=1)
         Hash Cond: (i.customer_id = c.customer_id)
         ->  Hash Right Join  (cost=15.27..58.61 rows=2240 width=22) (actual time=0.199..1.193 rows=2240 loops=1)
               Hash Cond: (il.invoice_id = i.invoice_id)
               ->  Seq Scan on invoice_line il  (cost=0.00..37.40 rows=2240 width=12) (actual time=0.003..0.204 rows=2240 loops=1)
               ->  Hash  (cost=10.12..10.12 rows=412 width=14) (actual time=0.190..0.191 rows=412 loops=1)
                     Buckets: 1024  Batches: 1  Memory Usage: 28kB
                     ->  Seq Scan on invoice i  (cost=0.00..10.12 rows=412 width=14) (actual time=0.004..0.100 rows=412 loops=1)
         ->  Hash  (cost=2.59..2.59 rows=59 width=18) (actual time=0.032..0.033 rows=59 loops=1)
               Buckets: 1024  Batches: 1  Memory Usage: 11kB
               ->  Seq Scan on customer c  (cost=0.00..2.59 rows=59 width=18) (actual time=0.006..0.017 rows=59 loops=1)
   ->  Hash  (cost=80.03..80.03 rows=3503 width=20) (actual time=1.374..1.374 rows=3503 loops=1)
         Buckets: 4096  Batches: 1  Memory Usage: 214kB
         ->  Seq Scan on track t  (cost=0.00..80.03 rows=3503 width=20) (actual time=0.006..0.635 rows=3503 loops=1)
 Planning Time: 0.383 ms
 Execution Time: 4.574 ms
(18 rows)

```
где:
- Hash Full Join: Вершина плана выполнения, которая выполняет FULL JOIN между invoice_line и track. Это создает полный набор результатов, включая строки с отсутствующими совпадениями.
- Hash Join: Внутреннее соединение между invoice и customer. Строится хэш-таблица для более быстрого выполнения соединения.
- Hash Right Join: Левостороннее соединение между invoice и invoice_line. Используется хэш для соединения строк на основе invoice_id.
- Seq Scan: Последовательное сканирование таблиц для извлечения всех строк. Это выполняется для каждой таблицы: invoice_line, invoice, customer, track.
## 6. Сделать комментарии на каждый запрос
Комментарии даны выше для каждого из шагов.
## 7. К работе приложить структуру таблиц, для которых выполнялись соединения
Теперь покажу список таблиц, их структуру:
```
chinook_auto_increment=# \dt
             List of relations
 Schema |      Name      | Type  |  Owner
--------+----------------+-------+----------
 public | album          | table | postgres
 public | artist         | table | postgres
 public | customer       | table | postgres
 public | employee       | table | postgres
 public | genre          | table | postgres
 public | invoice        | table | postgres
 public | invoice_line   | table | postgres
 public | media_type     | table | postgres
 public | playlist       | table | postgres
 public | playlist_track | table | postgres
 public | track          | table | postgres
(11 rows)

chinook_auto_increment=# \d+ album
                                                              Table "public.album"
  Column   |          Type          | Collation | Nullable |           Default            | Storage  | Compression | Stats target | Description
-----------+------------------------+-----------+----------+------------------------------+----------+-------------+--------------+-------------
 album_id  | integer                |           | not null | generated always as identity | plain    |             |              |
 title     | character varying(160) |           | not null |                              | extended |             |              |
 artist_id | integer                |           | not null |                              | plain    |             |              |
Indexes:
    "album_pkey" PRIMARY KEY, btree (album_id)
    "album_artist_id_idx" btree (artist_id)
Foreign-key constraints:
    "album_artist_id_fkey" FOREIGN KEY (artist_id) REFERENCES artist(artist_id)
Referenced by:
    TABLE "track" CONSTRAINT "track_album_id_fkey" FOREIGN KEY (album_id) REFERENCES album(album_id)
Access method: heap

chinook_auto_increment=# \d+ artist
                                                             Table "public.artist"
  Column   |          Type          | Collation | Nullable |           Default            | Storage  | Compression | Stats target | Description
-----------+------------------------+-----------+----------+------------------------------+----------+-------------+--------------+-------------
 artist_id | integer                |           | not null | generated always as identity | plain    |             |              |
 name      | character varying(120) |           |          |                              | extended |             |              |
Indexes:
    "artist_pkey" PRIMARY KEY, btree (artist_id)
Referenced by:
    TABLE "album" CONSTRAINT "album_artist_id_fkey" FOREIGN KEY (artist_id) REFERENCES artist(artist_id)
Access method: heap

chinook_auto_increment=# \d+ customer
                                                              Table "public.customer"
     Column     |         Type          | Collation | Nullable |           Default            | Storage  | Compression | Stats target | Description
----------------+-----------------------+-----------+----------+------------------------------+----------+-------------+--------------+-------------
 customer_id    | integer               |           | not null | generated always as identity | plain    |             |              |
 first_name     | character varying(40) |           | not null |                              | extended |             |              |
 last_name      | character varying(20) |           | not null |                              | extended |             |              |
 company        | character varying(80) |           |          |                              | extended |             |              |
 address        | character varying(70) |           |          |                              | extended |             |              |
 city           | character varying(40) |           |          |                              | extended |             |              |
 state          | character varying(40) |           |          |                              | extended |             |              |
 country        | character varying(40) |           |          |                              | extended |             |              |
 postal_code    | character varying(10) |           |          |                              | extended |             |              |
 phone          | character varying(24) |           |          |                              | extended |             |              |
 fax            | character varying(24) |           |          |                              | extended |             |              |
 email          | character varying(60) |           | not null |                              | extended |             |              |
 support_rep_id | integer               |           |          |                              | plain    |             |              |
Indexes:
    "customer_pkey" PRIMARY KEY, btree (customer_id)
    "customer_support_rep_id_idx" btree (support_rep_id)
    "idx_customer_email" btree (email)
Foreign-key constraints:
    "customer_support_rep_id_fkey" FOREIGN KEY (support_rep_id) REFERENCES employee(employee_id)
Referenced by:
    TABLE "invoice" CONSTRAINT "invoice_customer_id_fkey" FOREIGN KEY (customer_id) REFERENCES customer(customer_id)
Access method: heap

chinook_auto_increment=# \d+ employee
                                                                Table "public.employee"
   Column    |            Type             | Collation | Nullable |           Default            | Storage  | Compression | Stats target | Description
-------------+-----------------------------+-----------+----------+------------------------------+----------+-------------+--------------+-------------
 employee_id | integer                     |           | not null | generated always as identity | plain    |             |              |
 last_name   | character varying(20)       |           | not null |                              | extended |             |              |
 first_name  | character varying(20)       |           | not null |                              | extended |             |              |
 title       | character varying(30)       |           |          |                              | extended |             |              |
 reports_to  | integer                     |           |          |                              | plain    |             |              |
 birth_date  | timestamp without time zone |           |          |                              | plain    |             |              |
 hire_date   | timestamp without time zone |           |          |                              | plain    |             |              |
 address     | character varying(70)       |           |          |                              | extended |             |              |
 city        | character varying(40)       |           |          |                              | extended |             |              |
 state       | character varying(40)       |           |          |                              | extended |             |              |
 country     | character varying(40)       |           |          |                              | extended |             |              |
 postal_code | character varying(10)       |           |          |                              | extended |             |              |
 phone       | character varying(24)       |           |          |                              | extended |             |              |
 fax         | character varying(24)       |           |          |                              | extended |             |              |
 email       | character varying(60)       |           |          |                              | extended |             |              |
Indexes:
    "employee_pkey" PRIMARY KEY, btree (employee_id)
    "employee_reports_to_idx" btree (reports_to)
Foreign-key constraints:
    "employee_reports_to_fkey" FOREIGN KEY (reports_to) REFERENCES employee(employee_id)
Referenced by:
    TABLE "customer" CONSTRAINT "customer_support_rep_id_fkey" FOREIGN KEY (support_rep_id) REFERENCES employee(employee_id)
    TABLE "employee" CONSTRAINT "employee_reports_to_fkey" FOREIGN KEY (reports_to) REFERENCES employee(employee_id)
Access method: heap

chinook_auto_increment=# \d+ genre
                                                             Table "public.genre"
  Column  |          Type          | Collation | Nullable |           Default            | Storage  | Compression | Stats target | Description
----------+------------------------+-----------+----------+------------------------------+----------+-------------+--------------+-------------
 genre_id | integer                |           | not null | generated always as identity | plain    |             |              |
 name     | character varying(120) |           |          |                              | extended |             |              |
Indexes:
    "genre_pkey" PRIMARY KEY, btree (genre_id)
Referenced by:
    TABLE "track" CONSTRAINT "track_genre_id_fkey" FOREIGN KEY (genre_id) REFERENCES genre(genre_id)
Access method: heap

chinook_auto_increment=# \d+ invoice
                                                                    Table "public.invoice"
       Column        |            Type             | Collation | Nullable |           Default            | Storage  | Compression | Stats target | Description
---------------------+-----------------------------+-----------+----------+------------------------------+----------+-------------+--------------+-------------
 invoice_id          | integer                     |           | not null | generated always as identity | plain    |             |              |
 customer_id         | integer                     |           | not null |                              | plain    |             |              |
 invoice_date        | timestamp without time zone |           | not null |                              | plain    |             |              |
 billing_address     | character varying(70)       |           |          |                              | extended |             |              |
 billing_city        | character varying(40)       |           |          |                              | extended |             |              |
 billing_state       | character varying(40)       |           |          |                              | extended |             |              |
 billing_country     | character varying(40)       |           |          |                              | extended |             |              |
 billing_postal_code | character varying(10)       |           |          |                              | extended |             |              |
 total               | numeric(10,2)               |           | not null |                              | main     |             |              |
Indexes:
    "invoice_pkey" PRIMARY KEY, btree (invoice_id)
    "idx_invoice_total_discounted" btree ((total * 0.9))
    "invoice_customer_id_idx" btree (customer_id)
Foreign-key constraints:
    "invoice_customer_id_fkey" FOREIGN KEY (customer_id) REFERENCES customer(customer_id)
Referenced by:
    TABLE "invoice_line" CONSTRAINT "invoice_line_invoice_id_fkey" FOREIGN KEY (invoice_id) REFERENCES invoice(invoice_id)
Access method: heap

chinook_auto_increment=# \d+ invoice_line
                                                        Table "public.invoice_line"
     Column      |     Type      | Collation | Nullable |           Default            | Storage | Compression | Stats target | Description
-----------------+---------------+-----------+----------+------------------------------+---------+-------------+--------------+-------------
 invoice_line_id | integer       |           | not null | generated always as identity | plain   |             |              |
 invoice_id      | integer       |           | not null |                              | plain   |             |              |
 track_id        | integer       |           | not null |                              | plain   |             |              |
 unit_price      | numeric(10,2) |           | not null |                              | main    |             |              |
 quantity        | integer       |           | not null |                              | plain   |             |              |
Indexes:
    "invoice_line_pkey" PRIMARY KEY, btree (invoice_line_id)
    "idx_invoice_line_invoiceid_trackid" btree (invoice_id, track_id)
    "invoice_line_invoice_id_idx" btree (invoice_id)
    "invoice_line_track_id_idx" btree (track_id)
Foreign-key constraints:
    "invoice_line_invoice_id_fkey" FOREIGN KEY (invoice_id) REFERENCES invoice(invoice_id)
    "invoice_line_track_id_fkey" FOREIGN KEY (track_id) REFERENCES track(track_id)
Access method: heap

chinook_auto_increment=# \d+ media_type
                                                             Table "public.media_type"
    Column     |          Type          | Collation | Nullable |           Default            | Storage  | Compression | Stats target | Description
---------------+------------------------+-----------+----------+------------------------------+----------+-------------+--------------+-------------
 media_type_id | integer                |           | not null | generated always as identity | plain    |             |              |
 name          | character varying(120) |           |          |                              | extended |             |              |
Indexes:
    "media_type_pkey" PRIMARY KEY, btree (media_type_id)
Referenced by:
    TABLE "track" CONSTRAINT "track_media_type_id_fkey" FOREIGN KEY (media_type_id) REFERENCES media_type(media_type_id)
Access method: heap

chinook_auto_increment=# \d+ playlist
                                                             Table "public.playlist"
   Column    |          Type          | Collation | Nullable |           Default            | Storage  | Compression | Stats target | Description
-------------+------------------------+-----------+----------+------------------------------+----------+-------------+--------------+-------------
 playlist_id | integer                |           | not null | generated always as identity | plain    |             |              |
 name        | character varying(120) |           |          |                              | extended |             |              |
Indexes:
    "playlist_pkey" PRIMARY KEY, btree (playlist_id)
Referenced by:
    TABLE "playlist_track" CONSTRAINT "playlist_track_playlist_id_fkey" FOREIGN KEY (playlist_id) REFERENCES playlist(playlist_id)
Access method: heap

chinook_auto_increment=# \d+ playlist_track
                                        Table "public.playlist_track"
   Column    |  Type   | Collation | Nullable | Default | Storage | Compression | Stats target | Description
-------------+---------+-----------+----------+---------+---------+-------------+--------------+-------------
 playlist_id | integer |           | not null |         | plain   |             |              |
 track_id    | integer |           | not null |         | plain   |             |              |
Indexes:
    "playlist_track_pkey" PRIMARY KEY, btree (playlist_id, track_id)
    "playlist_track_playlist_id_idx" btree (playlist_id)
    "playlist_track_track_id_idx" btree (track_id)
Foreign-key constraints:
    "playlist_track_playlist_id_fkey" FOREIGN KEY (playlist_id) REFERENCES playlist(playlist_id)
    "playlist_track_track_id_fkey" FOREIGN KEY (track_id) REFERENCES track(track_id)
Access method: heap

chinook_auto_increment=# \d+ track
                                                                Table "public.track"
    Column     |          Type          | Collation | Nullable |           Default            | Storage  | Compression | Stats target | Description
---------------+------------------------+-----------+----------+------------------------------+----------+-------------+--------------+-------------
 track_id      | integer                |           | not null | generated always as identity | plain    |             |              |
 name          | character varying(200) |           | not null |                              | extended |             |              |
 album_id      | integer                |           |          |                              | plain    |             |              |
 media_type_id | integer                |           | not null |                              | plain    |             |              |
 genre_id      | integer                |           |          |                              | plain    |             |              |
 composer      | character varying(220) |           |          |                              | extended |             |              |
 milliseconds  | integer                |           | not null |                              | plain    |             |              |
 bytes         | integer                |           |          |                              | plain    |             |              |
 unit_price    | numeric(10,2)          |           | not null |                              | main     |             |              |
Indexes:
    "track_pkey" PRIMARY KEY, btree (track_id)
    "idx_track_name_fts" gin (to_tsvector('english'::regconfig, name::text))
    "track_album_id_idx" btree (album_id)
    "track_genre_id_idx" btree (genre_id)
    "track_media_type_id_idx" btree (media_type_id)
Foreign-key constraints:
    "track_album_id_fkey" FOREIGN KEY (album_id) REFERENCES album(album_id)
    "track_genre_id_fkey" FOREIGN KEY (genre_id) REFERENCES genre(genre_id)
    "track_media_type_id_fkey" FOREIGN KEY (media_type_id) REFERENCES media_type(media_type_id)
Referenced by:
    TABLE "invoice_line" CONSTRAINT "invoice_line_track_id_fkey" FOREIGN KEY (track_id) REFERENCES track(track_id)
    TABLE "playlist_track" CONSTRAINT "playlist_track_track_id_fkey" FOREIGN KEY (track_id) REFERENCES track(track_id)
Access method: heap

chinook_auto_increment=# SELECT
    tc.table_name AS foreign_table,
    kcu.column_name AS foreign_column,
    ccu.table_name AS primary_table,
    ccu.column_name AS primary_column
FROM
    information_schema.table_constraints AS tc
    JOIN information_schema.key_column_usage AS kcu
      ON tc.constraint_name = kcu.constraint_name
      AND tc.table_schema = kcu.table_schema
    JOIN information_schema.constraint_column_usage AS ccu
      ON ccu.constraint_name = tc.constraint_name
      AND ccu.table_schema = tc.table_schema
WHERE tc.constraint_type = 'FOREIGN KEY';
 foreign_table  | foreign_column | primary_table | primary_column
----------------+----------------+---------------+----------------
 album          | artist_id      | artist        | artist_id
 customer       | support_rep_id | employee      | employee_id
 employee       | reports_to     | employee      | employee_id
 invoice        | customer_id    | customer      | customer_id
 invoice_line   | invoice_id     | invoice       | invoice_id
 invoice_line   | track_id       | track         | track_id
 playlist_track | playlist_id    | playlist      | playlist_id
 playlist_track | track_id       | track         | track_id
 track          | album_id       | album         | album_id
 track          | genre_id       | genre         | genre_id
 track          | media_type_id  | media_type    | media_type_id
(11 rows)


```
## 8. Придумайте 3 своих метрики на основе показанных представлений
Для демонстрации использования различных типов соединений, предлагаю следующие 3 метрики, которые помогут оценить данные в базе.\
1. Общая сумма всех заказов каждого клиента:
```
chinook_auto_increment=# SELECT
    c.customer_id,
    c.first_name,
    c.last_name,
    SUM(i.total) AS total_spent
FROM
    customer c
INNER JOIN
    invoice i ON c.customer_id = i.customer_id
GROUP BY
    c.customer_id, c.first_name, c.last_name
ORDER BY
    total_spent DESC;


 customer_id | first_name |  last_name   | total_spent
-------------+------------+--------------+-------------
           6 | Helena     | Holý         |       49.62
          26 | Richard    | Cunningham   |       47.62
          57 | Luis       | Rojas        |       46.62
          45 | Ladislav   | Kovács       |       45.62
          46 | Hugh       | O'Reilly     |       45.62
          28 | Julia      | Barnett      |       43.62
          37 | Fynn       | Zimmermann   |       43.62
          24 | Frank      | Ralston      |       43.62
           7 | Astrid     | Gruber       |       42.62
           ...
          55 | Mark       | Taylor       |       37.62
          27 | Patrick    | Gray         |       37.62
           2 | Leonie     | Köhler       |       37.62
          16 | Frank      | Harris       |       37.62
          11 | Alexandre  | Rocha        |       37.62
          59 | Puja       | Srivastava   |       36.64
(59 rows)

```
2. Количество клиентов, которые сделали хотя бы один заказ:
```
chinook_auto_increment=# SELECT
    c.customer_id,
    c.first_name,
    c.last_name,
    SUM(i.total) AS total_spent
FROM
    customer c
INNER JOIN
    invoice i ON c.customer_id = i.customer_id
GROUP BY
    c.customer_id, c.first_name, c.last_name
ORDER BY
    total_spent DESC;
chinook_auto_increment=# SELECT
    COUNT(DISTINCT c.customer_id) AS active_customers
FROM
    customer c
INNER JOIN
    invoice i ON c.customer_id = i.customer_id;
 active_customers
------------------
               59
(1 row)

```
3. Средняя стоимость заказа по каждому жанру музыки:
```
chinook_auto_increment=# SELECT
    g.name AS genre_name,
    AVG(il.unit_price * il.quantity) AS avg_order_value
FROM
    genre g
INNER JOIN
    track t ON g.genre_id = t.genre_id
INNER JOIN
    invoice_line il ON t.track_id = il.track_id
GROUP BY
    g.name
ORDER BY
    avg_order_value DESC;
     genre_name     |    avg_order_value
--------------------+------------------------
 Sci Fi & Fantasy   |     1.9900000000000000
 TV Shows           |     1.9900000000000000
 Science Fiction    |     1.9900000000000000
 Comedy             |     1.9900000000000000
 Drama              |     1.9900000000000000
 Pop                | 0.99000000000000000000
 Easy Listening     | 0.99000000000000000000
 Alternative        | 0.99000000000000000000
 Rock               | 0.99000000000000000000
 World              | 0.99000000000000000000
 Reggae             | 0.99000000000000000000
 Blues              | 0.99000000000000000000
 Rock And Roll      | 0.99000000000000000000
 Alternative & Punk | 0.99000000000000000000
 Metal              | 0.99000000000000000000
 Soundtrack         | 0.99000000000000000000
 Bossa Nova         | 0.99000000000000000000
 Hip Hop/Rap        | 0.99000000000000000000
 Heavy Metal        | 0.99000000000000000000
 Jazz               | 0.99000000000000000000
 Latin              | 0.99000000000000000000
 Electronica/Dance  | 0.99000000000000000000
 R&B/Soul           | 0.99000000000000000000
 Classical          | 0.99000000000000000000
(24 rows)

```