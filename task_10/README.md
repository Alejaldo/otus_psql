# Задание №10 "Репликация"

## 0. Создание ВМ и установка PostgeSQL 15 на каждой из ВМ
Использую Yandex Cloud и создаю 4 виртуальные машины (ВМ), каждую с характеристиками:
```
OS: Ubuntu 22.04
Platform: Intel Ice Lake
vCPU: 4
RAM: 8 GB
Disk space: 20 GB
```
Теперь захожу поочередно в каждую из ВМ и устанавливаю PostgeSQL 15:
- kntxt-vm-1:
```
karussia@kntxt-vm-1:~$ sudo apt update
...
karussia@kntxt-vm-1:~$ wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
Warning: apt-key is deprecated. Manage keyring files in trusted.gpg.d instead (see apt-key(8)).
OK
karussia@kntxt-vm-1:~$ sudo apt update
...
karussia@kntxt-vm-1:~$ sudo apt install postgresql-15 -y
...
karussia@kntxt-vm-1:~$ sudo systemctl status postgresql
● postgresql.service - PostgreSQL RDBMS
     Loaded: loaded (/lib/systemd/system/postgresql.service; enabled; vendor preset: enabled)
     Active: active (exited) since Fri 2024-05-24 22:48:53 UTC; 26s ago
   Main PID: 4458 (code=exited, status=0/SUCCESS)
        CPU: 1ms

May 24 22:48:53 kntxt-vm-1 systemd[1]: Starting PostgreSQL RDBMS...
May 24 22:48:53 kntxt-vm-1 systemd[1]: Finished PostgreSQL RDBMS.
```
kntxt-vm-2:
```
...
karussia@kntxt-vm-2:~$ sudo systemctl status postgresql
● postgresql.service - PostgreSQL RDBMS
     Loaded: loaded (/lib/systemd/system/postgresql.service; enabled; vendor preset: enabled)
     Active: active (exited) since Fri 2024-05-24 22:49:03 UTC; 19s ago
   Main PID: 4165 (code=exited, status=0/SUCCESS)
        CPU: 1ms

May 24 22:49:03 kntxt-vm-2 systemd[1]: Starting PostgreSQL RDBMS...
May 24 22:49:03 kntxt-vm-2 systemd[1]: Finished PostgreSQL RDBMS.
```
kntxt-vm-3:
```
...
karussia@kntxt-vm-3:~$ sudo systemctl status postgresql
● postgresql.service - PostgreSQL RDBMS
     Loaded: loaded (/lib/systemd/system/postgresql.service; enabled; vendor preset: enabled)
     Active: active (exited) since Fri 2024-05-24 22:48:58 UTC; 28s ago
   Main PID: 4198 (code=exited, status=0/SUCCESS)
        CPU: 1ms

May 24 22:48:58 kntxt-vm-3 systemd[1]: Starting PostgreSQL RDBMS...
May 24 22:48:58 kntxt-vm-3 systemd[1]: Finished PostgreSQL RDBMS.
```
kntxt-vm-4:
```
...
karussia@kntxt-vm-4:~$ sudo systemctl status postgresql
● postgresql.service - PostgreSQL RDBMS
     Loaded: loaded (/lib/systemd/system/postgresql.service; enabled; vendor preset: enabled)
     Active: active (exited) since Fri 2024-05-24 22:49:00 UTC; 28s ago
   Main PID: 4664 (code=exited, status=0/SUCCESS)
        CPU: 1ms

May 24 22:49:00 kntxt-vm-4 systemd[1]: Starting PostgreSQL RDBMS...
May 24 22:49:00 kntxt-vm-4 systemd[1]: Finished PostgreSQL RDBMS.
```
Теперь буду настраивать файл pg_hba.conf на каждой ВМ чтобы объединить эти машины между собой:
kntxt-vm-1:
```
sudo nano /etc/postgresql/15/main/pg_hba.conf
...
host    replication     all             158.160.9.148/32          md5
host    replication     all             51.250.111.207/32         md5
host    replication     all             158.160.24.54/32          md5

sudo systemctl reload postgresql
```
kntxt-vm-2:
```
sudo nano /etc/postgresql/15/main/pg_hba.conf
...
host    replication     all             51.250.96.220/32          md5
host    replication     all             51.250.111.207/32         md5
host    replication     all             158.160.24.54/32          md5

sudo systemctl reload postgresql
```
kntxt-vm-3:
```
sudo nano /etc/postgresql/15/main/pg_hba.conf
...
host    replication     all             51.250.96.220/32          md5
host    replication     all             158.160.9.148/32          md5
host    replication     all             158.160.24.54/32          md5

sudo systemctl reload postgresql
```
kntxt-vm-4:
```
sudo nano /etc/postgresql/15/main/pg_hba.conf
...
host    replication     all             51.250.96.220/32          md5
host    replication     all             158.160.9.148/32          md5
host    replication     all             51.250.111.207/32         md5

sudo systemctl reload postgresql
```

