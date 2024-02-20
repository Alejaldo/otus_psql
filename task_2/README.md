# Задание №2

## 0. создать ВМ с Ubuntu 20.04/22.04 или развернуть докер любым удобным способом
***Виртуальная машина уже создана в Yandex Cloud (4 GB RAM, 15 GB HDD, vCPU 2, Ubuntu 22.04 LTS) с привязкой ssh локальной машины, подключение делаю командой c терминала ноутбука***
```
ssh -i ~/.ssh/id_rsa chamonix@158.160.134.54
```
## 1. Поставить на виртуальной машине YandexCloud Docker Engine
***Ставлю докер в ВМ YandexCloud:***
```
sudo apt update
sudo apt install docker.io
```
***Проверяю версию установленного доекра:***
```
$ docker --version
Docker version 24.0.5, build 24.0.5-0ubuntu1~22.04.1
```
## 2. Сделать каталог /var/lib/postgres
***Создаю нужную папку***
```
sudo mkdir /var/lib/postgres
```
## 3. Развернуть контейнер с PostgreSQL 15 смонтировав в него /var/lib/postgresql
***Создаю сеть***
```
$ sudo docker network create chamonix-net
5c79c777ed35ccce85e3258384527e0dd266c6b5a8abf8e97a376f93acb9849f
```
***Затем создаю контейнер-сервер:***
```
$ sudo docker run --name kntxt-server --network chamonix-net -e POSTGRES_PASSWORD=mysecretpassword -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15
aeff6742cf3c6ffbc139550c9099e6d506596ea1dcf3b536541355ab9aa5fddb
```
## 4. Развернуть контейнер с клиентом postgres
***Запускаю клиент-контейнер который удлаиться после выхода выхода из psql сессии***
```
$ sudo docker run -it --rm --name kntxt-client --network chamonix-net postgres:15 psql -h kntxt-server -U postgres
Password for user postgres:
psql (15.5 (Debian 15.5-1.pgdg120+1))
Type "help" for help.

postgres=#
```
## 5. Подключится из контейнера с клиентом к контейнеру с сервером и сделать таблицу с парой строк
***На предыдущем шаге совершено подключение из клиент-контейнера kntxt-client к сервер-контейнеру kntxt-server, теперь создаю новую базу данных данных***
```
postgres=# CREATE DATABASE kntxt_db;
CREATE DATABASE
postgres=# \c kntxt_db
You are now connected to database "kntxt_db" as user "postgres".
kntxt_db=#

```
***создаю в базе kntxt_db таблицу techno_tracks***
```
kntxt_db=# CREATE TABLE techno_tracks (
    id SERIAL PRIMARY KEY,
    track_name TEXT,
    sub_genre TEXT,
    artist_name TEXT,
    planned_release_date DATE,
    beat_rate INT,
    label_name TEXT,
    sale_price DECIMAL,
    commission_rate DECIMAL
);
CREATE TABLE
```
***и добавляю несколько записей в таблицу:***
```
kntxt_db=# INSERT INTO techno_tracks (track_name, sub_genre, artist_name, planned_release_date, beat_rate, label_name, sale_price, commission_rate) VALUES
('Track 1', 'ACID techno', 'Charlotte de Witte', '2023-01-01', 128, 'KNTXT', 10.99, 15.0),
('Track 2', 'Industrial techno', 'Hi-Lo', '2023-02-01', 130, 'Heldeep', 12.99, 12.0);
INSERT 0 2
```
***И проверяю что записи были успешно добавлены***
```
kntxt_db=# SELECT * FROM techno_tracks;
 id | track_name |     sub_genre     |    artist_name     | planned_release_date | beat_rate | label_name | sale_price | commission_rate
----+------------+-------------------+--------------------+----------------------+-----------+------------+------------+-----------------
  1 | Track 1    | ACID techno       | Charlotte de Witte | 2023-01-01           |       128 | KNTXT      |      10.99 |            15.0
  2 | Track 2    | Industrial techno | Hi-Lo              | 2023-02-01           |       130 | Heldeep    |      12.99 |            12.0
(2 rows)
```
## 6. Подключится к контейнеру с сервером с ноутбука/компьютера извне инстансов GCP/ЯО/места установки докера
***Делаю подключение к контейнеру с сервером с ноутбука из нового окна терминала сразу к базе данных kntxt_db:***
```
psql -p 5432 -h 158.160.134.54 -U postgres -d kntxt_db
Password for user postgres:
psql (15.3 (Ubuntu 15.3-1.pgdg18.04+1), server 15.5 (Debian 15.5-1.pgdg120+1))
Type "help" for help.

kntxt_db=#

```
***Проверяю данные в таблице:***
```
kntxt_db=# SELECT * FROM techno_tracks;
 id | track_name |     sub_genre     |    artist_name     | planned_release_date | beat_rate | label_name | sale_price | commission_rate
----+------------+-------------------+--------------------+----------------------+-----------+------------+------------+-----------------
  1 | Track 1    | ACID techno       | Charlotte de Witte | 2023-01-01           |       128 | KNTXT      |      10.99 |            15.0
  2 | Track 2    | Industrial techno | Hi-Lo              | 2023-02-01           |       130 | Heldeep    |      12.99 |            12.0
(2 rows)

```
## 7. Удалить контейнер с сервером
***Выхожу из подключения с ноутбука к серверу и выхожу из сессии контейнер-клиента коммандой `exit` и в терминальном окне с виртуальной машиной останавливаю сервер-контейнер:***
```
$ sudo docker stop kntxt-server
kntxt-server
```
***затем удаляю:***
```
$ sudo docker rm kntxt-server
kntxt-server
```
***Проверяю список активных контейнеров чтобы убедиться что контейнер-сервер был удален***
```
$ sudo docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
```
***записей не вижу, затем проверяю список всех контейнеров***
```
sudo docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED             STATUS                  PORTS     NAMES
```
***и свой сервер-контейнер kntxt-server не вижу, значит он был удален успешно***
## 8. Создать его заново
***Создаю заново сервер-контейнер kntxt-server***
```
$ sudo docker run --name kntxt-server --network chamonix-net -e POSTGRES_PASSWORD=mysecretpassword -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15
343b60d05f8d9004157dc2eb78191f202adb510b309f41605d917186a6614d30
```
## 9. Подключится снова из контейнера с клиентом к контейнеру с сервером
***Теперь создаю контейнер-клиент kntxt-client подключаясь к kntxt-server:***
```
$ sudo docker run -it --rm --name kntxt-client --network chamonix-net postgres:15 psql -h kntxt-server -U postgres
Password for user postgres:
psql (15.5 (Debian 15.5-1.pgdg120+1))
Type "help" for help.

postgres=#
```
## 10. Проверить, что данные остались на месте
***Перехожу в базу kntxt_db***
```
postgres=# \c kntxt_db
You are now connected to database "kntxt_db" as user "postgres".
kntxt_db=#
```
***вижу что подключение к базе прошло успешно, проверяю данные:***
```
kntxt_db=# SELECT * FROM techno_tracks;
 id | track_name |     sub_genre     |    artist_name     | planned_release_date | beat_rate | label_name | sale_price | commission_rate
----+------------+-------------------+--------------------+----------------------+-----------+------------+------------+-----------------
  1 | Track 1    | ACID techno       | Charlotte de Witte | 2023-01-01           |       128 | KNTXT      |      10.99 |            15.0
  2 | Track 2    | Industrial techno | Hi-Lo              | 2023-02-01           |       130 | Heldeep    |      12.99 |            12.0
(2 rows)
```
***вижу что все записи есть и они без изменений.***

## Выводы
***В рамках данной задачи был использован подход Docker Volumes для хранения данных базы данных
Команда -v /var/lib/postgres:/var/lib/postgresql/data, создала постоянное хранилище на хост-системе (/var/lib/postgres) и монтировла его в контейнер (/var/lib/postgresql/data).
Все данные, сохраняемые PostgreSQL, фактически хранятся на хост-системе вне самого контейнера. Даже когда контейнер удаляется, данные на хост-системе остаются нетронутыми.***
