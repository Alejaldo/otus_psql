# Задание №9 "Резервное копирование и восстановление"

## 1. Создаем ВМ/докер c ПГ.
Для разнообразия буду использовать ранее созданную локально docker сеть "kntxt-psql-network" в которой используется контейнер "pg-cluster" использующий PostgreSQL 15 версии.\
Проверяю, что сеть в норме:
```
NETWORK ID     NAME                 DRIVER    SCOPE
ed49be95f3d9   bridge               bridge    local
ff03a5139a15   docker_gwbridge      bridge    local
176e8fe1f8a8   host                 host      local
fiindvcgi0jh   ingress              overlay   swarm
10bea201521a   kntxt-psql-network   bridge    local
7bb589e0049c   none                 null      local
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
CONTAINER ID   IMAGE                                                              COMMAND                  CREATED       STATUS                      PORTS                                                                 NAMES
...
bcf6240d7a5f   postgres:15                                                        "docker-entrypoint.s…"   6 weeks ago   Exited (0) 6 weeks ago                                                                            pg-cluster
```
Запускаю снова контейнер "pg-cluster":
```
[~]$ docker start pg-cluster
pg-cluster
[~]$ docker ps
CONTAINER ID   IMAGE                                                              COMMAND                  CREATED       STATUS          PORTS                                                                 NAMES
...
bcf6240d7a5f   postgres:15                                                        "docker-entrypoint.s…"   6 weeks ago   Up 3 seconds    0.0.0.0:5477->5432/tcp, :::5477->5432/tcp                             pg-cluster
[~]$ docker exec -it pg-cluster psql -U postgres
psql (15.6 (Debian 15.6-1.pgdg120+2))
Type "help" for help.

postgres=# \l
                                                List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    | ICU Locale | Locale Provider |   Access privileges
-----------+----------+----------+------------+------------+------------+-----------------+-----------------------
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            |
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/postgres          +
           |          |          |            |            |            |                 | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/postgres          +
           |          |          |            |            |            |                 | postgres=CTc/postgres
 testdb    | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =Tc/postgres         +
           |          |          |            |            |            |                 | postgres=CTc/postgres+
           |          |          |            |            |            |                 | readonly=c/postgres
(4 rows)
```
Вижу что все в рабочем состоянии могу приступать к дальнейшим шагам.