Теперь все 4 ВМ готовы.
## 1. На первой ВМ создаем таблицы test для записи, test2 для запросов на чтение.
Теперь на ВМ kntxt-vm-1 создаю необходимые 2 таблицы:
```
karussia@kntxt-vm-1:~$ sudo -i -u postgres
postgres@kntxt-vm-1:~$ psql
psql (15.7 (Ubuntu 15.7-1.pgdg22.04+1))
Type "help" for help.

postgres=# CREATE DATABASE technolabels;
CREATE DATABASE
postgres=# \c technolabels
You are now connected to database "technolabels" as user "postgres".
technolabels=# CREATE TABLE test (
    id SERIAL PRIMARY KEY,
    data VARCHAR(100)
);
CREATE TABLE
technolabels=# CREATE TABLE test2 (
    id SERIAL PRIMARY KEY,
    data VARCHAR(100)
);
CREATE TABLE
technolabels=# \dt
         List of relations
 Schema | Name  | Type  |  Owner
--------+-------+-------+----------
 public | test  | table | postgres
 public | test2 | table | postgres
(2 rows)

```
## 2. Создаем публикацию таблицы test и подписываемся на публикацию таблицы test2 с ВМ №2.
Далее на ВМ kntxt-vm-1 создаю PUBLICATION:
```
technolabels=# CREATE PUBLICATION my_pub_test FOR TABLE test;
WARNING:  wal_level is insufficient to publish logical changes
HINT:  Set wal_level to "logical" before creating subscriptions.
CREATE PUBLICATION
```
Теперь на ВМ kntxt-vm-2 создаю таблицу test2:
```
karussia@kntxt-vm-2:~$ sudo -i -u postgres
postgres@kntxt-vm-2:~$ psql
psql (15.7 (Ubuntu 15.7-1.pgdg22.04+1))
Type "help" for help.

postgres=# CREATE DATABASE technolabels;
CREATE DATABASE
postgres=# \c technolabels
You are now connected to database "technolabels" as user "postgres".
technolabels=# CREATE TABLE test2 (
    id SERIAL PRIMARY KEY,
    data VARCHAR(100)
);
CREATE TABLE
```
Также как и на первой ВМ создаю PUBLICATION:
```
technolabels=# CREATE PUBLICATION my_pub_test2 FOR TABLE test2;
WARNING:  wal_level is insufficient to publish logical changes
HINT:  Set wal_level to "logical" before creating subscriptions.
CREATE PUBLICATION
```
Возврашаюсь к первой ВМ и создаю подписку на таблицу test2 второй ВМ, но сначала задам пароль на второй ВМ у пользователя postgres:
```
technolabels=# exit
postgres@kntxt-vm-2:~$ psql
psql (15.7 (Ubuntu 15.7-1.pgdg22.04+1))
Type "help" for help.

postgres=# ALTER USER postgres PASSWORD 'secretpassword';
ALTER ROLE
```
Итак, вот теперь на первой ВМ произвожу подписку:
```
technolabels=# CREATE SUBSCRIPTION my_sub_test2 CONNECTION 'host=158.160.9.148 port=5432 dbname=technolabels user=postgres password=secretpassword' PUBLICATION my_pub_test2;
ERROR:  could not connect to the publisher: connection to server at "158.160.9.148", port 5432 failed: Connection refused
  Is the server running on that host and accepting TCP/IP connections?
```
Понимаю что забыл на второй ВМ в файле postgresql.conf настроить listen_addresses, возвращаюсь к этому файлу и устанавливаю listen_addresses = '*'
и еще добавляю запись "host    all             all             51.250.96.220/32        md5" в файл /etc/postgresql/15/main/pg_hba.conf
Теперь на первой ВМ проверяю работает ли связь:
```
karussia@kntxt-vm-1:~$ sudo -i -u postgres
postgres@kntxt-vm-1:~$ psql -h 158.160.9.148 -U postgres -d technolabels -W
Password:
psql (15.7 (Ubuntu 15.7-1.pgdg22.04+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, compression: off)
Type "help" for help.

technolabels=
```
Вижу что теперь все работает, пробую снова сделать подписку:
```
technolabels=# exit
postgres@kntxt-vm-1:~$ CREATE SUBSCRIPTION my_sub_test2 CONNECTION 'host=158.160.9.148 port=5432 dbname=technolabels user=postgres password=secretpassword' PUBLICATION my_pub_test2;
CREATE: command not found
postgres@kntxt-vm-1:~$ psql
psql (15.7 (Ubuntu 15.7-1.pgdg22.04+1))
Type "help" for help.

postgres=# \c technolabels
You are now connected to database "technolabels" as user "postgres".
technolabels=# CREATE SUBSCRIPTION my_sub_test2 CONNECTION 'host=158.160.9.148 port=5432 dbname=technolabels user=postgres password=secretpassword' PUBLICATION my_pub_test2;
ERROR:  could not create replication slot "my_sub_test2": ERROR:  logical decoding requires wal_level >= logical
```
Вижу теперь ошибку что в файле postgresql.conf на ВМ первой и второй скорректировать значение wal_level на logical (по умолчанию там значение "#wal_level = replica")\
После чего не забываю перезагрузить командой "sudo systemctl restart postgresql" на обеих машинах.\
Проверяю еще раз:
```
karussia@kntxt-vm-1:~$ sudo -i -u postgres
postgres@kntxt-vm-1:~$ psql -d technolabels
psql (15.7 (Ubuntu 15.7-1.pgdg22.04+1))
Type "help" for help.

technolabels=# CREATE SUBSCRIPTION my_sub_test2 CONNECTION 'host=158.160.9.148 port=5432 dbname=technolabels user=postgres password=secretpassword' PUBLICATION my_pub_test2;
NOTICE:  created replication slot "my_sub_test2" on publisher
CREATE SUBSCRIPTION
```
Теперь подписка создана

