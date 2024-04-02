# Задание №5

## 1. Создать инстанс ВМ с 2 ядрами и 4 Гб ОЗУ и SSD 10GB
***Создал в Yandex Cloud виртуальную машину (ВМ) с параметрами:***
```
OS: Ubuntu 22.04
vCPU: 4
RAM: 8 GB
Disk space: 20 GB
```
## 2. Установить на него PostgreSQL 15 с дефолтными настройками
***Устанавливаю PostgreSQL 15 в ВМ:***
```
karussia@kntxt-vm:~$ sudo apt update
...
karussia@kntxt-vm:~$ sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
karussia@kntxt-vm:~$ wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
Warning: apt-key is deprecated. Manage keyring files in trusted.gpg.d instead (see apt-key(8)).
OK
karussia@kntxt-vm:~$ sudo apt update
...
karussia@kntxt-vm:~$ sudo apt install postgresql-15
...
karussia@kntxt-vm:~$ sudo systemctl status postgresql.service
● postgresql.service - PostgreSQL RDBMS
     Loaded: loaded (/lib/systemd/system/postgresql.service; enabled; vendor preset: enabled)
     Active: active (exited) since Sun 2024-03-31 19:28:44 UTC; 13min ago
   Main PID: 4394 (code=exited, status=0/SUCCESS)
        CPU: 1ms

Mar 31 19:28:44 kntxt-vm systemd[1]: Starting PostgreSQL RDBMS...
Mar 31 19:28:44 kntxt-vm systemd[1]: Finished PostgreSQL RDBMS.

```
## 3. Создать БД для тестов: выполнить pgbench -i postgres
***Создаю базу данных:***
```
karussia@kntxt-vm:~$ sudo -i -u postgres
postgres@kntxt-vm:~$ pgbench -i postgres
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.07 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 1.19 s (drop tables 0.00 s, create tables 0.00 s, client-side generate 0.09 s, vacuum 0.04 s, primary keys 1.05 s).
```
## 4. Запустить pgbench -c8 -P 6 -T 60 -U postgres postgres
***Исполняю команду `pgbench -c8 -P 6 -T 60 -U postgres postgres`:***
```
postgres@kntxt-vm:~$ pgbench -c8 -P 6 -T 60 -U postgres postgres
pgbench (15.6 (Ubuntu 15.6-1.pgdg22.04+1))
starting vacuum...end.
progress: 6.0 s, 882.3 tps, lat 9.032 ms stddev 6.265, 0 failed
progress: 12.0 s, 554.8 tps, lat 14.416 ms stddev 14.351, 0 failed
progress: 18.0 s, 785.3 tps, lat 10.189 ms stddev 7.902, 0 failed
progress: 24.0 s, 848.0 tps, lat 9.435 ms stddev 6.713, 0 failed
progress: 30.0 s, 738.8 tps, lat 10.815 ms stddev 8.150, 0 failed
progress: 36.0 s, 789.5 tps, lat 10.138 ms stddev 6.632, 0 failed
progress: 42.0 s, 490.0 tps, lat 16.330 ms stddev 57.637, 0 failed
progress: 48.0 s, 838.8 tps, lat 9.530 ms stddev 8.145, 0 failed
progress: 54.0 s, 822.5 tps, lat 9.733 ms stddev 7.403, 0 failed
progress: 60.0 s, 750.2 tps, lat 10.667 ms stddev 8.160, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 45010
number of failed transactions: 0 (0.000%)
latency average = 10.662 ms
latency stddev = 16.842 ms
initial connection time = 15.157 ms
tps = 750.173318 (without initial connection time)
```
## 5. Применить параметры настройки PostgreSQL из прикрепленного к материалам занятия файла
***В файле `/etc/postgresql/15/main/postgresql.conf` применяю следующие настройки:***
```
max_connections = 40
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 512MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 500
random_page_cost = 4.0
effective_io_concurrency = 2
work_mem = 6553kB
min_wal_size = 4GB
max_wal_size = 16GB
```
***сохраняю и делаю рестарт postgresql***
```
sudo systemctl restart postgresql
```
## 6. Протестировать заново
***Запускаю еще раз `pgbench -c8 -P 6 -T 60 -U postgres postgres`:***
```
postgres@kntxt-vm:~$ pgbench -c8 -P 6 -T 60 -U postgres postgres
pgbench (15.6 (Ubuntu 15.6-1.pgdg22.04+1))
starting vacuum...end.
progress: 6.0 s, 764.0 tps, lat 10.427 ms stddev 7.650, 0 failed
progress: 12.0 s, 772.7 tps, lat 10.357 ms stddev 9.208, 0 failed
progress: 18.0 s, 764.5 tps, lat 10.461 ms stddev 6.901, 0 failed
progress: 24.0 s, 605.7 tps, lat 13.119 ms stddev 13.429, 0 failed
progress: 30.0 s, 670.5 tps, lat 12.020 ms stddev 11.520, 0 failed
progress: 36.0 s, 775.2 tps, lat 10.319 ms stddev 10.105, 0 failed
progress: 42.0 s, 753.3 tps, lat 10.606 ms stddev 8.136, 0 failed
progress: 48.0 s, 788.7 tps, lat 10.155 ms stddev 7.687, 0 failed
progress: 54.0 s, 575.0 tps, lat 13.823 ms stddev 18.902, 0 failed
progress: 60.0 s, 693.5 tps, lat 11.609 ms stddev 10.243, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 42986
number of failed transactions: 0 (0.000%)
latency average = 11.164 ms
latency stddev = 10.624 ms
initial connection time = 15.311 ms
tps = 716.470088 (without initial connection time)
```
## 7. Что изменилось и почему?
***Опишу свои наблюдения:***
Сравнивая результаты тестов нагрузки pgbench до и после применения настроек, можно заметить несколько ключевых различий в производительности PostgreSQL.
- До Применения Настроек (с дефолтными параметрами):
1. Общее количество обработанных транзакций: 45010
2. Средняя задержка: 10.662 мс
3. Стандартное отклонение задержки: 16.842 мс
4. Транзакции в секунду (TPS): 750.173318 (без учета времени начального соединения)
- После Применения Настроек:
1. Общее количество обработанных транзакций: 42986
2. Средняя задержка: 11.164 мс
3. Стандартное отклонение задержки: 10.624 мс
4. Транзакции в секунду (TPS): 716.470088 (без учета времени начального соединения)
- Анализ Различий:
1. Количество транзакций: После применения настроек количество обработанных транзакций уменьшилось. Это может быть связано с тем, что некоторые из примененных настроек увеличивают нагрузку на систему (например, увеличение maintenance_work_mem и размеров WAL файлов), что может повлиять на общую производительность при определенных условиях тестирования.
2. Средняя задержка: Средняя задержка увеличилась незначительно. Это может быть результатом большей нагрузки на систему из-за более высоких требований к памяти и дисковой подсистеме.
3. Стандартное отклонение задержки: Стандартное отклонение задержки снизилось, что указывает на более стабильное время ответа системы после применения настроек. Это может быть результатом улучшенного управления памятью и эффективности кеширования.
4. Транзакции в секунду (TPS): Показатель TPS снизился после применения настроек. Это может указывать на то, что система тратит больше ресурсов на обработку каждой транзакции из-за увеличенного объема операций ввода-вывода или обработки данных в памяти.

