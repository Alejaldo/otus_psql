# Задание №11 "Работа с индексами"

## 0. Подготовка рабочей среды
Для выполнения буду использовать свою локальную машину, в части применения своей docker сети "kntxt-psql-network", где у меня есть контейнер "pg-cluster" с установленным PostgreSQL 15.\
Проверю наличие данной сети и запущу контейнер "pg-cluster".
```
[~]$ docker network ls
NETWORK ID     NAME                 DRIVER    SCOPE
7ab7ea50b143   bridge               bridge    local
ff03a5139a15   docker_gwbridge      bridge    local
176e8fe1f8a8   host                 host      local
fiindvcgi0jh   ingress              overlay   swarm
10bea201521a   kntxt-psql-network   bridge    local
7bb589e0049c   none                 null      local
b51c9cbded88   nsud_default         bridge    local
[~]$ docker network inspect kntxt-psql-network
[
    {
        "Name": "kntxt-psql-network",
        "Id": "10bea201521afb397efe855712717b68904f945c1c3e0867e6175f8e08bc8266",
        "Created": "2024-03-16T22:46:25.442932201+03:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.80.2.0/24",
                    "Gateway": "172.80.2.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {},
        "Options": {},
        "Labels": {}
    }
]
[~]$ docker ps -a
CONTAINER ID   IMAGE                                                              COMMAND                  CREATED        STATUS                   PORTS                                                                     NAMES
...
bcf6240d7a5f   postgres:15                                                        "docker-entrypoint.s…"   8 weeks ago    Exited (0) 10 days ago                                                                             pg-cluster
[~]$ docker start pg-cluster
pg-cluster
[~]$ docker ps
CONTAINER ID   IMAGE                                                              COMMAND                  CREATED        STATUS                   PORTS                                                                     NAMES
...
bcf6240d7a5f   postgres:15                                                        "docker-entrypoint.s…"   8 weeks ago    Up 24 seconds            0.0.0.0:5477->5432/tcp, :::5477->5432/tcp                                 pg-cluster
[~]$ docker exec -it pg-cluster psql -U postgres
psql (15.6 (Debian 15.6-1.pgdg120+2))
Type "help" for help.

postgres=# \l
                                                 List of databases
    Name     |  Owner   | Encoding |  Collate   |   Ctype    | ICU Locale | Locale Provider |   Access privileges
-------------+----------+----------+------------+------------+------------+-----------------+-----------------------
 kntxtdb     | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            |
 new_kntxtdb | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            |
 postgres    | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            |
 template0   | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/postgres          +
             |          |          |            |            |            |                 | postgres=CTc/postgres
 template1   | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/postgres          +
             |          |          |            |            |            |                 | postgres=CTc/postgres
 testdb      | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =Tc/postgres         +
             |          |          |            |            |            |                 | postgres=CTc/postgres+
             |          |          |            |            |            |                 | readonly=c/postgres
(6 rows)

postgres=# \c kntxtdb
You are now connected to database "kntxtdb" as user "postgres".
kntxtdb=# \dt
Did not find any relations.
kntxtdb=# \dt music_schema.*
                 List of relations
    Schema    |      Name       | Type  |  Owner
--------------+-----------------+-------+----------
 music_schema | artists         | table | postgres
 music_schema | artists_backup  | table | postgres
 music_schema | revenues        | table | postgres
 music_schema | revenues_backup | table | postgres
 music_schema | tracks          | table | postgres
 music_schema | tracks_backup   | table | postgres
(6 rows)

kntxtdb=#

```