## 3. На 2 ВМ создаем таблицы test2 для записи, test для запросов на чтение.
Таблица test2 на второй ВМ уже была создана на предыдущем шаге, теперь создаем таблицу test:
```
karussia@kntxt-vm-2:~$ sudo -i -u postgres
postgres@kntxt-vm-2:~$ psql
psql (15.7 (Ubuntu 15.7-1.pgdg22.04+1))
Type "help" for help.

postgres=# \c technolabels
You are now connected to database "technolabels" as user "postgres".
technolabels=# CREATE TABLE test (
    id SERIAL PRIMARY KEY,
    data VARCHAR(100)
);
CREATE TABLE
```


## 4. Создаем публикацию таблицы test2 и подписываемся на публикацию таблицы test1 с ВМ №1.
"CREATE PUBLICATION my_pub_test2 FOR TABLE test2;" - уже производился для test2 на втором шаге.\
На стороне первой ВМ задаю пароль пользователю postgres:
```
postgres@kntxt-vm-1:~$ psql
psql (15.7 (Ubuntu 15.7-1.pgdg22.04+1))
Type "help" for help.

postgres=# ALTER USER postgres PASSWORD 'secretpassword'
postgres-# ;
```
добавляю значение "listen_addresses = '*'" в файл /etc/postgresql/15/main/postgresql.conf,
а также запись "host    all             all             158.160.9.148/32          md5" в файл /etc/postgresql/15/main/pg_hba.conf,
после чего на сторое второй ВМ создаю нужную подписку:
```
technolabels=# CREATE SUBSCRIPTION my_sub_test CONNECTION 'host=51.250.96.220 port=5432 dbname=technolabels user=postgres password=secretpassword' PUBLICATION my_pub_test;
NOTICE:  created replication slot "my_sub_test" on publisher
CREATE SUBSCRIPTION
```
Подписка упешно создана.

## 5. 3 ВМ использовать как реплику для чтения и бэкапов (подписаться на таблицы из ВМ №1 и №2 ).
Теперь на третьей ВМ настрою /etc/postgresql/15/main/postgresql.conf в части установки wal_level в значение logical и listen_addresses = '*':
```
karussia@kntxt-vm-3:~$ sudo nano /etc/postgresql/15/main/postgresql.conf
...
listen_addresses = '*'
...
wal_level = logical
...
karussia@kntxt-vm-3:~$ sudo systemctl restart postgresql
```
и добавляю записи в /etc/postgresql/15/main/pg_hba.conf файл:
```
host    all             all             51.250.96.220/32          md5
host    all             all             158.160.9.148/32          md5
```
Затем надо добавить аналогичное в ВМ первой и второй:
kntxt-vm-1 и kntxt-vm-2:
```
host    all             all             51.250.111.207/32         md5
```
Создаю БД и таблицы на 3 ВМ:
```
karussia@kntxt-vm-3:~$ sudo -i -u postgres
postgres@kntxt-vm-3:~$ psql
psql (15.7 (Ubuntu 15.7-1.pgdg22.04+1))
Type "help" for help.

postgres=# CREATE DATABASE technolabels;
CREATE DATABASE
postgres=# \c technolabels
You are now connected to database "technolabels" as user "postgres"
technolabels=# CREATE TABLE IF NOT EXISTS test (
    id SERIAL PRIMARY KEY,
    data VARCHAR(100)
);
CREATE TABLE
technolabels=# CREATE TABLE IF NOT EXISTS test2 (
    id SERIAL PRIMARY KEY,
    data VARCHAR(100)
);
CREATE TABLE
```

