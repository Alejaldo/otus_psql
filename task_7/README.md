# Задание №7

## 1. Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд. Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.
***Создаю в Yandex Cloud виртуальную машину (ВМ) с параметрами:***
```
OS: Ubuntu 22.04
vCPU: 4
RAM: 8 GB
Disk space: 20 GB
```
***Устанавливаю PostgreSQL 15:***
```
karussia@kntxt-vm:~$ sudo apt update
karussia@kntxt-vm:~$
karussia@kntxt-vm:~$ wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
karussia@kntxt-vm:~$ sudo apt update
karussia@kntxt-vm:~$ sudo apt install postgresql-15 -y
karussia@kntxt-vm:~$ sudo systemctl status postgresql
● postgresql.service - PostgreSQL RDBMS
     Loaded: loaded (/lib/systemd/system/postgresql.service; enabled; vendor preset: enabled)
     Active: active (exited) since Fri 2024-05-10 07:12:11 UTC; 18min ago
   Main PID: 4245 (code=exited, status=0/SUCCESS)
        CPU: 1ms

May 10 07:12:11 kntxt-vm systemd[1]: Starting PostgreSQL RDBMS...
May 10 07:12:11 kntxt-vm systemd[1]: Finished PostgreSQL RDBMS
```
***Открываю файл конфигурации PostgreSQL /etc/postgresql/15/main/postgresql.conf:***
```
karussia@kntxt-vm:~$ sudo nano /etc/postgresql/15/main/postgresql.conf
```
***в файле нахожу следующие строки, которые по умолчанию закомментированы:***
```
#log_lock_waits = off                   # log lock waits >= deadlock_timeout
...
#deadlock_timeout = 1s
```
***Расскомментирую эти 2 строки и меняю у них значения***
```
log_lock_waits = on         # включает логирование ожидания блокировок
deadlock_timeout = 200ms    # установите порог времени для логирования
```
***Таким образом включаю логирование ожидания блокировок (`log_lock_waits = on`) и установливаю порог времени для логирования 200 миллисекунд (deadlock_timeout = 200ms)***
***Затем перезапускаю PostgreSQL для применения изменений:***
```
karussia@kntxt-vm:~$ sudo systemctl restart postgresql
```
***Далее создам базу данных музыкального фестиваля с таблицами для дальнейшего использования***
```
karussia@kntxt-vm:~$ sudo -i -u postgres
postgres@kntxt-vm:~$ psql
psql (15.7 (Ubuntu 15.7-1.pgdg22.04+1))
Type "help" for help.

postgres=# CREATE DATABASE awakenings_summer_fest_2024;
CREATE DATABASE
postgres=# \c awakenings_summer_fest_2024
You are now connected to database "awakenings_summer_fest_2024" as user "postgres".
awakenings_summer_fest_2024=# CREATE TABLE managers (
    id SERIAL PRIMARY KEY,
    full_name VARCHAR(255),
    email VARCHAR(255),
    telephone VARCHAR(50)
);
CREATE TABLE
awakenings_summer_fest_2024=# CREATE TABLE areas (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255),
    capacity INTEGER,
    manager_id INTEGER REFERENCES managers(id)
);
CREATE TABLE
awakenings_summer_fest_2024=# CREATE TABLE artists (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255),
    performance_date DATE,
    performance_time TIME,
    area_id INTEGER REFERENCES areas(id)
);
CREATE TABLE
awakenings_summer_fest_2024=# INSERT INTO managers (full_name, email, telephone) VALUES
('Jeroen van Aken', 'j.vanaken@awakenings.nl', '+310123456789'),
('Lotte de Vries', 'l.devries@awakenings.nl', '+310987654321'),
('Pieter Bakker', 'p.bakker@awakenings.nl', '+310456789123'),
('Emma Boer', 'e.boer@awakenings.nl', '+310321654987');
INSERT 0 4
awakenings_summer_fest_2024=# INSERT INTO areas (name, capacity, manager_id) VALUES
('AREA V', 10000, 1),
('AREA Y', 5000, 2),
('AREA U', 3000, 3),
('AREA W', 3000, 4),
('AREA X', 2000, 1),
('AREA C', 2000, 2),
('AREA D', 2000, 3),
('AREA H', 2000, 4);
INSERT 0 8
awakenings_summer_fest_2024=# INSERT INTO artists (name, performance_date, performance_time, area_id) VALUES
('999999999', '2024-07-05', '17:00', (SELECT id FROM areas WHERE name = 'AREA V')),
('CERA', '2024-07-05', '18:00', (SELECT id FROM areas WHERE name = 'AREA V')),
('KHIN', '2024-07-05', '19:00', (SELECT id FROM areas WHERE name = 'AREA V')),
('DEBORAH DE LUCA', '2024-07-05', '20:00', (SELECT id FROM areas WHERE name = 'AREA V')),
('I HATE MODELS', '2024-07-05', '21:00', (SELECT id FROM areas WHERE name = 'AREA V')),
('NICO MORENO', '2024-07-05', '22:00', (SELECT id FROM areas WHERE name = 'AREA V')),
('SPACE 92', '2024-07-05', '23:00', (SELECT id FROM areas WHERE name = 'AREA V'))
...
('CHRIS STUSSY', '2024-07-07', '19:00', (SELECT id FROM areas WHERE name = 'AREA C')),
('CINCITY', '2024-07-07', '20:00', (SELECT id FROM areas WHERE name = 'AREA C')),
('SAMUEL DEEP', '2024-07-07', '23:00', (SELECT id FROM areas WHERE name = 'AREA H'));EA H')),
INSERT 0 48
awakenings_summer_fest_2024=# SELECT COUNT(*) FROM artists;
 count
-------
   118
(1 row)
```
***Для воспроизведения ситуации с блокировками будут использовать 2 сессии (открою второй терминал со входом в ВМ)***
В первом терминале находясь уже в базе данных awakenings_summer_fest_2024 начинаю транзакцию обновления времени выступления артиста:
```
awakenings_summer_fest_2024=# BEGIN;
BEGIN
awakenings_summer_fest_2024=*# UPDATE artists SET performance_time = '17:30' WHERE name = 'CERA' AND performance_date = '2024-07-05';
UPDATE 1
```
Затем во втором терминале захожу в эту же базу данных и попытаюсь выполнить аналогичное обновление:
```
karussia@kntxt-vm:~$ sudo -i -u postgres
postgres@kntxt-vm:~$ psql
psql (15.7 (Ubuntu 15.7-1.pgdg22.04+1))
Type "help" for help.

postgres=# \c awakenings_summer_fest_2024
You are now connected to database "awakenings_summer_fest_2024" as user "postgres".
awakenings_summer_fest_2024=# UPDATE artists SET performance_time = '18:30' WHERE name = 'CERA' AND performance_date = '2024-07-05';
```
Этот запрос теперь ожидает освобождения блокировки, удерживаемой первой сессией
Теперь возвращаюст в первую сессию и выполняю COMMIT:
```
COMMIT;
```
теперь во втором терминале вижу
```
UPDATE 1
```
***Теперь проверяю данные из журнала логов***
```
karussia@kntxt-vm:~$ sudo cat /var/log/postgresql/postgresql-15-main.log
2024-05-10 07:12:15.553 UTC [5211] LOG:  starting PostgreSQL 15.7 (Ubuntu 15.7-1.pgdg22.04+1) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 11.4.0-1ubuntu1~22.04) 11.4.0, 64-bit
2024-05-10 07:12:15.553 UTC [5211] LOG:  listening on IPv6 address "::1", port 5432
2024-05-10 07:12:15.553 UTC [5211] LOG:  listening on IPv4 address "127.0.0.1", port 5432
2024-05-10 07:12:15.562 UTC [5211] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"
2024-05-10 07:12:15.570 UTC [5238] LOG:  database system was shut down at 2024-05-10 07:12:12 UTC
2024-05-10 07:12:15.576 UTC [5211] LOG:  database system is ready to accept connections
2024-05-10 07:17:15.670 UTC [5236] LOG:  checkpoint starting: time
2024-05-10 07:17:19.727 UTC [5236] LOG:  checkpoint complete: wrote 43 buffers (0.3%); 0 WAL file(s) added, 0 removed, 0 recycled; write=4.044 s, sync=0.005 s, total=4.057 s; sync files=11, longest=0.003 s, average=0.001 s; distance=252 kB, estimate=252 kB
2024-05-10 07:39:57.874 UTC [5211] LOG:  received fast shutdown request
2024-05-10 07:39:57.877 UTC [5211] LOG:  aborting any active transactions
2024-05-10 07:39:57.881 UTC [5211] LOG:  background worker "logical replication launcher" (PID 5241) exited with exit code 1
2024-05-10 07:39:57.881 UTC [5236] LOG:  shutting down
2024-05-10 07:39:57.883 UTC [5236] LOG:  checkpoint starting: shutdown immediate
2024-05-10 07:39:57.893 UTC [5236] LOG:  checkpoint complete: wrote 0 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.001 s, sync=0.001 s, total=0.013 s; sync files=0, longest=0.000 s, average=0.000 s; distance=0 kB, estimate=227 kB
2024-05-10 07:39:57.898 UTC [5211] LOG:  database system is shut down
2024-05-10 07:39:58.053 UTC [6141] LOG:  starting PostgreSQL 15.7 (Ubuntu 15.7-1.pgdg22.04+1) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 11.4.0-1ubuntu1~22.04) 11.4.0, 64-bit
2024-05-10 07:39:58.053 UTC [6141] LOG:  listening on IPv6 address "::1", port 5432
2024-05-10 07:39:58.053 UTC [6141] LOG:  listening on IPv4 address "127.0.0.1", port 5432
2024-05-10 07:39:58.059 UTC [6141] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"
2024-05-10 07:39:58.067 UTC [6144] LOG:  database system was shut down at 2024-05-10 07:39:57 UTC
2024-05-10 07:39:58.074 UTC [6141] LOG:  database system is ready to accept connections
2024-05-10 07:44:58.167 UTC [6142] LOG:  checkpoint starting: time
2024-05-10 07:44:58.187 UTC [6142] LOG:  checkpoint complete: wrote 3 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.005 s, sync=0.002 s, total=0.021 s; sync files=2, longest=0.001 s, average=0.001 s; distance=0 kB, estimate=0 kB
2024-05-10 11:10:01.721 UTC [6142] LOG:  checkpoint starting: time
2024-05-10 11:11:36.127 UTC [6142] LOG:  checkpoint complete: wrote 944 buffers (5.8%); 0 WAL file(s) added, 0 removed, 0 recycled; write=94.387 s, sync=0.006 s, total=94.406 s; sync files=311, longest=0.005 s, average=0.001 s; distance=4285 kB, estimate=4285 kB
2024-05-10 12:27:41.611 UTC [22021] postgres@postgres ERROR:  relation "artists" does not exist at character 15
2024-05-10 12:27:41.611 UTC [22021] postgres@postgres STATEMENT:  SELECT * FROM artists;
2024-05-10 12:27:49.062 UTC [22021] postgres@postgres ERROR:  relation "artists" does not exist at character 15
2024-05-10 12:27:49.062 UTC [22021] postgres@postgres STATEMENT:  SELECT * FROM artists;
2024-05-10 12:30:02.097 UTC [6142] LOG:  checkpoint starting: time
2024-05-10 12:30:03.529 UTC [6142] LOG:  checkpoint complete: wrote 15 buffers (0.1%); 0 WAL file(s) added, 0 removed, 0 recycled; write=1.408 s, sync=0.004 s, total=1.432 s; sync files=14, longest=0.003 s, average=0.001 s; distance=33 kB, estimate=3860 kB
2024-05-10 12:47:27.277 UTC [22262] postgres@awakenings_summer_fest_2024 LOG:  process 22262 still waiting for ShareLock on transaction 746 after 200.075 ms
2024-05-10 12:47:27.277 UTC [22262] postgres@awakenings_summer_fest_2024 DETAIL:  Process holding the lock: 22038. Wait queue: 22262.
2024-05-10 12:47:27.277 UTC [22262] postgres@awakenings_summer_fest_2024 CONTEXT:  while updating tuple (0,2) in relation "artists"
2024-05-10 12:47:27.277 UTC [22262] postgres@awakenings_summer_fest_2024 STATEMENT:  UPDATE artists SET performance_time = '18:30' WHERE name = 'CERA' AND performance_date = '2024-07-05';
2024-05-10 12:50:02.925 UTC [6142] LOG:  checkpoint starting: time
2024-05-10 12:50:03.147 UTC [6142] LOG:  checkpoint complete: wrote 2 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.201 s, sync=0.006 s, total=0.223 s; sync files=2, longest=0.006 s, average=0.003 s; distance=8 kB, estimate=3474 kB
2024-05-10 12:54:41.356 UTC [22262] postgres@awakenings_summer_fest_2024 LOG:  process 22262 acquired ShareLock on transaction 746 after 434279.554 ms
2024-05-10 12:54:41.356 UTC [22262] postgres@awakenings_summer_fest_2024 CONTEXT:  while updating tuple (0,2) in relation "artists"
2024-05-10 12:54:41.356 UTC [22262] postgres@awakenings_summer_fest_2024 STATEMENT:  UPDATE artists SET performance_time = '18:30' WHERE name = 'CERA' AND performance_date = '2024-07-05';
2024-05-10 12:55:02.193 UTC [6142] LOG:  checkpoint starting: time
2024-05-10 12:55:02.306 UTC [6142] LOG:  checkpoint complete: wrote 2 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.102 s, sync=0.003 s, total=0.113 s; sync files=2, longest=0.002 s, average=0.002 s; distance=8 kB, estimate=3128 kB
```
***где вижу следующее:***
`LOG:  process 22262 still waiting for ShareLock on transaction 746 after 200.075 ms` - это сообщение указывает, что процесс с идентификатором 22262 ожидал освобождения блокировки типа ShareLock более 200 мс.
`DETAIL:  Process holding the lock: 22038. Wait queue: 22262.` - здесь указывается, что процесс 22038 удерживал блокировку, а процесс 22262 находился в очереди ожидания.
`CONTEXT:  while updating tuple (0,2) in relation "artists"` - это сообщение говорит о том, что блокировка возникла в процессе обновления записи в таблице artists.
`STATEMENT:  UPDATE artists SET performance_time = '18:30' WHERE name = 'CERA' AND performance_date = '2024-07-05';` - это мой запрос, который вызвал блокировку, он пытался обновить время выступления артиста "CERA".
`LOG:  process 22262 acquired ShareLock on transaction 746 after 434279.554 ms` - после ожидания (более 7 минут), процесс 22262 наконец получил необходимую блокировку и смог выполнить операцию, что показывает, что блокировка была успешно освобождена после COMMIT.