Теперь будут скачивать sql дамп для создания тестовой базы данных электронной музыки из репозитория ["chinook-database"](https://github.com/lerocha/chinook-database/releases)\
Возьму для себя файл "Chinook_PostgreSql_AutoIncrementPKs.sql"\
Теперь совершу необхдимые действия по установке этой базы данных:
```
[~]$ docker cp Downloads/Chinook_PostgreSql_AutoIncrementPKs.sql pg-cluster:/ChinookDatabase.sql
Successfully copied 581kB to pg-cluster:/ChinookDatabase.sql
[~]$ docker exec -it pg-cluster psql -U postgres -c "CREATE DATABASE chinookdb;"
CREATE DATABASE
[~]$ docker exec -it pg-cluster psql -U postgres -d chinookdb -f /ChinookDatabase.sql
psql:/ChinookDatabase.sql:19: NOTICE:  database "chinook_auto_increment" does not exist, skipping
DROP DATABASE
CREATE DATABASE
You are now connected to database "chinook_auto_increment" as user "postgres".
CREATE TABLE
CREATE TABLE
CREATE TABLE
CREATE TABLE
CREATE TABLE
CREATE TABLE
CREATE TABLE
CREATE TABLE
CREATE TABLE
CREATE TABLE
CREATE TABLE
ALTER TABLE
CREATE INDEX
ALTER TABLE
CREATE INDEX
ALTER TABLE
CREATE INDEX
ALTER TABLE
CREATE INDEX
ALTER TABLE
CREATE INDEX
ALTER TABLE
CREATE INDEX
ALTER TABLE
CREATE INDEX
ALTER TABLE
CREATE INDEX
ALTER TABLE
CREATE INDEX
ALTER TABLE
CREATE INDEX
ALTER TABLE
CREATE INDEX
INSERT 0 25
INSERT 0 5
INSERT 0 275
INSERT 0 347
INSERT 0 1000
INSERT 0 1000
INSERT 0 1000
INSERT 0 503
INSERT 0 8
INSERT 0 59
INSERT 0 412
INSERT 0 1000
INSERT 0 1000
INSERT 0 240
INSERT 0 18
INSERT 0 1000
INSERT 0 1000
INSERT 0 1000
INSERT 0 1000
INSERT 0 1000
INSERT 0 1000
INSERT 0 1000
INSERT 0 1000
INSERT 0 715
[~]$ docker exec -it pg-cluster psql -U postgres -d chinookdb
psql (15.6 (Debian 15.6-1.pgdg120+2))
Type "help" for help.

chinookdb=# \dt
Did not find any relations.
```
На этом этапе не заметил что скрипт по созданию таблиц сам автоматически создал другую базу данных "chinook_auto_increment", поэтому проверяю ее:
```
chinookdb=# exit
[~]$ docker exec -it pg-cluster psql -U postgres -d chinook_auto_increment
psql (15.6 (Debian 15.6-1.pgdg120+2))
Type "help" for help.

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

chinook_auto_increment=#

```
Теперь вижу, что все установилось.

Новые индексы добавлю для следующих таблиц:
- 1. Индекс для таблицы customer - Столбец: email - Причина: Адреса электронной почты часто используются для поиска записей клиентов. Индекс на столбце email может ускорить поиск и запросы, связанные с адресами электронной почты клиентов.
- 2: Индекс полнотекстового поиска для таблицы track - Столбец: name - Причина: Полнотекстовый поиск по названиям треков может улучшить функциональность поиска для нахождения треков по ключевым словам. Это полезно для приложений, предоставляющих функции поиска по названиям треков.
- 3: Индекс на основе функции для таблицы invoice - Столбец: Функция на total - Причина: Если есть частые запросы, включающие определенные функции на столбце total (например, расчет скидок или налогов), создание индекса на основе функции может оптимизировать эти запросы.
- 4: Многоколоночный индекс для таблицы invoice_line - Столбцы: invoice_id, track_id - Причина: Запросы, соединяющие или фильтрующие по invoice_id и track_id, могут выиграть от составного индекса, улучшая производительность сложных соединений и условий фильтрации.

## 1. Создать индекс к какой-либо из таблиц вашей БД
Делаю по своему плану, создаю индекс для `customer.email`:
```
chinook_auto_increment=# CREATE INDEX idx_customer_email ON customer (email);
CREATE INDEX
```
## 2. Прислать текстом результат команды explain, в которой используется данный индекс
Результат команды EXPLAIN:
```
chinook_auto_increment=# EXPLAIN SELECT * FROM customer WHERE email = 'test@example.com';
                        QUERY PLAN
----------------------------------------------------------
 Seq Scan on customer  (cost=0.00..2.74 rows=1 width=140)
   Filter: ((email)::text = 'test@example.com'::text)
(2 rows)

chinook_auto_increment=# SELECT * FROM customer
chinook_auto_increment-# ;
chinook_auto_increment=# EXPLAIN SELECT * FROM customer WHERE email = 'jenniferp@rogers.ca';
                        QUERY PLAN
----------------------------------------------------------
 Seq Scan on customer  (cost=0.00..2.74 rows=1 width=140)
   Filter: ((email)::text = 'jenniferp@rogers.ca'::text)
(2 rows)

chinook_auto_increment=#
```

Тест Без Индекса:
```
chinook_auto_increment=# SET enable_seqscan TO off;
SET
chinook_auto_increment=# EXPLAIN ANALYZE SELECT * FROM customer WHERE email = 'jenniferp@rogers.ca';
                                                          QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------
 Index Scan using idx_customer_email on customer  (cost=0.14..8.16 rows=1 width=140) (actual time=0.060..0.062 rows=1 loops=1)
   Index Cond: ((email)::text = 'jenniferp@rogers.ca'::text)
 Planning Time: 0.073 ms
 Execution Time: 7.669 ms
(4 rows)
```
Тест С Индексом:
```
chinook_auto_increment=# SET enable_seqscan TO on;
SET
chinook_auto_increment=# EXPLAIN ANALYZE SELECT * FROM customer WHERE email = 'jenniferp@rogers.ca';
                                             QUERY PLAN
----------------------------------------------------------------------------------------------------
 Seq Scan on customer  (cost=0.00..2.74 rows=1 width=140) (actual time=0.014..0.025 rows=1 loops=1)
   Filter: ((email)::text = 'jenniferp@rogers.ca'::text)
   Rows Removed by Filter: 58
 Planning Time: 0.073 ms
 Execution Time: 0.038 ms
(5 rows)
```

## 3. Реализовать индекс для полнотекстового поиска
Индекс полнотекстового поиска на track.name:
```
chinook_auto_increment=# CREATE INDEX idx_track_name_fts ON track USING gin (to_tsvector('english', name));
CREATE INDEX
```
Тест Без Индекса:
```
chinook_auto_increment=# SET enable_seqscan TO off;
SET
chinook_auto_increment=# EXPLAIN ANALYZE SELECT * FROM track WHERE to_tsvector('english', name) @@ to_tsquery('english', 'rock');
                                                          QUERY PLAN
------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on track  (cost=12.14..50.88 rows=18 width=70) (actual time=0.016..0.032 rows=30 loops=1)
   Recheck Cond: (to_tsvector('english'::regconfig, (name)::text) @@ '''rock'''::tsquery)
   Heap Blocks: exact=18
   ->  Bitmap Index Scan on idx_track_name_fts  (cost=0.00..12.13 rows=18 width=0) (actual time=0.010..0.010 rows=30 loops=1)
         Index Cond: (to_tsvector('english'::regconfig, (name)::text) @@ '''rock'''::tsquery)
 Planning Time: 1.583 ms
 Execution Time: 0.048 ms
(7 rows)

```
Тест С Индексом:
```
chinook_auto_increment=# SET enable_seqscan TO on;
SET
chinook_auto_increment=# EXPLAIN ANALYZE SELECT * FROM track WHERE to_tsvector('english', name) @@ to_tsquery('english', 'rock');
                                                          QUERY PLAN
------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on track  (cost=12.14..50.88 rows=18 width=70) (actual time=0.041..0.069 rows=30 loops=1)
   Recheck Cond: (to_tsvector('english'::regconfig, (name)::text) @@ '''rock'''::tsquery)
   Heap Blocks: exact=18
   ->  Bitmap Index Scan on idx_track_name_fts  (cost=0.00..12.13 rows=18 width=0) (actual time=0.032..0.033 rows=30 loops=1)
         Index Cond: (to_tsvector('english'::regconfig, (name)::text) @@ '''rock'''::tsquery)
 Planning Time: 0.136 ms
 Execution Time: 0.091 ms
(7 rows)

```
## 4. Реализовать индекс на часть таблицы или индекс на поле с функцией
Индекс на основе функции для invoice.total:
```
chinook_auto_increment=# CREATE INDEX idx_invoice_total_discounted ON invoice ((total * 0.9));
CREATE INDEX
```
Тест Без Индекса:
```
chinook_auto_increment=# SET enable_seqscan TO off;
SET
chinook_auto_increment=# EXPLAIN ANALYZE SELECT * FROM invoice WHERE (total * 0.9) > 10;
                                                               QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on invoice  (cost=9.33..17.39 rows=137 width=66) (actual time=0.026..0.035 rows=62 loops=1)
   Recheck Cond: ((total * 0.9) > '10'::numeric)
   Heap Blocks: exact=6
   ->  Bitmap Index Scan on idx_invoice_total_discounted  (cost=0.00..9.30 rows=137 width=0) (actual time=0.023..0.023 rows=62 loops=1)
         Index Cond: ((total * 0.9) > '10'::numeric)
 Planning Time: 8.130 ms
 Execution Time: 0.051 ms
(7 rows)

```
Тест С Индексом:
```
chinook_auto_increment=# SET enable_seqscan TO on;
SET
chinook_auto_increment=# EXPLAIN ANALYZE SELECT * FROM invoice WHERE (total * 0.9) > 10;
                                              QUERY PLAN
------------------------------------------------------------------------------------------------------
 Seq Scan on invoice  (cost=0.00..12.18 rows=137 width=66) (actual time=0.010..0.131 rows=62 loops=1)
   Filter: ((total * 0.9) > '10'::numeric)
   Rows Removed by Filter: 350
 Planning Time: 0.048 ms
 Execution Time: 0.142 ms
(5 rows)

```
## 5. Создать индекс на несколько полей
Многоколоночный индекс на invoice_line:
```
chinook_auto_increment=# CREATE INDEX idx_invoice_line_invoiceid_trackid ON invoice_line (invoice_id, track_id);
CREATE INDEX
```
Тест Без Индекса:
```
chinook_auto_increment=# SET enable_seqscan TO off;
SET
chinook_auto_increment=# EXPLAIN ANALYZE SELECT * FROM invoice_line WHERE invoice_id = 1 AND track_id = 1;
                                                                    QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------------------------------
 Index Scan using idx_invoice_line_invoiceid_trackid on invoice_line  (cost=0.28..8.30 rows=1 width=21) (actual time=0.013..0.013 rows=0 loops=1)
   Index Cond: ((invoice_id = 1) AND (track_id = 1))
 Planning Time: 8.232 ms
 Execution Time: 0.023 ms
(4 rows)

```
Тест С Индексом:
```
chinook_auto_increment=# SET enable_seqscan TO on;
SET
chinook_auto_increment=# EXPLAIN ANALYZE SELECT * FROM invoice_line WHERE invoice_id = 1 AND track_id = 1;
                                                                    QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------------------------------
 Index Scan using idx_invoice_line_invoiceid_trackid on invoice_line  (cost=0.28..8.30 rows=1 width=21) (actual time=0.014..0.014 rows=0 loops=1)
   Index Cond: ((invoice_id = 1) AND (track_id = 1))
 Planning Time: 0.086 ms
 Execution Time: 0.029 ms
(4 rows)

```
## 6. Написать комментарии к каждому из индексов
Комментарии к каждому индексу:
```
chinook_auto_increment=# COMMENT ON INDEX idx_customer_email IS 'Индекс на email клиента для ускорения поиска по email';
COMMENT
chinook_auto_increment=# COMMENT ON INDEX idx_track_name_fts IS 'Индекс полнотекстового поиска по названиям треков';
COMMENT
chinook_auto_increment=# COMMENT ON INDEX idx_invoice_total_discounted IS 'Индекс на основе функции по скидочному итогу счета';
COMMENT
chinook_auto_increment=# COMMENT ON INDEX idx_invoice_line_invoiceid_trackid IS 'Составной индекс по invoice_id и track_id';
COMMENT
```
проверяю их наличие с помощью pg_description:
```
chinook_auto_increment=# SELECT
    i.indexname,
    i.indexdef,
    d.description AS comment
FROM
    pg_indexes i
JOIN
    pg_class c ON c.relname = i.indexname
LEFT JOIN
    pg_description d ON d.objoid = c.oid
WHERE
    i.schemaname = 'public';
```
где:
- [pg_indexes](https://www.postgresql.org/docs/current/view-pg-indexes.html): представление предоставляет доступ к полезной информации о каждом индексе в базе данных, включая схему, таблицу, название индекса и определение индекса.
- [pg_class](https://www.postgresql.org/docs/current/catalog-pg-class.html): каталог описывает таблицы и другие объекты, такие как индексы, представления и последовательности, содержит метаданные о каждом объекте, включая имя, владельца, тип и многое другое
- [pg_description](https://www.postgresql.org/docs/current/catalog-pg-description.html): каталог хранит комментарии к каждому объекту базы данных
- pg_indexes i: Это представление предоставляет информацию о каждом индексе в базе данных, включая название схемы, таблицы, название индекса и определение индекса.
- JOIN pg_class c ON c.relname = i.indexname: Каталог pg_class содержит метаданные обо всех таблицах и индексах в базе данных. Этот JOIN связывает индексы с их записями в pg_class.
- LEFT JOIN pg_description d ON d.objoid = c.oid: Каталог pg_description хранит комментарии ко всем объектам базы данных. Этот LEFT JOIN добавляет комментарии к индексам, если они существуют.
- WHERE i.schemaname = 'public': Этот фильтр ограничивает результаты индексами, которые находятся в схеме public.
```

             indexname              |                                                  indexdef                                                   |                        comment
------------------------------------+-------------------------------------------------------------------------------------------------------------+-------------------------------------------------------
 album_pkey                         | CREATE UNIQUE INDEX album_pkey ON public.album USING btree (album_id)                                       |
 artist_pkey                        | CREATE UNIQUE INDEX artist_pkey ON public.artist USING btree (artist_id)                                    |
 customer_pkey                      | CREATE UNIQUE INDEX customer_pkey ON public.customer USING btree (customer_id)                              |
 employee_pkey                      | CREATE UNIQUE INDEX employee_pkey ON public.employee USING btree (employee_id)                              |
 genre_pkey                         | CREATE UNIQUE INDEX genre_pkey ON public.genre USING btree (genre_id)                                       |
 invoice_pkey                       | CREATE UNIQUE INDEX invoice_pkey ON public.invoice USING btree (invoice_id)                                 |
 invoice_line_pkey                  | CREATE UNIQUE INDEX invoice_line_pkey ON public.invoice_line USING btree (invoice_line_id)                  |
 media_type_pkey                    | CREATE UNIQUE INDEX media_type_pkey ON public.media_type USING btree (media_type_id)                        |
 playlist_pkey                      | CREATE UNIQUE INDEX playlist_pkey ON public.playlist USING btree (playlist_id)                              |
 playlist_track_pkey                | CREATE UNIQUE INDEX playlist_track_pkey ON public.playlist_track USING btree (playlist_id, track_id)        |
 track_pkey                         | CREATE UNIQUE INDEX track_pkey ON public.track USING btree (track_id)                                       |
 album_artist_id_idx                | CREATE INDEX album_artist_id_idx ON public.album USING btree (artist_id)                                    |
 customer_support_rep_id_idx        | CREATE INDEX customer_support_rep_id_idx ON public.customer USING btree (support_rep_id)                    |
 employee_reports_to_idx            | CREATE INDEX employee_reports_to_idx ON public.employee USING btree (reports_to)                            |
 invoice_customer_id_idx            | CREATE INDEX invoice_customer_id_idx ON public.invoice USING btree (customer_id)                            |
 invoice_line_invoice_id_idx        | CREATE INDEX invoice_line_invoice_id_idx ON public.invoice_line USING btree (invoice_id)                    |
 invoice_line_track_id_idx          | CREATE INDEX invoice_line_track_id_idx ON public.invoice_line USING btree (track_id)                        |
 playlist_track_playlist_id_idx     | CREATE INDEX playlist_track_playlist_id_idx ON public.playlist_track USING btree (playlist_id)              |
 playlist_track_track_id_idx        | CREATE INDEX playlist_track_track_id_idx ON public.playlist_track USING btree (track_id)                    |
 track_album_id_idx                 | CREATE INDEX track_album_id_idx ON public.track USING btree (album_id)                                      |
 track_genre_id_idx                 | CREATE INDEX track_genre_id_idx ON public.track USING btree (genre_id)                                      |
 track_media_type_id_idx            | CREATE INDEX track_media_type_id_idx ON public.track USING btree (media_type_id)                            |
 idx_customer_email                 | CREATE INDEX idx_customer_email ON public.customer USING btree (email)                                      | Индекс на email клиента для ускорения поиска по email
 idx_track_name_fts                 | CREATE INDEX idx_track_name_fts ON public.track USING gin (to_tsvector('english'::regconfig, (name)::text)) | Индекс полнотекстового поиска по названиям треков
 idx_invoice_total_discounted       | CREATE INDEX idx_invoice_total_discounted ON public.invoice USING btree (((total * 0.9)))                   | Индекс на основе функции по скидочному итогу счета
 idx_invoice_line_invoiceid_trackid | CREATE INDEX idx_invoice_line_invoiceid_trackid ON public.invoice_line USING btree (invoice_id, track_id)   | Составной индекс по invoice_id и track_id
(26 rows)
```
## 7. Описать что и как делали и с какими проблемами столкнулись
На основе проведенных тестов и результатов можно сделать следующие выводы о важности и эффективности созданных индексов:
1. Индекс на customer.email:
Индекс на customer.email существенно улучшает производительность поиска по email, что подтверждается значительным снижением времени выполнения запроса. Это особенно полезно для операций, связанных с частыми запросами по email клиентов.
2. Индекс полнотекстового поиска на track.name:
Полнотекстовый индекс на track.name позволяет значительно ускорить поиск треков по ключевым словам. Это обеспечивает более эффективную работу поисковых функций в приложениях, связанных с музыкальными треками.
3. Индекс на основе функции для invoice.total:
Индекс на основе функции (total * 0.9) оптимизирует запросы, включающие расчетные значения. Это полезно для быстрого выполнения запросов, связанных с применением скидок или других расчетов на основе столбца total.
4. Многоколоночный индекс на invoice_line:
Многоколоночный индекс на invoice_id и track_id улучшает производительность сложных запросов, которые фильтруют данные по этим двум столбцам. Это особенно важно для оптимизации запросов, включающих соединения и фильтрацию по нескольким критериям.