Теперь делаю ряд проверок, чтобы убедиться в корректности работы:\
В kntxt-vm-1:
```
karussia@kntxt-vm-1:~$ sudo -i -u postgres
postgres@kntxt-vm-1:~$ psql -d technolabels
psql (15.7 (Ubuntu 15.7-1.pgdg22.04+1))
Type "help" for help.

technolabels=# \dt
         List of relations
 Schema | Name  | Type  |  Owner
--------+-------+-------+----------
 public | test  | table | postgres
 public | test2 | table | postgres
(2 rows)

technolabels=# \dRp+
                          Publication my_pub_test
  Owner   | All tables | Inserts | Updates | Deletes | Truncates | Via root
----------+------------+---------+---------+---------+-----------+----------
 postgres | f          | t       | t       | t       | t         | f
Tables:
    "public.test"

technolabels=# INSERT INTO test (data) VALUES ('test data 1');
INSERT 0 1

```
В kntxt-vm-2:
```
karussia@kntxt-vm-2:~$ sudo -i -u postgres
postgres@kntxt-vm-2:~$ psql -d technolabels
psql (15.7 (Ubuntu 15.7-1.pgdg22.04+1))
Type "help" for help.

technolabels=# \dt
         List of relations
 Schema | Name  | Type  |  Owner
--------+-------+-------+----------
 public | test  | table | postgres
 public | test2 | table | postgres
(2 rows)

technolabels=# \dRp+
                          Publication my_pub_test2
  Owner   | All tables | Inserts | Updates | Deletes | Truncates | Via root
----------+------------+---------+---------+---------+-----------+----------
 postgres | f          | t       | t       | t       | t         | f
Tables:
    "public.test2"

technolabels=# INSERT INTO test2 (data) VALUES ('test data 2');
INSERT 0 1

```
В kntxt-vm-3:
```
technolabels=# SELECT * FROM test;
 id |    data
----+-------------
  1 | test data 1
(1 row)

technolabels=# SELECT * FROM test2;
 id |    data
----+-------------
  1 | test data 2
(1 row)

```
После проверок вижу что на 3 ВМ можно просматривать внось добавленные данные в таблицы, значит все успешно настроилось.

## 6. реализовать горячее реплицирование для высокой доступности на 4ВМ. Источником должна выступать ВМ №3.
Для начала на 3 ВМ делаю настройку в файле /etc/postgresql/15/main/postgresql.conf следующих параметров:
```
wal_level = replica
wal_keep_size = 64
hot_standby = on
listen_addresses = '*'
```
На 4 ВМ добавляю запись "host    replication     all             158.160.24.54/32          md5" в файл "/etc/postgresql/15/main/pg_hba.conf"\
И меняю пароль юзера:
```
karussia@kntxt-vm-4:~$ sudo -i -u postgres
postgres@kntxt-vm-4:~$ psql
psql (15.7 (Ubuntu 15.7-1.pgdg22.04+1))
Type "help" for help.

postgres=# ALTER USER postgres PASSWORD 'secretpassword';
ALTER ROLE
```
Затем на обеих машинах делаю рестарт "sudo systemctl restart postgresql"\
На 4 ВМ делаю остановку PostgreSQL и очищаю директорию /var/lib/postgresql/15/main/:
```
karussia@kntxt-vm-4:~$ sudo systemctl stop postgresql
karussia@kntxt-vm-4:~$ sudo rm -rf /var/lib/postgresql/15/main/*
```
На 3 ВМ буду запускать pg_basebackup:
```
karussia@kntxt-vm-3:~$ sudo -i -u postgres
postgres@kntxt-vm-3:~$ pg_basebackup -h 158.160.24.54 -D /var/lib/postgresql/15/main -U postgres -P -v --wal-method=stream
Password:
pg_basebackup: error: directory "/var/lib/postgresql/15/main" exists but is not empty
```
Вижу что надо произвести очистку директория на 3 ВМ:
```
postgres@kntxt-vm-3:~$ exit
logout
karussia@kntxt-vm-3:~$ sudo systemctl stop postgresql
karussia@kntxt-vm-3:~$ sudo rm -rf /var/lib/postgresql/15/main
karussia@kntxt-vm-3:~$ sudo mkdir /var/lib/postgresql/15/main
karussia@kntxt-vm-3:~$ sudo chown postgres:postgres /var/lib/postgresql/15/main
karussia@kntxt-vm-3:~$ sudo chmod 700 /var/lib/postgresql/15/main
karussia@kntxt-vm-3:~$ sudo -i -u postgres
postgres@kntxt-vm-3:~$ pg_basebackup -h 158.160.24.54 -D /var/lib/postgresql/15/main -U postgres -P -v --wal-method=stream
Password:
pg_basebackup: initiating base backup, waiting for checkpoint to complete
pg_basebackup: checkpoint completed
pg_basebackup: write-ahead log start point: 0/2000028 on timeline 1
pg_basebackup: starting background WAL receiver
pg_basebackup: created temporary replication slot "pg_basebackup_6899"
22987/22987 kB (100%), 1/1 tablespace
pg_basebackup: write-ahead log end point: 0/2000100
pg_basebackup: waiting for background process to finish streaming ...
pg_basebackup: syncing data to disk ...
pg_basebackup: renaming backup_manifest.tmp to backup_manifest
pg_basebackup: base backup completed
```
Забыл, что нужно установить пароль также и тут на 3 ВМ, поэтому запускаю снова PostgreSQL и устанавлю этот пароль:
```
karussia@kntxt-vm-3:~$ sudo -i -u postgres
postgres@kntxt-vm-3:~$ psql
psql: error: connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: No such file or directory
  Is the server running locally and accepting connections on that socket?
postgres@kntxt-vm-3:~$ exit
logout
karussia@kntxt-vm-3:~$ sudo systemctl start postgresql
karussia@kntxt-vm-3:~$ sudo -i -u postgres
postgres@kntxt-vm-3:~$ psql
psql (15.7 (Ubuntu 15.7-1.pgdg22.04+1))
Type "help" for help.

postgres=# ALTER USER postgres PASSWORD 'secretpassword';
ALTER ROLE

```
Теперь на 4 ВМ создаю файл standby.signal:
```
karussia@kntxt-vm-4:~$ sudo -i -u postgres
postgres@kntxt-vm-4:~$ touch /var/lib/postgresql/15/main/standby.signal
```
Потом в файле /etc/postgresql/15/main/postgresql.conf параметрам primary_conninfo, primary_slot_name и restore_command присваиваю значения:
```
primary_conninfo = 'host=51.250.111.207 port=5432 user=postgres password=secretpassword'
primary_slot_name = 'replication_slot_4'
restore_command = 'cp /path_to_archive/%f %p'
```