## 2. Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны. Пришлите список блокировок и объясните, что значит каждая.
***Теперь будут использовать 3 терминала для работы с 3 кейсами***
***В первом терминале***
```
karussia@kntxt-vm:~$ sudo -i -u postgres
postgres@kntxt-vm:~$ psql -d awakenings_summer_fest_2024
psql (15.7 (Ubuntu 15.7-1.pgdg22.04+1))
Type "help" for help.

awakenings_summer_fest_2024=# BEGIN;
BEGIN
awakenings_summer_fest_2024=*# UPDATE artists SET performance_time = '17:45' WHERE name = 'CERA' AND performance_date = '2024-07-05';
UPDATE 1
```
***Во втором терминале***
```
karussia@kntxt-vm:~$ sudo -i -u postgres
postgres@kntxt-vm:~$ psql -d awakenings_summer_fest_2024
psql (15.7 (Ubuntu 15.7-1.pgdg22.04+1))
Type "help" for help.

awakenings_summer_fest_2024=# BEGIN;
BEGIN
awakenings_summer_fest_2024=*# UPDATE artists SET performance_time = '18:45' WHERE name = 'CERA' AND performance_date = '2024-07-05';
```
***В третьем терминале***
```
karussia@kntxt-vm:~$ sudo -i -u postgres
postgres@kntxt-vm:~$ psql -d awakenings_summer_fest_2024
psql (15.7 (Ubuntu 15.7-1.pgdg22.04+1))
Type "help" for help.

awakenings_summer_fest_2024=# BEGIN;
BEGIN
awakenings_summer_fest_2024=*# UPDATE artists SET performance_time = '19:45' WHERE name = 'CERA' AND performance_date = '2024-07-05';
```
***В пером терминале изучу блокировки***
```
SELECT * FROM pg_locks;
   locktype    | database | relation | page | tuple | virtualxid | transactionid | classid | objid | objsubid | virtualtransaction |  pid  |       mode       | granted | fastpath |           waitstart
---------------+----------+----------+------+-------+------------+---------------+---------+-------+----------+--------------------+-------+------------------+---------+----------+-------------------------------
 relation      |    16388 |    16415 |      |       |            |               |         |       |          | 4/631              | 22558 | RowExclusiveLock | t       | t        |
 relation      |    16388 |    16411 |      |       |            |               |         |       |          | 4/631              | 22558 | RowExclusiveLock | t       | t        |
 virtualxid    |          |          |      |       | 4/631      |               |         |       |          | 4/631              | 22558 | ExclusiveLock    | t       | t        |
 relation      |    16388 |    16415 |      |       |            |               |         |       |          | 5/12               | 22646 | RowExclusiveLock | t       | t        |
 relation      |    16388 |    16411 |      |       |            |               |         |       |          | 5/12               | 22646 | RowExclusiveLock | t       | t        |
 virtualxid    |          |          |      |       | 5/12       |               |         |       |          | 5/12               | 22646 | ExclusiveLock    | t       | t        |
 relation      |    16388 |    12073 |      |       |            |               |         |       |          | 3/1053             | 22471 | AccessShareLock  | t       | t        |
 relation      |    16388 |    16415 |      |       |            |               |         |       |          | 3/1053             | 22471 | RowExclusiveLock | t       | t        |
 relation      |    16388 |    16411 |      |       |            |               |         |       |          | 3/1053             | 22471 | RowExclusiveLock | t       | t        |
 virtualxid    |          |          |      |       | 3/1053     |               |         |       |          | 3/1053             | 22471 | ExclusiveLock    | t       | t        |
 transactionid |          |          |      |       |            |           748 |         |       |          | 4/631              | 22558 | ShareLock        | f       | f        | 2024-05-10 13:33:09.598918+00
 tuple         |    16388 |    16411 |    0 |   120 |            |               |         |       |          | 4/631              | 22558 | ExclusiveLock    | t       | f        |
 transactionid |          |          |      |       |            |           750 |         |       |          | 5/12               | 22646 | ExclusiveLock    | t       | f        |
 tuple         |    16388 |    16411 |    0 |   120 |            |               |         |       |          | 5/12               | 22646 | ExclusiveLock    | f       | f        | 2024-05-10 13:34:10.619158+00
 transactionid |          |          |      |       |            |           749 |         |       |          | 4/631              | 22558 | ExclusiveLock    | t       | f        |
 transactionid |          |          |      |       |            |           748 |         |       |          | 3/1053             | 22471 | ExclusiveLock    | t       | f        |
(16 rows)
```
Из данной таблицы вижу следующее:
а) Relation Locks (RowExclusiveLock).
Эти блокировки захватываются при выполнении операций, которые изменяют строки, как например UPDATE. Каждая транзакция, пытающаяся обновить данные в таблице, должна захватить RowExclusiveLock на этой таблице.
У меня есть несколько таких блокировок на таблицах artists и areas (ID таблиц 16415 и 16411 соответственно). Это означает, что транзакции активно работают с этими таблицами.
б) VirtualXID Locks (ExclusiveLock).
Эти блокировки используются для координации между транзакциями, предотвращая взаимные блокировки и другие конфликты.
Показывает, что транзакция активна и должна быть завершена перед тем, как другие процессы смогут взаимодействовать с заблокированными ресурсами.
в) Tuple Locks (ExclusiveLock).
Эти блокировки на уровне строк указывают на то, что конкретные строки таблицы в данный момент обновляются.
В твблице вижу блокировки на кортежах в таблице artists, что свидетельствует о попытках обновления строк.
г) TransactionID Locks (ShareLock и ExclusiveLock).
ExclusiveLock на transactionid обозначает, что транзакция ожидает завершения других транзакций, прежде чем она сможет продолжить выполнение.
ShareLock обычно используется в операциях, которые нуждаются в консистентности данных на протяжении транзакции, например, в запросах SELECT ... FOR SHARE.