## 2. Создаем БД, схему и в ней таблицу.
Создаю базу данных, схему и таблицы:
```
postgres=# CREATE DATABASE kntxtdb;
CREATE DATABASE
postgres=# \c kntxtdb
You are now connected to database "kntxtdb" as user "postgres".
kntxtdb=# CREATE SCHEMA music_schema;
CREATE SCHEMA
kntxtdb=# CREATE TABLE music_schema.artists (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);
CREATE TABLE
kntxtdb=# CREATE TABLE music_schema.tracks (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    BPM INTEGER CHECK (BPM BETWEEN 128 AND 138),
    release_date DATE NOT NULL,
    artist_id INTEGER NOT NULL,
    FOREIGN KEY (artist_id) REFERENCES music_schema.artists(id)
);
CREATE TABLE
kntxtdb=# CREATE TABLE music_schema.revenues (
    id SERIAL PRIMARY KEY,
    track_id INTEGER NOT NULL,
    artist_id INTEGER NOT NULL,
    source JSONB,
    FOREIGN KEY (track_id) REFERENCES music_schema.tracks(id),
    FOREIGN KEY (artist_id) REFERENCES music_schema.artists(id)
);
CREATE TABLE
```
## 3. Заполним таблицы автосгенерированными 100 записями.
Теперь заполню таблицы данными, где 100 записями заполню таблицу "music_schema.tracks":
```
kntxtdb=# INSERT INTO music_schema.artists (name) VALUES
('Alignment'),
('Amazingblaze'),
('Charlotte de Witte'),
('Chris Liebing'),
('Indira Paganotto'),
('Monoloc'),
('ONYVAA');
INSERT 0 7
kntxtdb=# INSERT INTO music_schema.tracks (name, BPM, release_date, artist_id)
SELECT
    'Techno Track ' || s,
    (random() * (138-128) + 128)::int,
    '2022-01-01'::date + (random() * (365*2))::int,
    (SELECT id FROM music_schema.artists ORDER BY random() LIMIT 1)
FROM generate_series(1, 100) s;
INSERT 0 100
kntxtdb=# INSERT INTO music_schema.revenues (track_id, artist_id, source)
SELECT
    t.id,
    t.artist_id,
    jsonb_build_object('source', (ARRAY['Beatport', 'Spotify', 'Others'])[floor(random() * 3 + 1)], 'amount', round((random() * 1000)::numeric, 2))
FROM music_schema.tracks t
ORDER BY random()
LIMIT 100;
INSERT 0 100

```
## 4. Под линукс пользователем Postgres создадим каталог для бэкапов
Теперь выхожу из базы данных в терминал и зайду в командную строку контейнера, где и создам папку для бэкапов:
```
kntxtdb=# \q
[~]$ docker exec -it pg-cluster bash
root@bcf6240d7a5f:/# su - postgres
postgres@bcf6240d7a5f:~$ mkdir -p /var/lib/postgresql/backups
```
## 5. Сделаем логический бэкап используя утилиту COPY
Буду делать копии в формат .csv
```
postgres@bcf6240d7a5f:~$ psql -d kntxtdb -c "\copy music_schema.artists TO '/var/lib/postgresql/backups/artists_backup.csv' CSV HEADER"
COPY 7
postgres@bcf6240d7a5f:~$ psql -d kntxtdb -c "\copy music_schema.tracks TO '/var/lib/postgresql/backups/tracks_backup.csv' CSV HEADER"
COPY 100
postgres@bcf6240d7a5f:~$ psql -d kntxtdb -c "\copy music_schema.revenues TO '/var/lib/postgresql/backups/revenues_backup.csv' CSV HEADER"
COPY 100
```
## 6. Восстановим в 2 таблицу данные из бэкапа.
Создам 3 таблицы дублирующие мои 3 текущие, но пустые:
```
postgres@bcf6240d7a5f:~$ exit
logout
root@bcf6240d7a5f:/# exit
exit
[~]$ docker exec -it pg-cluster psql -U postgres
psql (15.6 (Debian 15.6-1.pgdg120+2))
Type "help" for help.

postgres=# \c kntxtdb
You are now connected to database "kntxtdb" as user "postgres".
kntxtdb=# CREATE TABLE music_schema.artists_backup (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);
CREATE TABLE
kntxtdb=# CREATE TABLE music_schema.tracks_backup (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    BPM INTEGER CHECK (BPM BETWEEN 128 AND 138),
    release_date DATE NOT NULL,
    artist_id INTEGER NOT NULL,
    FOREIGN KEY (artist_id) REFERENCES music_schema.artists_backup(id)
);
CREATE TABLE
kntxtdb=# CREATE TABLE music_schema.revenues_backup (
    id SERIAL PRIMARY KEY,
    track_id INTEGER NOT NULL,
    artist_id INTEGER NOT NULL,
    source JSONB,
    FOREIGN KEY (track_id) REFERENCES music_schema.tracks_backup(id),
    FOREIGN KEY (artist_id) REFERENCES music_schema.artists_backup(id)
);
CREATE TABLE
kntxtdb=#
```
Возвращась снова в командную строку контейнера для совершения операций по восстановлению данных в дублирующие таблицы:
```
kntxtdb=# \q
[~]$ docker exec -it pg-cluster bash
root@bcf6240d7a5f:/# psql -d kntxtdb -c "\copy music_schema.artists_backup FROM '/var/lib/postgresql/backups/artists_backup.csv' CSV HEADER"
psql: error: connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: FATAL:  role "root" does not exist
root@bcf6240d7a5f:/# su - postgres
postgres@bcf6240d7a5f:~$ psql -d kntxtdb -c "\copy music_schema.artists_backup FROM '/var/lib/postgresql/backups/artists_backup.csv' CSV HEADER"
COPY 7
postgres@bcf6240d7a5f:~$ psql -d kntxtdb -c "\copy music_schema.tracks_backup FROM '/var/lib/postgresql/backups/tracks_backup.csv' CSV HEADER"
COPY 100
postgres@bcf6240d7a5f:~$ psql -d kntxtdb -c "\copy music_schema.revenues_backup FROM '/var/lib/postgresql/backups/revenues_backup.csv' CSV HEADER"
COPY 100
```
Перехожу в базу для проверки данных:
```
postgres@bcf6240d7a5f:~$ exit
logout
root@bcf6240d7a5f:/# exit
exit
[~]$ docker exec -it pg-cluster psql -U postgres
psql (15.6 (Debian 15.6-1.pgdg120+2))
Type "help" for help.

postgres=# \c kntxtdb
You are now connected to database "kntxtdb" as user "postgres".
kntxtdb=# SELECT * FROM music_schema.artists_backup;
 id |        name
----+--------------------
  1 | Alignment
  2 | Amazingblaze
  3 | Charlotte de Witte
  4 | Chris Liebing
  5 | Indira Paganotto
  6 | Monoloc
  7 | ONYVAA
(7 rows)

kntxtdb=# SELECT * FROM music_schema.tracks_backup ORDER BY id DESC LIMIT 10;
 id  |       name       | bpm | release_date | artist_id
-----+------------------+-----+--------------+-----------
 100 | Techno Track 100 | 134 | 2023-08-03   |         3
  99 | Techno Track 99  | 131 | 2023-10-30   |         3
  98 | Techno Track 98  | 136 | 2023-07-23   |         3
  97 | Techno Track 97  | 129 | 2023-09-25   |         3
  96 | Techno Track 96  | 135 | 2023-06-11   |         3
  95 | Techno Track 95  | 138 | 2023-12-19   |         3
  94 | Techno Track 94  | 132 | 2023-10-23   |         3
  93 | Techno Track 93  | 136 | 2023-08-19   |         3
  92 | Techno Track 92  | 129 | 2023-10-06   |         3
  91 | Techno Track 91  | 136 | 2022-08-06   |         3
(10 rows)

kntxtdb=# SELECT * FROM music_schema.revenues_backup ORDER BY id DESC LIMIT 10;
 id  | track_id | artist_id |                 source
-----+----------+-----------+----------------------------------------
 100 |       39 |         3 | {"amount": 999.60, "source": "Others"}
  99 |       30 |         3 | {"amount": 999.07, "source": "Others"}
  98 |      100 |         3 | {"amount": 976.19, "source": "Others"}
  97 |       70 |         3 | {"amount": 971.10, "source": "Others"}
  96 |       96 |         3 | {"amount": 969.66, "source": "Others"}
  95 |       53 |         3 | {"amount": 963.85, "source": "Others"}
  94 |       78 |         3 | {"amount": 961.25, "source": "Others"}
  93 |       72 |         3 | {"amount": 948.38, "source": "Others"}
  92 |       87 |         3 | {"amount": 937.42, "source": "Others"}
  91 |       97 |         3 | {"amount": 933.85, "source": "Others"}
(10 rows)
```
Вижу что все успешно скопировалось из бэкапов в дублирующие таблицы.
## 7. Используя утилиту pg_dump создадим бэкап в кастомном сжатом формате двух таблиц
Перехожу обратно в контейнер под postgres юзером где и сделаю бэкап всех 6 моих таблиц:
```
kntxtdb=# \q
[~]$ docker exec -it pg-cluster bash
root@bcf6240d7a5f:/# su - postgres
postgres@bcf6240d7a5f:~$ pg_dump -d kntxtdb \
    -t music_schema.artists \
    -t music_schema.artists_backup \
    -t music_schema.tracks \
    -t music_schema.tracks_backup \
    -t music_schema.revenues \
    -t music_schema.revenues_backup \
    -F c -f /var/lib/postgresql/backups/custom_backup.dump
```
## 8. Используя утилиту pg_restore восстановим в новую БД только вторую таблицу!
Не выходя из командной строки контейнера там же создам новую базу данных и буду делать операции по восстановлению в нее 3 таблиц music_schema.artists_backup, music_schema.tracks_backup, music_schema.revenues_backup
```
postgres@bcf6240d7a5f:~$ psql -U postgres -c "CREATE DATABASE new_kntxtdb;"
CREATE DATABASE
postgres@bcf6240d7a5f:~$ pg_restore -d new_kntxtdb -t music_schema.artists_backup -t music_schema.tracks_backup -t music_schema.revenues_backup /var/lib/postgresql/backups/custom_backup.dump
postgres@bcf6240d7a5f:~$ psql -U postgres -d new_kntxtdb
psql (15.6 (Debian 15.6-1.pgdg120+2))
Type "help" for help.

new_kntxtdb=# \dt music_schema.*
Did not find any relation named "music_schema.*".
new_kntxtdb=# \t
Tuples only is on.
new_kntxtdb=# \dt
Did not find any relations.
```
Вижу что доупстил ошибку, а именно забыл что нужно создать и схему в новой базе данных, поэтому возвращаюсь в командную сроку контейнера, где создам схему для новой базы данных и заново сделаю воссоздание и проверю что все успешно:
```
new_kntxtdb=# \q
postgres@bcf6240d7a5f:~$ psql -U postgres -d new_kntxtdb -c "CREATE SCHEMA music_schema;"
CREATE SCHEMA
postgres@bcf6240d7a5f:~$ pg_restore -d new_kntxtdb --schema=music_schema -t artists_backup -t tracks_backup -t revenues_backup /var/lib/postgresql/backups/custom_backup.dump
postgres@bcf6240d7a5f:~$ psql -U postgres -d new_kntxtdb
psql (15.6 (Debian 15.6-1.pgdg120+2))
Type "help" for help.

new_kntxtdb=# \dt music_schema.*
                 List of relations
    Schema    |      Name       | Type  |  Owner
--------------+-----------------+-------+----------
 music_schema | artists_backup  | table | postgres
 music_schema | revenues_backup | table | postgres
 music_schema | tracks_backup   | table | postgres
(3 rows)

new_kntxtdb=# SELECT * FROM music_schema.artists_backup;
 id |        name
----+--------------------
  1 | Alignment
  2 | Amazingblaze
  3 | Charlotte de Witte
  4 | Chris Liebing
  5 | Indira Paganotto
  6 | Monoloc
  7 | ONYVAA
(7 rows)

new_kntxtdb=# SELECT * FROM music_schema.tracks_backup ORDER BY id DESC LIMIT 10;
 id  |       name       | bpm | release_date | artist_id
-----+------------------+-----+--------------+-----------
 100 | Techno Track 100 | 134 | 2023-08-03   |         3
  99 | Techno Track 99  | 131 | 2023-10-30   |         3
  98 | Techno Track 98  | 136 | 2023-07-23   |         3
  97 | Techno Track 97  | 129 | 2023-09-25   |         3
  96 | Techno Track 96  | 135 | 2023-06-11   |         3
  95 | Techno Track 95  | 138 | 2023-12-19   |         3
  94 | Techno Track 94  | 132 | 2023-10-23   |         3
  93 | Techno Track 93  | 136 | 2023-08-19   |         3
  92 | Techno Track 92  | 129 | 2023-10-06   |         3
  91 | Techno Track 91  | 136 | 2022-08-06   |         3
(10 rows)

new_kntxtdb=# SELECT * FROM music_schema.revenues_backup ORDER BY id DESC LIMIT 10;
 id  | track_id | artist_id |                 source
-----+----------+-----------+----------------------------------------
 100 |       39 |         3 | {"amount": 999.60, "source": "Others"}
  99 |       30 |         3 | {"amount": 999.07, "source": "Others"}
  98 |      100 |         3 | {"amount": 976.19, "source": "Others"}
  97 |       70 |         3 | {"amount": 971.10, "source": "Others"}
  96 |       96 |         3 | {"amount": 969.66, "source": "Others"}
  95 |       53 |         3 | {"amount": 963.85, "source": "Others"}
  94 |       78 |         3 | {"amount": 961.25, "source": "Others"}
  93 |       72 |         3 | {"amount": 948.38, "source": "Others"}
  92 |       87 |         3 | {"amount": 937.42, "source": "Others"}
  91 |       97 |         3 | {"amount": 933.85, "source": "Others"}
(10 rows)

```
В итоги вижу, что все успешно восстановилось в новую базу данных.