Затем исполняю
```
karussia@kntxt-vm-4:~$ sudo chown -R postgres:postgres /var/lib/postgresql/15/main
karussia@kntxt-vm-4:~$ sudo systemctl start postgresql

```
Проверяю статус

```
 pid | status | receive_start_lsn | receive_start_tli | written_lsn | flushed_lsn | received_tli | last_msg_send_time | last_msg_receipt_time | latest_end_lsn | latest_end_time | slot_name | sender_host | sender_port | conninfo
-----+--------+-------------------+-------------------+-------------+-------------+--------------+--------------------+-----------------------+----------------+-----------------+-----------+-------------+-------------+----------
(0 rows)

```
Делаю увдления Replication Slot:

На первой ВМ
```
karussia@kntxt-vm-1:~$ sudo -i -u postgres
postgres@kntxt-vm-1:~$ psql
psql (15.7 (Ubuntu 15.7-1.pgdg22.04+1))
Type "help" for help.

postgres=# SELECT * FROM pg_replication_slots;
postgres=# SELECT pg_drop_replication_slot('sub_test_vm1');
 pg_drop_replication_slot
--------------------------

(1 row)

postgres=#

```

На второй ВМ
```
karussia@kntxt-vm-2:~$ sudo -i -u postgres
postgres@kntxt-vm-2:~$ psql
psql (15.7 (Ubuntu 15.7-1.pgdg22.04+1))
Type "help" for help.

postgres=# SELECT * FROM pg_replication_slots;
postgres=# SELECT pg_drop_replication_slot('sub_test2_vm2');
 pg_drop_replication_slot
--------------------------

(1 row)

postgres=#
```
На третьей ВМ наоборот настраиваю подписку:
```
postgres=# \c technolabels
You are now connected to database "technolabels" as user "postgres".
technolabels=# CREATE SUBSCRIPTION sub_test_vm1 CONNECTION 'host=51.250.96.220 port=5432 dbname=technolabels user=postgres password=secretpassword' PUBLICATION my_pub_test;
NOTICE:  created replication slot "sub_test_vm1" on publisher
CREATE SUBSCRIPTION
technolabels=# CREATE SUBSCRIPTION sub_test2_vm2 CONNECTION 'host=158.160.9.148 port=5432 dbname=technolabels user=postgres password=secretpassword' PUBLICATION my_pub_test2;
NOTICE:  created replication slot "sub_test2_vm2" on publisher
CREATE SUBSCRIPTION

```

Теперь добавляю данных на первой и второй машинах:
```
karussia@kntxt-vm-1:~$ sudo -i -u postgres
postgres@kntxt-vm-1:~$ psql -d technolabels
psql (15.7 (Ubuntu 15.7-1.pgdg22.04+1))
Type "help" for help.

technolabels=# INSERT INTO test (data) VALUES ('test data from vm-1');
INSERT 0 1
technolabels=#

```
+
```
karussia@kntxt-vm-2:~$ sudo -i -u postgres
postgres@kntxt-vm-2:~$ psql -d technolabels
psql (15.7 (Ubuntu 15.7-1.pgdg22.04+1))
Type "help" for help.

technolabels=# INSERT INTO test2 (data) VALUES ('test data from vm-2');
INSERT 0 1
technolabels=#
```
Проверяю на третьей:
```
technolabels=# SELECT * FROM test;
 id |        data
----+---------------------
  1 | test data 1
  2 | test data 1
  3 | test data 1
  4 | test data from vm-1
(4 rows)

technolabels=# SELECT * FROM test2;
 id |        data
----+---------------------
  1 | test data 2
  2 | test data 2
  3 | test data from vm-2
(3 rows)

technolabels=#
```