В моем случае вижу, что transactionid 748 имеет ShareLock, которая не предоставлена (granted = f). Это означает, что транзакция 748 ожидает освобождения ресурсов, которые заблокированы другими транзакциями.
Транзакции с virtualxid 4/631 и 5/12 держат ExclusiveLock и RowExclusiveLock на таблицах и строках, указывая на активное обновление данных. Одновременно они ждут, когда освободятся блокировки, установленные друг другом или третьими транзакциями.
Все три сеанса активно участвуют в блокировках. Не предоставленные блокировки (granted = f) показывают, где именно происходят задержки из-за ожидания освобождения ресурсов.

## 3. Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?
***Создаю еще кейс для взаимной блокировки в трех терминалах***

***Терминал 1. Начинаю транзакцию обновляя время выступления артиста '999999999' на AREA V***
```
awakenings_summer_fest_2024=# BEGIN;
BEGIN
awakenings_summer_fest_2024=*# UPDATE artists SET performance_time = '17:30' WHERE name = '999999999' AND performance_date = '2024-07-05';
UPDATE 1
```
***Терминал 2. Обновляю время выступления артиста 'JORIS VOORN' на AREA Y, затем обновляю время артиста из первого сеанса***
```
awakenings_summer_fest_2024=# BEGIN;
BEGIN
awakenings_summer_fest_2024=*# UPDATE artists SET performance_time = '20:00' WHERE name = 'JORIS VOORN' AND performance_date = '2024-07-05';
UPDATE 1
awakenings_summer_fest_2024=*# UPDATE artists SET performance_time = '18:00' WHERE name = '999999999' AND performance_date = '2024-07-05';
```
***Терминал 3. Обновляю время выступления артиста 'DJ RUSH' на AREA U, затем пробую обновить время выступления артистов из первого и второго сеанса***
```
awakenings_summer_fest_2024=*# BEGIN;
WARNING:  there is already a transaction in progress
BEGIN
awakenings_summer_fest_2024=*# UPDATE artists SET performance_time = '19:00' WHERE name = 'DJ RUSH' AND performance_date = '2024-07-05';
UPDATE 1
awakenings_summer_fest_2024=*# UPDATE artists SET performance_time = '21:00' WHERE name = 'JORIS VOORN' AND performance_date = '2024-07-05';
UPDATE artists SET performance_time = '17:45' WHERE name = '999999999' AND performance_date = '2024-07-05';
```
Таким образом вижу как во второй и третьей сессии процесс встал, мне данные правки, допустим, не нужны, поэтому исполняю `ROLLBACK;` в первой сессии, затем во второй, после чего все 3 сессии стали свободны.
На всякий случай делаю проверку остутствия текущих блокировок:
```
awakenings_summer_fest_2024=# SELECT * FROM pg_locks WHERE NOT granted;
 locktype | database | relation | page | tuple | virtualxid | transactionid | classid | objid | objsubid | virtualtransaction | pid | mode | granted | fastpath | waitstart
----------+----------+----------+------+-------+------------+---------------+---------+-------+----------+--------------------+-----+------+---------+----------+-----------
(0 rows)
```
Таблица пуста, блокировок нет.