## 8. Создать таблицу с текстовым полем и заполнить случайными или сгенерированными данным в размере 1млн строк
***Создам таблицу с текстовым полем и заполню её данными в размере 1 млн строк. Буду использовать функцию `generate_series`, чтобы сгенерировать данные:
```
karussia@kntxt-vm:~$ sudo -i -u postgres
postgres@kntxt-vm:~$ psql
psql (15.6 (Ubuntu 15.6-1.pgdg22.04+1))
Type "help" for help.

postgres=# CREATE DATABASE techno_music;
CREATE DATABASE
postgres=# \q
postgres@kntxt-vm:~$ psql -d techno_music
psql (15.6 (Ubuntu 15.6-1.pgdg22.04+1))
Type "help" for help.

techno_music=# CREATE TABLE techno_events (
    event_id SERIAL PRIMARY KEY,
    artist_name VARCHAR(100),
    event_date DATE,
    location VARCHAR(100),
    attendees INTEGER
);
CREATE TABLE
techno_music=# INSERT INTO techno_events (artist_name, event_date, location, attendees)
SELECT
    ('{Charlotte de Witte,Adam Beyer,Hi-Lo,Amelie Lens}'::text[])[floor(random()*4)+1],
    '2022-01-01'::date + (random() * (365*3))::int,
    ('{Berlin,Amsterdam,Barcelona,Ibiza}'::text[])[floor(random()*4)+1],
    floor(random()* (5000-100+1) + 100)
FROM generate_series(1, 1000000);
INSERT 0 1000000
```
***Создание таблицы и генерация данных отработало успешно.***
## 9. Посмотреть размер файла с таблицей
***Проверяю размер файла с таблицей techno_events:***
```
techno_music=# SELECT pg_size_pretty(pg_total_relation_size('techno_events'));
 pg_size_pretty
----------------
 84 MB
(1 row)
```
## 10. 5 раз обновить все строчки и добавить к каждой строчке любой символ
***Для таблицы techno_events можно увеличить количество посетителей на 1 для каждого события, представив, что к каждому событию добавился дополнительный посетитель. Это действие повторю 5 раз:***
```
techno_music=# DO $$
BEGIN
    FOR i IN 1..5 LOOP
        UPDATE techno_events SET attendees = attendees + 1;
    END LOOP;
END$$;
DO
```
## 11. Посмотреть количество мертвых строчек в таблице и когда последний раз приходил автовакуум
***Проверяю количество мертвых точке и время отработки автовакуума для моей таблицы techno_events***
```
techno_music=# SELECT n_dead_tup, last_autovacuum
FROM pg_stat_user_tables
WHERE relname = 'techno_events';
 n_dead_tup |        last_autovacuum
------------+-------------------------------
          0 | 2024-04-02 20:43:51.615305+00
(1 row)
```
## 12. Подождать некоторое время, проверяя, пришел ли автовакуум
## 13. 5 раз обновить все строчки и добавить к каждой строчке любой символ
***Делаю цикл из 5 обновлений еще раз и проверяю данные по времени работы автовакуума:***
```
techno_music=# DO $$
BEGIN
    FOR i IN 1..5 LOOP
        UPDATE techno_events SET attendees = attendees + 1;
    END LOOP;
END$$;
DO
techno_music=# SELECT n_dead_tup, last_autovacuum
FROM pg_stat_user_tables
WHERE relname = 'techno_events';
 n_dead_tup |       last_autovacuum
------------+------------------------------
          0 | 2024-04-02 20:54:50.49729+00
(1 row)
```
## 14. Посмотреть размер файла с таблицей
***Проверяю размер файла с таблицей techno_events:***
```
techno_music=# SELECT pg_size_pretty(pg_total_relation_size('techno_events'));
 pg_size_pretty
----------------
 460 MB
(1 row)
```
## 15. Отключить Автовакуум на конкретной таблице
***Отключаю автовакуум:***
```
techno_music=# ALTER TABLE techno_events SET (autovacuum_enabled = false);
ALTER TABLE
```
## 16. 10 раз обновить все строчки и добавить к каждой строчке любой символ
***Запускаю цикл из 10 кругов обновления записей, добавляя на одного посетителя мероприятия больше:***
```
techno_music=# DO $$
BEGIN
    FOR i IN 1..10 LOOP
        UPDATE techno_events SET attendees = attendees + 1;
    END LOOP;
END$$;
DO
```
## 17. Посмотреть размер файла с таблицей
***Проверяю размер файла с таблицей techno_events:***
```
techno_music=# SELECT pg_size_pretty(pg_total_relation_size('techno_events'));
 pg_size_pretty
----------------
 815 MB
(1 row)
```
## 18. Объясните полученный результат
***Проверяю на всякий случай как выглядит статиститка после отключения автовакуума и продолжения работы с записями:***
```
techno_music=# SELECT n_dead_tup, last_autovacuum
FROM pg_stat_user_tables
WHERE relname = 'techno_events';
 n_dead_tup |       last_autovacuum
------------+------------------------------
   10000000 | 2024-04-02 20:54:50.49729+00
(1 row)

```
***После применения различных операций обновления и изменения настроек автовакуума могу выделить следующие ключевые моменты:***
1. Увеличение размера таблицы: На шагах 9 и 14 мы видели, как размер файла таблицы techno_events увеличивался с 84 МБ до 460 МБ, а затем до 815 МБ после последовательных обновлений. Это увеличение размера обусловлено тем, что каждое обновление строки создаёт "мертвую" или устаревшую версию этой строки, которая должна быть очищена процессом автовакуума. Однако, когда автовакуум был отключён, эти "мертвые" строки накапливались, увеличивая размер файла таблицы.
2. Эффективность автовакуума: Автовакуум — это процесс, который помогает управлять использованием пространства, очищая устаревшие данные. В моменты, когда автовакуум был включён, я видел, что количество мертвых строк контролировалось, и пространство использовалось более эффективно. После его отключения (шаг 15), накопление мертвых строк привело к значительному увеличению размера таблицы.
3. Важность регулярной очистки: Отключение автовакуума демонстрирует необходимость регулярной очистки для поддержания производительности и эффективного использования пространства в базе данных. Накопление мертвых строк не только увеличивает размер таблицы, но и может снизить производительность запросов из-за дополнительных затрат на обработку и пропуск этих строк.