на 4 ВМ пока есть ошибки:
```
karussia@kntxt-vm-4:~$ sudo tail -n 50 /var/log/postgresql/postgresql-15-main.log
cp: cannot stat '/path_to_archive/00000002.history': No such file or directory
2024-05-25 02:07:17.151 UTC [7560] LOG:  waiting for WAL to become available at 0/30001D8
cp: cannot stat '/path_to_archive/000000010000000000000003': No such file or directory
2024-05-25 02:07:22.150 UTC [9758] FATAL:  could not start WAL streaming: ERROR:  replication slot "replication_slot_4" does not exist
cp: cannot stat '/path_to_archive/00000002.history': No such file or directory
...
cp: cannot stat '/path_to_archive/00000002.history': No such file or directory
2024-05-25 02:08:17.186 UTC [7560] LOG:  waiting for WAL to become available at 0/30001D8

```
Нахожу что забыл написать актуальное значение для restore_command в файле /etc/postgresql/15/main/postgresql.conf и произвожу деятельнось по остановке и очистке директория как на 3 ВМ:
```
karussia@kntxt-vm-4:~$ sudo nano /etc/postgresql/15/main/postgresql.conf
karussia@kntxt-vm-4:~$ sudo systemctl stop postgresql
karussia@kntxt-vm-4:~$ sudo rm -rf /var/lib/postgresql/15/main/*
karussia@kntxt-vm-4:~$ sudo -i -u postgres
postgres@kntxt-vm-4:~$ pg_basebackup -h 51.250.111.207 -D /var/lib/postgresql/15/main -U postgres -P -v --wal-method=stream
Password:
pg_basebackup: error: directory "/var/lib/postgresql/15/main" exists but is not empty
postgres@kntxt-vm-4:~$ exit
logout
karussia@kntxt-vm-4:~$ sudo rm -rf /var/lib/postgresql/15/main
karussia@kntxt-vm-4:~$ sudo mkdir /var/lib/postgresql/15/main
karussia@kntxt-vm-4:~$ sudo chown postgres:postgres /var/lib/postgresql/15/main
karussia@kntxt-vm-4:~$ sudo chmod 700 /var/lib/postgresql/15/main
karussia@kntxt-vm-4:~$ sudo -i -u postgres
postgres@kntxt-vm-4:~$ pg_basebackup -h 51.250.111.207 -D /var/lib/postgresql/15/main -U postgres -P -v --wal-method=stream
Password:
pg_basebackup: initiating base backup, waiting for checkpoint to complete
pg_basebackup: checkpoint completed
pg_basebackup: write-ahead log start point: 0/5000028 on timeline 1
pg_basebackup: starting background WAL receiver
pg_basebackup: created temporary replication slot "pg_basebackup_7460"
30686/30686 kB (100%), 1/1 tablespace
pg_basebackup: write-ahead log end point: 0/5000100
pg_basebackup: waiting for background process to finish streaming ...
pg_basebackup: syncing data to disk ...
pg_basebackup: renaming backup_manifest.tmp to backup_manifest
pg_basebackup: base backup completed
postgres@kntxt-vm-4:~$ exit
logout
karussia@kntxt-vm-4:~$ touch /var/lib/postgresql/15/main/standby.signal
touch: cannot touch '/var/lib/postgresql/15/main/standby.signal': Permission denied
karussia@kntxt-vm-4:~$ sudo -i -u postgres
postgres@kntxt-vm-4:~$ touch /var/lib/postgresql/15/main/standby.signal
postgres@kntxt-vm-4:~$ exit
logout
karussia@kntxt-vm-4:~$ sudo nano /etc/postgresql/15/main/postgresql.conf
karussia@kntxt-vm-4:~$ sudo mkdir -p /var/lib/postgresql/15/wal_archive
karussia@kntxt-vm-4:~$ sudo chown postgres:postgres /var/lib/postgresql/15/wal_archive
karussia@kntxt-vm-4:~$ sudo chmod 700 /var/lib/postgresql/15/wal_archive
karussia@kntxt-vm-4:~$ sudo systemctl start postgresql
karussia@kntxt-vm-4:~$ sudo tail -f /var/log/postgresql/postgresql-15-main.log
cp: cannot stat '/var/lib/postgresql/15/wal_archive/000000010000000000000005': No such file or directory
2024-05-25 02:36:02.058 UTC [11675] LOG:  recovered replication state of node 1 to 0/1970700
2024-05-25 02:36:02.058 UTC [11675] LOG:  recovered replication state of node 2 to 0/19A9E78
2024-05-25 02:36:02.103 UTC [11675] LOG:  redo starts at 0/5000028
cp: cannot stat '/var/lib/postgresql/15/wal_archive/000000010000000000000006': No such file or directory
2024-05-25 02:36:02.123 UTC [11675] LOG:  completed backup recovery with redo LSN 0/5000028 and end LSN 0/5000100
2024-05-25 02:36:02.123 UTC [11675] LOG:  consistent recovery state reached at 0/5000100
2024-05-25 02:36:02.123 UTC [11672] LOG:  database system is ready to accept read-only connections
cp: cannot stat '/var/lib/postgresql/15/wal_archive/000000010000000000000006': No such file or directory
2024-05-25 02:36:02.156 UTC [11684] LOG:  started streaming WAL from primary at 0/6000000 on timeline 1
^C
karussia@kntxt-vm-4:~$ sudo -i -u postgres
postgres@kntxt-vm-4:~$ psql -c "SELECT * FROM pg_stat_wal_receiver;"
pid  |  status   | receive_start_lsn | receive_start_tli | written_lsn | flushed_lsn | received_tli |      last_msg_send_time       |    last_msg_receipt_time     | latest_end_lsn |        latest_end_time        |     slot_name      |  sender_host   | sender_port |                                                                                                                                                                    conninfo
-------+-----------+-------------------+-------------------+-------------+-------------+--------------+-------------------------------+------------------------------+----------------+-------------------------------+--------------------+----------------+-------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 11684 | streaming | 0/6000000         |                 1 | 0/6000060   | 0/6000060   |            1 | 2024-05-25 02:37:02.218146+00 | 2024-05-25 02:37:02.21866+00 | 0/6000060      | 2024-05-25 02:36:02.155834+00 | replication_slot_4 | 51.250.111.207 |        5432 | user=postgres password=******** channel_binding=prefer dbname=replication host=51.250.111.207 port=5432 fallback_application_name=15/main sslmode=prefer sslcompression=0 sslcertmode=allow sslsni=1 ssl_min_protocol_version=TLSv1.2 gssencmode=prefer krbsrvname=postgres gssdelegation=0 target_session_attrs=any load_balance_hosts=disable
(1 row)
```
Теперь проверяю:
```
karussia@kntxt-vm-4:~$ sudo -i -u postgres
postgres@kntxt-vm-4:~$ psql -c "SELECT * FROM pg_stat_wal_receiver;"
postgres@kntxt-vm-4:~$ psql -d technolabels
psql (15.7 (Ubuntu 15.7-1.pgdg22.04+1))
Type "help" for help.

technolabels=# SELECT * FROM test;
 id |        data
----+---------------------
  1 | test data 1
  2 | test data 1
  3 | test data 1
  4 | test data from vm-1
(4 rows)

technolabels=# SELECT * FROM test2;
 id |        data
----+---------------------
  1 | test data 2
  2 | test data 2
  3 | test data from vm-2
(3 rows)

technolabels=#

```
И в целом вижу, что данные добавленные в таблицах на первой и второй ВМ могу видеть на 4 ВМ через 3 ВМ.