***Анализировать журнал сообщений PostgreSQL (/var/log/postgresql/postgresql-15-main.log) для постфактум понимания взаимоблокировок (deadlocks) возможно, и это может быть очень полезно для диагностики проблем с производительностью и конкуренцией.***
Журналы предоставляют ценную информацию о том, что происходило в момент возникновения блокировки:
а) PostgreSQL записывает в журнал когда обнаруживает взаимоблокировку, указывая какие процессы участвовали и какие ресурсы они ожидали. Пример сообщения:
```
2024-05-10 15:03:39.825 UTC [22646] postgres@awakenings_summer_fest_2024 LOG:  process 22646 still waiting for ShareLock on transaction 749 after 200.166 ms
2024-05-10 15:03:39.825 UTC [22646] postgres@awakenings_summer_fest_2024 DETAIL:  Process holding the lock: 22558. Wait queue: 22646.
```
Эти сообщения дают представление о том, какие сессии были заблокированы, какие идентификаторы транзакций были задействованы, и сколько времени потребовалось для разрешения блокировки.
б) Настройки параметров deadlock_timeout и lock_timeout могут быть найдены в журнале и помогают понять, через какое время система начинает реагировать на взаимоблокировку и как долго транзакции ждут получения блокировки перед тайм-аутом.
```
awakenings_summer_fest_2024=# SHOW deadlock_timeout;
 deadlock_timeout
------------------
 200ms
(1 row)

awakenings_summer_fest_2024=# SHOW lock_timeout;
 lock_timeout
--------------
 0
(1 row)
```
в) Журналы также показывают последовательность SQL-команд, которые были выполнены до возникновения блокировки. Это помогает понять логику взаимодействия транзакций и может указывать на возможные изменения в логике приложения или настройках базы данных для избежания подобных ситуаций в будущем.
г) Журналы также могут содержать информацию о загрузке системы, времени ответа и других параметрах производительности, что важно для комплексного анализа проблемы.