## 19. Не забудьте включить автовакуум)
***Включаю обратно автовакуум:***
```
techno_music=# ALTER TABLE techno_events SET (autovacuum_enabled = true);
ALTER TABLE
```
## 20. Написать анонимную процедуру, в которой в цикле 10 раз обновятся все строчки в искомой таблице. Не забыть вывести номер шага цикла.
***Создаю процедуру для повторного обновления данных в таблице techno_events, увеличивая количество посетителей с объявление номера шага:***
```
techno_music=# DO $$
DECLARE
    i INT;
BEGIN
    FOR i IN 1..10 LOOP
        RAISE NOTICE 'Шаг %', i;
        UPDATE techno_events SET attendees = attendees + 1;
    END LOOP;
END$$;
NOTICE:  Шаг 1
NOTICE:  Шаг 2
NOTICE:  Шаг 3
NOTICE:  Шаг 4
NOTICE:  Шаг 5
NOTICE:  Шаг 6
NOTICE:  Шаг 7
NOTICE:  Шаг 8
NOTICE:  Шаг 9
NOTICE:  Шаг 10
DO

```
***и также проверю как отработал автовакуум и размер файла с таблицей techno_events:***
```
techno_music=# SELECT n_dead_tup, last_autovacuum
FROM pg_stat_user_tables
WHERE relname = 'techno_events';
 n_dead_tup |        last_autovacuum
------------+-------------------------------
          0 | 2024-04-02 21:17:59.766455+00
(1 row)

techno_music=# SELECT pg_size_pretty(pg_total_relation_size('techno_events'));
 pg_size_pretty
----------------
 815 MB
(1 row)
```
***В итоги размер файла увеличился всего на 1 MB.***