---

# Итоговый вывод

Постараюсь теперь структурировать все то, что удалось сделать в данном таске.
   
1. **Настройка `pg_hba.conf` на каждой ВМ:**
   - **Цель:** Включить соединения репликации между ВМ.
   - **Детали:** Файл `pg_hba.conf` обновляется на каждой ВМ, чтобы разрешить соединения репликации с IP-адресов других ВМ. Это гарантирует, что каждая ВМ может взаимодействовать для целей репликации.

2. **Создание таблиц и публикаций:**
   - **Цель:** Настроить таблицы для репликации.
   - **Детали:** 
     - На **ВМ 1** создаются две таблицы (`test` и `test2`).
     - Создается публикация (`my_pub_test`) для таблицы `test`.
     - На **ВМ 2** создается таблица `test2`, и создается публикация (`my_pub_test2`) для таблицы `test2`.
   - **Кейс с проблемой:** Настройка `wal_level` на `logical` для включения логической репликации.

3. **Подписка на публикации:**
   - **Цель:** Настроить подписки для репликации данных.
   - **Детали:** 
     - На **ВМ 1** создается подписка на `my_pub_test2` (опубликованную ВМ 2).
     - На **ВМ 2** создается подписка на `my_pub_test` (опубликованную ВМ 1).
   - **Кейс с проблемой:** Обеспечение корректной настройки `listen_addresses` и конфигурации для репликации.

4. **Настройка ВМ 3 в качестве реплики только для чтения:**
   - **Цель:** Использовать ВМ 3 в качестве реплики только для чтения для балансировки нагрузки и создания резервных копий.
   - **Детали:** 
     - Файл `postgresql.conf` ВМ 3 настроен с `wal_level = logical` и `listen_addresses = '*'`.
     - ВМ 3 подписывается на публикации с ВМ 1 и ВМ 2, что позволяет ей реплицировать обе таблицы.
   - **Проверка:** Вставка данных в таблицы на ВМ 1 и ВМ 2 и проверка репликации на ВМ 3.