## 4. Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?
***Да, две транзакции могут заблокировать друг друга даже если обе выполняют команду UPDATE на одной и той же таблицы без использования условия WHERE. Это происходит, потому что команда UPDATE без WHERE пытается изменить все строки в таблице, что приводит к установлению блокировки на уровне строк для каждой строки в таблице. Если две такие транзакции запущены одновременно, каждая из них будет ждать, пока другая освободит блокировки на строках, что может привести к взаимоблокировке.***
Сессия 1:
```
awakenings_summer_fest_2024=# BEGIN;
BEGIN
awakenings_summer_fest_2024=*# UPDATE artists SET performance_time = performance_time;
```
Сессия 2:
```
awakenings_summer_fest_2024=# BEGIN;
BEGIN
awakenings_summer_fest_2024=*# UPDATE artists SET performance_time = performance_time;
```
В этом случае, каждая сессия начинает обновлять строки в таблице artists, и как только они начнут обновлять одни и те же строки, они заблокируют друг друга, ожидая освобождения блокировок, поскольку обе транзакции держат блокировки на уровне строк и ждут завершения другой транзакции.
В итоги выйду из созданной ситуации через Ctrl + C и затем ROLLBACK в обоих сеансах:
```
^CCancel request sent
ERROR:  canceling statement due to user request
awakenings_summer_fest_2024=!# ROLLBACK;
ROLLBACK
awakenings_summer_fest_2024=#
```