5. **Настройка горячей репликации на ВМ 4:**
   - **Цель:** Создать горячую реплику для обеспечения высокой доступности.
   - **Детали:** 
     - **ВМ 3** настроена в качестве основного сервера для потоковой репликации с следующими параметрами:
       - `archive_mode = on`
       - `archive_command = 'cp %p /var/lib/postgresql/15/wal_archive/%f'`
       - `wal_level = replica`
       - `wal_keep_size = 64`
       - `max_wal_senders = 10`
       - `max_replication_slots = 10`
     - Эти настройки включают архивирование WAL и настраивают процессы отправки WAL, необходимые для репликации.
     - **ВМ 4** настроена для работы в горячем резерве:
       - `primary_conninfo` указывает детали подключения к основному серверу (ВМ 3).
       - `primary_slot_name` указывает слот репликации для использования.
       - `restore_command` указывает команду для восстановления архивированных файлов WAL.
     - Создается файл `standby.signal`, чтобы перевести ВМ 4 в режим ожидания.
     - После настройки ВМ 4 успешно запускается в режиме только для чтения, получая WAL-журналы от ВМ 3.

#### Значение параметров конфигурации

- **primary_conninfo:** 
  - **Цель:** Указывает детали подключения к основному серверу для репликации.
  - **Значение:** Обеспечивает подключение реплики к основному серверу для потоковой передачи WAL-файлов.

- **primary_slot_name:** 
  - **Цель:** Указывает слот репликации на основном сервере.
  - **Значение:** Обеспечивает удержание WAL-файлов основным сервером до подтверждения получения репликой, предотвращая преждевременное удаление файлов WAL.

- **restore_command:**
  - **Цель:** Указывает, как восстанавливать архивированные файлы WAL.
  - **Значение:** Обеспечивает возможность реплики догнать основную базу данных, используя архивированные файлы WAL, если она отстала от текущего WAL потока.

#### Основные параметры на ВМ 3 (Основной для горячей реплики):

- **archive_mode = on:** 
  - **Цель:** Включает архивирование WAL-файлов.
  - **Значение:** Важно для восстановления файлов WAL на резервной реплике в случае необходимости.

- **archive_command:** 
  - **Цель:** Указывает команду для архивирования WAL-файлов.
  - **Значение:** Обеспечивает копирование файлов WAL в каталог, доступный для резервной реплики.

- **wal_level = replica:** 
  - **Цель:** Устанавливает уровень WAL для поддержки репликации.
  - **Значение:** Необходимо как для потоковой репликации, так и для логической репликации.

- **wal_keep_size:** 
  - **Цель:** Указывает количество файлов WAL, которые нужно сохранять.
  - **Значение:** Предотвращает преждевременное удаление файлов WAL, обеспечивая возможность реплики догнать основную базу данных.

- **max_wal_senders и max_replication_slots:**
  - **Цель:** Указывает количество процессов отправки WAL и слотов репликации.
  - **Значение:** Обеспечивает достаточное количество процессов и слотов для всех реплик.

#### Роли ВМ 3 и ВМ 4

- **ВМ 3 (Основной для горячей реплики):**
  - **Роль:** Действует как основной сервер, отправляющий WAL-журналы на ВМ 4. Также обрабатывает запросы на чтение и служит хранилищем резервных копий для логической репликации с ВМ 1 и ВМ 2.
  - **Функция:** Обеспечивает точку непрерывности для репликации, гарантируя, что изменения данных на ВМ 1 и ВМ 2 фиксируются и отправляются на ВМ 4.

- **ВМ 4 (Горячая реплика):**
  - **Роль:** Действует как горячая реплика, готовая взять на себя работу в случае отказа ВМ 3.
  - **Функция:** Обеспечивает высокую доступность, поддерживая почти реальную копию данных основного сервера, что минимизирует время простоя и потерю данных в случае отказа основного сервера.

#### Финальные мысли

Эта настройка достигает надежной и высокодоступной среды PostgreSQL с следующими функциями:
- **Балансировка нагрузки:** ВМ 3 может обрабатывать запросы на чтение, распределяя нагрузку.
- **Высокая доступность:** ВМ 4 действует как горячая реплика, готовая взять на себя работу в случае отказа ВМ 3.
- **Целостность данных:** Логическая репликация обеспечивает распространение изменений с ВМ 1 и ВМ 2 на ВМ 3 и далее на ВМ 4.

Эта конфигурация называется **"[Каскадная репликация](https://postgrespro.ru/docs/postgrespro/10/warm-standby#CASCADING-REPLICATION)"**, где одна реплика реплицируется от другой реплики, а не напрямую от основного сервера. Этот подход помогает снизить нагрузку на основной сервер и обеспечивает масштабируемое решение для высокой доступности и восстановления после сбоев.