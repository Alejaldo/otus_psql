# Задание №6

## 1. Настройте выполнение контрольной точки раз в 30 секунд.
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
karussia@kntxt-vm:~$ sudo apt install wget ca-certificates -y
karussia@kntxt-vm:~$ wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
karussia@kntxt-vm:~$ sudo apt update
karussia@kntxt-vm:~$ sudo apt install postgresql-15 -y
karussia@kntxt-vm:~$ sudo systemctl status postgresql
● postgresql.service - PostgreSQL RDBMS
     Loaded: loaded (/lib/systemd/system/postgresql.service; enabled; vendor preset: enabled)
     Active: active (exited) since Thu 2024-04-18 17:07:47 UTC; 4min 10s ago
   Main PID: 5000 (code=exited, status=0/SUCCESS)
        CPU: 1ms

Apr 18 17:07:47 kntxt-vm systemd[1]: Starting PostgreSQL RDBMS...
Apr 18 17:07:47 kntxt-vm systemd[1]: Finished PostgreSQL RDBMS.
```
***Отредактирую конфигурационный файл PostgreSQL postgresql.conf:***
```
karussia@kntxt-vm:~$ sudo nano /etc/postgresql/15/main/postgresql.conf
```
***в файле нахожу параметр `checkpoint_timeout` который изначально закомментирован и имеет занчение 5min, расскомментирую и поставлю значение 30s:***
```
checkpoint_timeout = 30s
```
***затем перезапускаю PostgreSQL для применения изменений:***
```
karussia@kntxt-vm:~$ sudo systemctl restart postgresql
```

## 2. 10 минут c помощью утилиты pgbench подавайте нагрузку.
***Сначала инициализируйте базу данных для pgbench:***
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
done in 1.15 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 0.09 s, vacuum 0.04 s, primary keys 1.00 s).
```
***запускаю pgbench, подавая нагрузку в течение 10 минут:***
```
postgres@kntxt-vm:~$ pgbench -c 10 -T 600 postgres
pgbench (15.6 (Ubuntu 15.6-1.pgdg22.04+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 413433
number of failed transactions: 0 (0.000%)
latency average = 14.513 ms
initial connection time = 19.825 ms
tps = 689.058154 (without initial connection time)

```

## 3. Измерьте, какой объем журнальных файлов был сгенерирован за это время. Оцените, какой объем приходится в среднем на одну контрольную точку.
***Для измерения объема журнальных файлов (WAL), сгенерированных за время нагрузочного тестирования, проверю размер каталога WAL***
```
postgres@kntxt-vm:~$ cd /var/lib/postgresql/15/main/pg_wal
postgres@kntxt-vm:~/15/main/pg_wal$ du -sh
65M .
```
***Средний размер WAL на 1 checkpoint = Number of Checkpoints/Total WAL Size = 65/20 = 3,25 MB***
***Таким образом pg_wal 65 МБ, количество контрольных точек 20 во время теста, каждой контрольной точке соответствует приблизительно 3.25 МБ WAL.***

## 4. Проверьте данные статистики: все ли контрольные точки выполнялись точно по расписанию. Почему так произошло?
***Чтобы проверить, выполнялись ли контрольные точки точно по расписанию, использую журналы PostgreSQL:***
***Открываю логи PostgreSQL:***
```
karussia@kntxt-vm:~$ sudo cat /var/log/postgresql/postgresql-15-main.log
2024-04-18 17:07:52.331 UTC [5881] LOG:  starting PostgreSQL 15.6 (Ubuntu 15.6-1.pgdg22.04+1) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 11.4.0-1ubuntu1~22.04) 11.4.0, 64-bit
2024-04-18 17:07:52.331 UTC [5881] LOG:  listening on IPv6 address "::1", port 5432
2024-04-18 17:07:52.331 UTC [5881] LOG:  listening on IPv4 address "127.0.0.1", port 5432
2024-04-18 17:07:52.333 UTC [5881] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"
2024-04-18 17:07:52.341 UTC [5884] LOG:  database system was shut down at 2024-04-18 17:07:48 UTC
2024-04-18 17:07:52.354 UTC [5881] LOG:  database system is ready to accept connections
2024-04-18 17:12:52.441 UTC [5882] LOG:  checkpoint starting: time
2024-04-18 17:12:56.472 UTC [5882] LOG:  checkpoint complete: wrote 43 buffers (0.3%); 0 WAL file(s) added, 0 removed, 0 recycled; write=4.015 s, sync=0.005 s, total=4.032 s; sync files=11, longest=0.003 s, average=0.001 s; distance=252 kB, estimate=252 kB
2024-04-18 17:23:18.050 UTC [5881] LOG:  received fast shutdown request
2024-04-18 17:23:18.054 UTC [5881] LOG:  aborting any active transactions
2024-04-18 17:23:18.057 UTC [5881] LOG:  background worker "logical replication launcher" (PID 5887) exited with exit code 1
2024-04-18 17:23:18.057 UTC [5882] LOG:  shutting down
2024-04-18 17:23:18.059 UTC [5882] LOG:  checkpoint starting: shutdown immediate
2024-04-18 17:23:18.075 UTC [5882] LOG:  checkpoint complete: wrote 0 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.001 s, sync=0.001 s, total=0.019 s; sync files=0, longest=0.000 s, average=0.000 s; distance=0 kB, estimate=227 kB
2024-04-18 17:23:18.079 UTC [5881] LOG:  database system is shut down
2024-04-18 17:23:18.206 GMT [6454] LOG:  invalid value for parameter "checkpoint_timeout": "30sec"
2024-04-18 17:23:18.206 GMT [6454] HINT:  Valid units for this parameter are "us", "ms", "s", "min", "h", and "d".
2024-04-18 17:23:18.206 UTC [6454] FATAL:  configuration file "/etc/postgresql/15/main/postgresql.conf" contains errors
pg_ctl: could not start server
Examine the log output.
2024-04-18 17:31:47.275 GMT [6495] LOG:  invalid value for parameter "checkpoint_timeout": "30sec"
2024-04-18 17:31:47.275 GMT [6495] HINT:  Valid units for this parameter are "us", "ms", "s", "min", "h", and "d".
2024-04-18 17:31:47.276 UTC [6495] FATAL:  configuration file "/etc/postgresql/15/main/postgresql.conf" contains errors
pg_ctl: could not start server
Examine the log output.
2024-04-18 17:37:57.136 GMT [6563] LOG:  invalid value for parameter "checkpoint_timeout": "30sec"
2024-04-18 17:37:57.136 GMT [6563] HINT:  Valid units for this parameter are "us", "ms", "s", "min", "h", and "d".
2024-04-18 17:37:57.136 UTC [6563] FATAL:  configuration file "/etc/postgresql/15/main/postgresql.conf" contains errors
pg_ctl: could not start server
Examine the log output.
2024-04-18 18:13:49.213 UTC [6664] LOG:  starting PostgreSQL 15.6 (Ubuntu 15.6-1.pgdg22.04+1) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 11.4.0-1ubuntu1~22.04) 11.4.0, 64-bit
2024-04-18 18:13:49.213 UTC [6664] LOG:  listening on IPv6 address "::1", port 5432
2024-04-18 18:13:49.213 UTC [6664] LOG:  listening on IPv4 address "127.0.0.1", port 5432
2024-04-18 18:13:49.234 UTC [6664] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"
2024-04-18 18:13:49.245 UTC [6667] LOG:  database system was shut down at 2024-04-18 17:23:18 UTC
2024-04-18 18:13:49.252 UTC [6664] LOG:  database system is ready to accept connections
2024-04-18 18:14:00.995 UTC [6689] karussia@postgres FATAL:  role "karussia" does not exist
2024-04-18 18:14:19.264 UTC [6665] LOG:  checkpoint starting: time
2024-04-18 18:14:19.279 UTC [6665] LOG:  checkpoint complete: wrote 3 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.004 s, sync=0.003 s, total=0.015 s; sync files=2, longest=0.002 s, average=0.002 s; distance=0 kB, estimate=0 kB
2024-04-18 18:15:05.321 UTC [6731] karussia@postgres FATAL:  role "karussia" does not exist
2024-04-18 18:15:49.369 UTC [6665] LOG:  checkpoint starting: time
2024-04-18 18:16:16.050 UTC [6665] LOG:  checkpoint complete: wrote 1707 buffers (10.4%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.667 s, sync=0.003 s, total=26.681 s; sync files=53, longest=0.002 s, average=0.001 s; distance=12824 kB, estimate=12824 kB
2024-04-18 18:17:19.113 UTC [6665] LOG:  checkpoint starting: time
2024-04-18 18:17:46.047 UTC [6665] LOG:  checkpoint complete: wrote 1734 buffers (10.6%); 0 WAL file(s) added, 0 removed, 0 recycled; write=26.872 s, sync=0.032 s, total=26.935 s; sync files=22, longest=0.013 s, average=0.002 s; distance=14466 kB, estimate=14466 kB
2024-04-18 18:17:49.050 UTC [6665] LOG:  checkpoint starting: time
2024-04-18 18:18:16.078 UTC [6665] LOG:  checkpoint complete: wrote 1928 buffers (11.8%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.968 s, sync=0.017 s, total=27.028 s; sync files=10, longest=0.009 s, average=0.002 s; distance=24778 kB, estimate=24778 kB
2024-04-18 18:18:19.081 UTC [6665] LOG:  checkpoint starting: time
2024-04-18 18:18:46.128 UTC [6665] LOG:  checkpoint complete: wrote 2082 buffers (12.7%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.972 s, sync=0.017 s, total=27.048 s; sync files=19, longest=0.006 s, average=0.001 s; distance=24521 kB, estimate=24752 kB
2024-04-18 18:18:49.131 UTC [6665] LOG:  checkpoint starting: time
2024-04-18 18:19:16.027 UTC [6665] LOG:  checkpoint complete: wrote 2019 buffers (12.3%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.853 s, sync=0.014 s, total=26.896 s; sync files=9, longest=0.010 s, average=0.002 s; distance=25242 kB, estimate=25242 kB
2024-04-18 18:19:19.030 UTC [6665] LOG:  checkpoint starting: time
2024-04-18 18:19:46.071 UTC [6665] LOG:  checkpoint complete: wrote 2208 buffers (13.5%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.979 s, sync=0.010 s, total=27.041 s; sync files=19, longest=0.008 s, average=0.001 s; distance=25501 kB, estimate=25501 kB
2024-04-18 18:19:49.074 UTC [6665] LOG:  checkpoint starting: time
2024-04-18 18:20:16.102 UTC [6665] LOG:  checkpoint complete: wrote 2047 buffers (12.5%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.982 s, sync=0.013 s, total=27.029 s; sync files=9, longest=0.007 s, average=0.002 s; distance=25120 kB, estimate=25463 kB
2024-04-18 18:20:19.105 UTC [6665] LOG:  checkpoint starting: time
2024-04-18 18:20:46.123 UTC [6665] LOG:  checkpoint complete: wrote 2171 buffers (13.3%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.961 s, sync=0.031 s, total=27.018 s; sync files=19, longest=0.023 s, average=0.002 s; distance=24288 kB, estimate=25345 kB
2024-04-18 18:20:49.124 UTC [6665] LOG:  checkpoint starting: time
2024-04-18 18:21:16.103 UTC [6665] LOG:  checkpoint complete: wrote 2039 buffers (12.4%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.958 s, sync=0.012 s, total=26.980 s; sync files=9, longest=0.008 s, average=0.002 s; distance=24414 kB, estimate=25252 kB
2024-04-18 18:21:19.106 UTC [6665] LOG:  checkpoint starting: time
2024-04-18 18:21:46.075 UTC [6665] LOG:  checkpoint complete: wrote 2183 buffers (13.3%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.868 s, sync=0.038 s, total=26.969 s; sync files=18, longest=0.033 s, average=0.003 s; distance=24712 kB, estimate=25198 kB
2024-04-18 18:21:49.078 UTC [6665] LOG:  checkpoint starting: time
2024-04-18 18:22:16.016 UTC [6665] LOG:  checkpoint complete: wrote 2037 buffers (12.4%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.889 s, sync=0.010 s, total=26.938 s; sync files=8, longest=0.006 s, average=0.002 s; distance=24394 kB, estimate=25118 kB
2024-04-18 18:22:19.019 UTC [6665] LOG:  checkpoint starting: time
2024-04-18 18:22:46.043 UTC [6665] LOG:  checkpoint complete: wrote 2173 buffers (13.3%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.974 s, sync=0.012 s, total=27.025 s; sync files=18, longest=0.010 s, average=0.001 s; distance=24019 kB, estimate=25008 kB
2024-04-18 18:22:49.046 UTC [6665] LOG:  checkpoint starting: time
2024-04-18 18:23:16.081 UTC [6665] LOG:  checkpoint complete: wrote 2034 buffers (12.4%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.974 s, sync=0.024 s, total=27.036 s; sync files=11, longest=0.020 s, average=0.003 s; distance=24158 kB, estimate=24923 kB
2024-04-18 18:23:19.085 UTC [6665] LOG:  checkpoint starting: time
2024-04-18 18:23:46.124 UTC [6665] LOG:  checkpoint complete: wrote 2163 buffers (13.2%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.973 s, sync=0.014 s, total=27.039 s; sync files=15, longest=0.009 s, average=0.001 s; distance=24114 kB, estimate=24842 kB
2024-04-18 18:23:49.127 UTC [6665] LOG:  checkpoint starting: time
2024-04-18 18:24:16.056 UTC [6665] LOG:  checkpoint complete: wrote 2018 buffers (12.3%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.874 s, sync=0.013 s, total=26.930 s; sync files=9, longest=0.007 s, average=0.002 s; distance=24153 kB, estimate=24773 kB
2024-04-18 18:24:19.059 UTC [6665] LOG:  checkpoint starting: time
2024-04-18 18:24:46.083 UTC [6665] LOG:  checkpoint complete: wrote 2134 buffers (13.0%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.984 s, sync=0.008 s, total=27.024 s; sync files=13, longest=0.004 s, average=0.001 s; distance=23866 kB, estimate=24682 kB
2024-04-18 18:24:49.086 UTC [6665] LOG:  checkpoint starting: time
2024-04-18 18:25:16.143 UTC [6665] LOG:  checkpoint complete: wrote 2000 buffers (12.2%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.967 s, sync=0.032 s, total=27.058 s; sync files=9, longest=0.026 s, average=0.004 s; distance=23890 kB, estimate=24603 kB
2024-04-18 18:25:19.144 UTC [6665] LOG:  checkpoint starting: time
2024-04-18 18:25:46.041 UTC [6665] LOG:  checkpoint complete: wrote 2366 buffers (14.4%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.860 s, sync=0.011 s, total=26.898 s; sync files=16, longest=0.008 s, average=0.001 s; distance=23223 kB, estimate=24465 kB
2024-04-18 18:25:49.044 UTC [6665] LOG:  checkpoint starting: time
2024-04-18 18:26:16.054 UTC [6665] LOG:  checkpoint complete: wrote 1976 buffers (12.1%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.962 s, sync=0.013 s, total=27.010 s; sync files=8, longest=0.012 s, average=0.002 s; distance=23414 kB, estimate=24360 kB
2024-04-18 18:26:19.056 UTC [6665] LOG:  checkpoint starting: time
2024-04-18 18:26:46.086 UTC [6665] LOG:  checkpoint complete: wrote 2108 buffers (12.9%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.972 s, sync=0.010 s, total=27.031 s; sync files=12, longest=0.006 s, average=0.001 s; distance=23954 kB, estimate=24319 kB
2024-04-18 18:26:49.088 UTC [6665] LOG:  checkpoint starting: time
2024-04-18 18:27:16.024 UTC [6665] LOG:  checkpoint complete: wrote 1975 buffers (12.1%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.886 s, sync=0.014 s, total=26.937 s; sync files=9, longest=0.007 s, average=0.002 s; distance=23758 kB, estimate=24263 kB
2024-04-18 18:28:19.088 UTC [6665] LOG:  checkpoint starting: time
2024-04-18 18:28:46.075 UTC [6665] LOG:  checkpoint complete: wrote 2354 buffers (14.4%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.972 s, sync=0.004 s, total=26.988 s; sync files=15, longest=0.004 s, average=0.001 s; distance=22270 kB, estimate=24064 kB
```

***Затем использую pg_stat_bgwriter "psql -U postgres -d postgres -c "SELECT * FROM pg_stat_bgwriter;"***
```
karussia@kntxt-vm:~$ sudo -i -u postgres
postgres@kntxt-vm:~$ psql -U postgres -d postgres -c "SELECT * FROM pg_stat_bgwriter;"

 checkpoints_timed | checkpoints_req | checkpoint_write_time | checkpoint_sync_time | buffers_checkpoint | buffers_clean | maxwritten_clean | buffers_backend | buffers_backend_fsync | buffers_alloc |          stats_reset
-------------------+-----------------+-----------------------+----------------------+--------------------+---------------+------------------+-----------------+-----------------------+---------------+-------------------------------
               103 |               1 |                596387 |                  361 |              45420 |             0 |                0 |            4530 |                     0 |          5309 | 2024-04-18 17:07:48.060813+00
(1 row)
```
***Из записей в логах очевидно, что контрольные точки происходят регулярно, как и было запланировано. PostgreSQL регистрирует каждое начало и завершение контрольной точки, есть несколько таких записей в логах. Это показывает, что контрольные точки действительно инициируются примерно каждые 30 секунд, как было настроено.

***Записи в pg_stat_bgwriter также предоставляют полезные данные:***
- checkpoints_timed: С момента последнего сброса статистики произошло 103 таймированных контрольных точек, что предполагает, что система часто достигает запланированного времени контрольной точки, чаще, чем минимально установленное время.***
- checkpoint_write_time и checkpoint_sync_time: Эти значения (в миллисекундах) указывают, сколько времени занимает процесс контрольной точки. Время записи значительно, что может быть областью для настройки производительности, если это становится узким местом.
- buffers_checkpoint: Показывает количество буферов, записанных во время контрольных точек, помогая оценить объем данных, обрабатываемых во время этих событий.
***Промежуточные выводы:
- Если контрольные точки происходят чаще, чем каждые 30 секунд (как может быть намекнуто checkpoints_timed выше ожидаемого за данный период), это может быть связано с другими настройками, которые заставляют контрольные точки происходить чаще, или высоким уровнем активности базы данных, вынуждающим сброс буферов на диск.
- Регулярные и эффективные контрольные точки критичны для хорошей производительности базы данных и времени восстановления. Если контрольные точки слишком частые или включают слишком много данных, они могут замедлить нормальную операцию базы данных. Напротив, редкие контрольные точки могут привести к более длительному времени восстановления после сбоя, потому что необходимо воспроизвести больше WAL.

## 5. Сравните tps в синхронном/асинхронном режиме утилитой pgbench. Объясните полученный результат.
***По умолчанию работает синхронный режим (в файле /etc/postgresql/15/main/postgresql.conf строка "#synchronous_commit = on  # synchronization level;" является закомментированной, при этом по умолчанию  PostgreSQL использует режим "synchronous_commit = on"), поэтому запускаю его:***
```
karussia@kntxt-vm:~$ sudo -i -u postgres
postgres@kntxt-vm:~$ pgbench -c 10 -T 600 -P 10 postgres
pgbench (15.6 (Ubuntu 15.6-1.pgdg22.04+1))
starting vacuum...end.
progress: 10.0 s, 323.1 tps, lat 30.870 ms stddev 37.715, 0 failed
progress: 20.0 s, 498.4 tps, lat 20.002 ms stddev 19.143, 0 failed
progress: 30.0 s, 307.1 tps, lat 32.624 ms stddev 31.610, 0 failed
progress: 40.0 s, 249.9 tps, lat 39.967 ms stddev 58.034, 0 failed
progress: 50.0 s, 511.5 tps, lat 19.560 ms stddev 16.870, 0 failed
progress: 60.0 s, 468.1 tps, lat 21.366 ms stddev 18.217, 0 failed
progress: 70.0 s, 373.1 tps, lat 26.827 ms stddev 30.541, 0 failed
progress: 80.0 s, 436.8 tps, lat 22.852 ms stddev 25.865, 0 failed
progress: 90.0 s, 432.8 tps, lat 23.163 ms stddev 23.364, 0 failed
progress: 100.0 s, 309.9 tps, lat 32.224 ms stddev 49.376, 0 failed
progress: 110.0 s, 454.8 tps, lat 21.990 ms stddev 22.698, 0 failed
progress: 120.0 s, 610.5 tps, lat 16.392 ms stddev 13.161, 0 failed
progress: 130.0 s, 285.7 tps, lat 34.933 ms stddev 49.742, 0 failed
progress: 140.0 s, 516.4 tps, lat 19.328 ms stddev 16.786, 0 failed
progress: 150.0 s, 314.4 tps, lat 31.892 ms stddev 31.104, 0 failed
progress: 160.0 s, 259.4 tps, lat 38.569 ms stddev 57.182, 0 failed
progress: 170.0 s, 409.4 tps, lat 24.372 ms stddev 24.060, 0 failed
progress: 180.0 s, 264.9 tps, lat 37.846 ms stddev 38.323, 0 failed
progress: 190.0 s, 312.7 tps, lat 31.992 ms stddev 48.958, 0 failed
progress: 200.0 s, 469.4 tps, lat 21.221 ms stddev 22.879, 0 failed
progress: 210.0 s, 513.6 tps, lat 19.510 ms stddev 19.404, 0 failed
progress: 220.0 s, 311.0 tps, lat 32.200 ms stddev 46.665, 0 failed
progress: 230.0 s, 485.0 tps, lat 20.589 ms stddev 19.413, 0 failed
progress: 240.0 s, 421.8 tps, lat 23.734 ms stddev 21.013, 0 failed
progress: 250.0 s, 497.7 tps, lat 20.102 ms stddev 25.524, 0 failed
progress: 260.0 s, 520.9 tps, lat 19.161 ms stddev 20.400, 0 failed
progress: 270.0 s, 395.7 tps, lat 25.322 ms stddev 22.637, 0 failed
progress: 280.0 s, 297.8 tps, lat 33.546 ms stddev 52.303, 0 failed
progress: 290.0 s, 515.7 tps, lat 19.380 ms stddev 19.642, 0 failed
progress: 300.0 s, 485.7 tps, lat 20.592 ms stddev 19.079, 0 failed
progress: 310.0 s, 229.4 tps, lat 43.560 ms stddev 51.165, 0 failed
progress: 320.0 s, 444.2 tps, lat 22.525 ms stddev 22.984, 0 failed
progress: 330.0 s, 305.7 tps, lat 32.712 ms stddev 33.764, 0 failed
progress: 340.0 s, 228.8 tps, lat 43.652 ms stddev 54.419, 0 failed
progress: 350.0 s, 441.4 tps, lat 22.701 ms stddev 19.998, 0 failed
progress: 360.0 s, 325.7 tps, lat 30.674 ms stddev 27.699, 0 failed
progress: 370.0 s, 432.8 tps, lat 23.135 ms stddev 33.024, 0 failed
progress: 380.0 s, 605.0 tps, lat 16.515 ms stddev 15.891, 0 failed
progress: 390.0 s, 666.9 tps, lat 15.005 ms stddev 11.627, 0 failed
progress: 400.0 s, 310.7 tps, lat 32.158 ms stddev 44.970, 0 failed
progress: 410.0 s, 487.4 tps, lat 20.497 ms stddev 18.558, 0 failed
progress: 420.0 s, 368.3 tps, lat 27.197 ms stddev 27.398, 0 failed
progress: 430.0 s, 473.5 tps, lat 21.126 ms stddev 24.138, 0 failed
progress: 440.0 s, 486.4 tps, lat 20.509 ms stddev 23.879, 0 failed
progress: 450.0 s, 337.1 tps, lat 29.685 ms stddev 29.791, 0 failed
progress: 460.0 s, 332.4 tps, lat 30.132 ms stddev 48.594, 0 failed
progress: 470.0 s, 600.7 tps, lat 16.584 ms stddev 14.039, 0 failed
progress: 480.0 s, 489.3 tps, lat 20.504 ms stddev 18.203, 0 failed
progress: 490.0 s, 329.9 tps, lat 30.272 ms stddev 42.475, 0 failed
progress: 500.0 s, 518.8 tps, lat 19.264 ms stddev 19.973, 0 failed
progress: 510.0 s, 471.3 tps, lat 21.251 ms stddev 19.224, 0 failed
progress: 520.0 s, 221.3 tps, lat 45.166 ms stddev 57.464, 0 failed
progress: 530.0 s, 597.7 tps, lat 16.701 ms stddev 15.171, 0 failed
progress: 540.0 s, 399.0 tps, lat 25.061 ms stddev 22.423, 0 failed
progress: 550.0 s, 271.1 tps, lat 36.995 ms stddev 56.321, 0 failed
progress: 560.0 s, 506.6 tps, lat 19.692 ms stddev 18.287, 0 failed
progress: 570.0 s, 588.0 tps, lat 17.039 ms stddev 16.488, 0 failed
progress: 580.0 s, 383.5 tps, lat 26.046 ms stddev 45.453, 0 failed
progress: 590.0 s, 500.5 tps, lat 19.960 ms stddev 18.598, 0 failed
progress: 600.0 s, 464.3 tps, lat 21.585 ms stddev 19.508, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 250719
number of failed transactions: 0 (0.000%)
latency average = 23.930 ms
latency stddev = 29.874 ms
initial connection time = 19.143 ms
tps = 417.868278 (without initial connection time)
```
***Для настройки асинхронного режима открою файл /etc/postgresql/15/main/postgresql.conf где сделаю запись
```
synchronous_commit = off
```
***и сделаю рестарт:***
```
karussia@kntxt-vm:~$ sudo systemctl restart postgresql
```
***теперь запускю ту же команду "pgbench -c 10 -T 60 -P 10 postgres", но теперь она будет использовать асинхронный режим:***
```
karussia@kntxt-vm:~$ sudo -i -u postgres
postgres@kntxt-vm:~$ pgbench -c 10 -T 600 -P 10 postgres
pgbench (15.6 (Ubuntu 15.6-1.pgdg22.04+1))
starting vacuum...end.
progress: 10.0 s, 5026.3 tps, lat 1.984 ms stddev 1.086, 0 failed
progress: 20.0 s, 5046.0 tps, lat 1.980 ms stddev 1.079, 0 failed
progress: 30.0 s, 4963.2 tps, lat 2.013 ms stddev 1.104, 0 failed
progress: 40.0 s, 4968.3 tps, lat 2.011 ms stddev 1.132, 0 failed
progress: 50.0 s, 4994.4 tps, lat 2.000 ms stddev 1.096, 0 failed
progress: 60.0 s, 4967.7 tps, lat 2.011 ms stddev 1.124, 0 failed
progress: 70.0 s, 4977.5 tps, lat 2.007 ms stddev 1.118, 0 failed
progress: 80.0 s, 5027.6 tps, lat 1.987 ms stddev 1.096, 0 failed
progress: 90.0 s, 5019.5 tps, lat 1.990 ms stddev 1.127, 0 failed
progress: 100.0 s, 4963.4 tps, lat 2.013 ms stddev 1.170, 0 failed
progress: 110.0 s, 5042.5 tps, lat 1.981 ms stddev 1.099, 0 failed
progress: 120.0 s, 4995.7 tps, lat 2.000 ms stddev 1.123, 0 failed
progress: 130.0 s, 4993.5 tps, lat 2.001 ms stddev 1.122, 0 failed
progress: 140.0 s, 4969.4 tps, lat 2.011 ms stddev 1.133, 0 failed
progress: 150.0 s, 4985.7 tps, lat 2.004 ms stddev 1.119, 0 failed
progress: 160.0 s, 4975.7 tps, lat 2.008 ms stddev 1.130, 0 failed
progress: 170.0 s, 5016.7 tps, lat 1.992 ms stddev 1.114, 0 failed
progress: 180.0 s, 5037.3 tps, lat 1.983 ms stddev 1.104, 0 failed
progress: 190.0 s, 5028.8 tps, lat 1.987 ms stddev 1.160, 0 failed
progress: 200.0 s, 4986.8 tps, lat 2.004 ms stddev 1.120, 0 failed
progress: 210.0 s, 4969.4 tps, lat 2.011 ms stddev 1.130, 0 failed
progress: 220.0 s, 4880.7 tps, lat 2.047 ms stddev 3.468, 0 failed
progress: 230.0 s, 4802.4 tps, lat 2.080 ms stddev 8.703, 0 failed
progress: 240.0 s, 5084.6 tps, lat 1.965 ms stddev 1.104, 0 failed
progress: 250.0 s, 5030.2 tps, lat 1.986 ms stddev 1.120, 0 failed
progress: 260.0 s, 5061.5 tps, lat 1.974 ms stddev 1.094, 0 failed
progress: 270.0 s, 5039.0 tps, lat 1.983 ms stddev 1.087, 0 failed
progress: 280.0 s, 4922.1 tps, lat 2.030 ms stddev 1.185, 0 failed
progress: 290.0 s, 5019.3 tps, lat 1.991 ms stddev 1.127, 0 failed
progress: 300.0 s, 5019.1 tps, lat 1.990 ms stddev 1.142, 0 failed
progress: 310.0 s, 4996.2 tps, lat 2.000 ms stddev 1.111, 0 failed
progress: 320.0 s, 5078.7 tps, lat 1.967 ms stddev 1.098, 0 failed
progress: 330.0 s, 5030.4 tps, lat 1.986 ms stddev 1.094, 0 failed
progress: 340.0 s, 4989.9 tps, lat 2.002 ms stddev 1.162, 0 failed
progress: 350.0 s, 5003.5 tps, lat 1.997 ms stddev 1.121, 0 failed
progress: 360.0 s, 4993.3 tps, lat 2.001 ms stddev 1.130, 0 failed
progress: 370.0 s, 4945.8 tps, lat 2.020 ms stddev 1.128, 0 failed
progress: 380.0 s, 4951.5 tps, lat 2.018 ms stddev 1.131, 0 failed
progress: 390.0 s, 4998.0 tps, lat 1.999 ms stddev 1.106, 0 failed
progress: 400.0 s, 5003.6 tps, lat 1.997 ms stddev 1.155, 0 failed
progress: 410.0 s, 5021.9 tps, lat 1.990 ms stddev 1.099, 0 failed
progress: 420.0 s, 4947.2 tps, lat 2.020 ms stddev 1.117, 0 failed
progress: 430.0 s, 4932.9 tps, lat 2.025 ms stddev 1.122, 0 failed
progress: 440.0 s, 4918.0 tps, lat 2.032 ms stddev 1.115, 0 failed
progress: 450.0 s, 4929.7 tps, lat 2.027 ms stddev 1.136, 0 failed
progress: 460.0 s, 4898.0 tps, lat 2.040 ms stddev 1.177, 0 failed
progress: 470.0 s, 4928.2 tps, lat 2.027 ms stddev 1.114, 0 failed
progress: 480.0 s, 4897.6 tps, lat 2.040 ms stddev 1.113, 0 failed
progress: 490.0 s, 4888.9 tps, lat 2.044 ms stddev 1.126, 0 failed
progress: 500.0 s, 4901.4 tps, lat 2.038 ms stddev 1.134, 0 failed
progress: 510.0 s, 4923.6 tps, lat 2.029 ms stddev 1.113, 0 failed
progress: 520.0 s, 4878.6 tps, lat 2.048 ms stddev 1.178, 0 failed
progress: 530.0 s, 4950.4 tps, lat 2.018 ms stddev 1.102, 0 failed
progress: 540.0 s, 4960.8 tps, lat 2.014 ms stddev 1.101, 0 failed
progress: 550.0 s, 4856.6 tps, lat 2.057 ms stddev 1.137, 0 failed
progress: 560.0 s, 4922.0 tps, lat 2.030 ms stddev 1.125, 0 failed
progress: 570.0 s, 4925.8 tps, lat 2.028 ms stddev 1.114, 0 failed
progress: 580.0 s, 4811.6 tps, lat 2.077 ms stddev 1.213, 0 failed
progress: 590.0 s, 4937.4 tps, lat 2.024 ms stddev 1.124, 0 failed
progress: 600.0 s, 4913.4 tps, lat 2.034 ms stddev 1.135, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 2981502
number of failed transactions: 0 (0.000%)
latency average = 2.011 ms
latency stddev = 1.625 ms
initial connection time = 19.439 ms
tps = 4969.205894 (without initial connection time)
```
***Сравнивая результаты двух тестов pgbench в синхронном и асинхронном режимах, сделаю ряд выводов о том, как режим подтверждения транзакций влияет на производительность системы:***
***1. Синхронный режим:***
- TPS (транзакций в секунду): 417.868278
- Средняя задержка: 23.930 мс
- Стандартное отклонение задержки: 29.874 мс
***2. Асинхронный режим:***
- TPS (транзакций в секунду): 4969.205894
- Средняя задержка: 2.011 мс
- Стандартное отклонение задержки: 1.625 мс
***3. Анализ результатов:***
- Производительность TPS: В асинхронном режиме TPS значительно выше (4969.20 против 417.87), так как в асинхронном режиме PostgreSQL не ждет подтверждения записи данных на диск перед тем, как сообщить клиенту об успешном завершении транзакции, что позволяет обрабатывать больше транзакций за тот же период времени.
- Задержка: Средняя задержка в асинхронном режиме значительно ниже, чем в синхронном (2.011 мс против 23.930 мс) - ожидание записи данных на диск в синхронном режиме увеличивает общее время ответа системы на операции.
- Стандартное отклонение задержки: В асинхронном режиме стандартное отклонение задержки также ниже, что указывает на более стабильное и предсказуемое время ответа на операции по сравнению с синхронным режимом.
***4. Выводы:***
- Влияние на производительность: Асинхронный режим явно выигрывает по производительности за счет уменьшения задержек и увеличения количества обработанных транзакций. Скорее всего асинхронный режим будет более предпочтительным выбором для проектов, где требования к производительности превышают требования к надежности данных.
- Риски асинхронного режима: потенциально при асинхронном режиме увеличивается риск потери данных в случае сбоя системы, например, транзакции могут быть отмечены как завершенные до того как данные фактически записаны на диск, хотя по разным причинам фактически данные могут и не записаться на диск.

## 6. Создайте новый кластер с включенной контрольной суммой страниц. Создайте таблицу. Вставьте несколько значений. Выключите кластер. Измените пару байт в таблице. Включите кластер и сделайте выборку из таблицы. Что и почему произошло? как проигнорировать ошибку и продолжить работу?
***Удалю существующий кластер и создам новый:***
```
karussia@kntxt-vm:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
karussia@kntxt-vm:~$ sudo pg_dropcluster 15 main --stop
karussia@kntxt-vm:~$ pg_lsclusters
Ver Cluster Port Status Owner Data directory Log file
karussia@kntxt-vm:~$ sudo pg_createcluster 15 main --start --data-checksums
Unknown option: data-checksums
karussia@kntxt-vm:~$ sudo pg_createcluster 15 main --start -- --data-checksums
Creating new PostgreSQL cluster 15/main ...
/usr/lib/postgresql/15/bin/initdb -D /var/lib/postgresql/15/main --auth-local peer --auth-host scram-sha-256 --no-instructions --data-checksums
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locale "C.UTF-8".
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are enabled.

fixing permissions on existing directory /var/lib/postgresql/15/main ... ok
creating subdirectories ... ok
selecting dynamic shared memory implementation ... posix
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting default time zone ... Etc/UTC
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
karussia@kntxt-vm:~$ sudo systemctl restart postgresql
```
***Затем я использую команду "sudo hexedit /var/lib/postgresql/15/main/base/database_oid/relfilenode" чтобы открыть в кодированом виде мои записи в таблице и поменять в них что-нибудь***
Сначала найду значение database_oid:
```
karussia@kntxt-vm:~$ sudo systemctl start postgresql
karussia@kntxt-vm:~$ sudo -i -u postgres
postgres@kntxt-vm:~$ psql
psql (15.6 (Ubuntu 15.6-1.pgdg22.04+1))
Type "help" for help.

postgres=# SELECT oid, datname FROM pg_database WHERE datname = 'usagov';
  oid  | datname
-------+---------
 16388 | usagov
(1 row)
```
и значение relfilenode:
```
postgres=# \c usagov
You are now connected to database "usagov" as user "postgres".
usagov=# SELECT relfilenode, relname FROM pg_class WHERE relname = 'senators';
 relfilenode | relname
-------------+----------
       16390 | senators
(1 row)
```
Теперь запускаю команду "sudo hexedit /var/lib/postgresql/15/main/base/16388/16390" (предварительно установив hexedit):
```
karussia@kntxt-vm:~$ sudo apt update && sudo apt install hexedit -y
karussia@kntxt-vm:~$ sudo hexedit /var/lib/postgresql/15/main/base/16388/16390
00000000   00 00 00 00  E8 0B BD 01  50 94 00 00  24 00 40 1F  00 20 04 20  00 00 00 00  C0 9F 7E 00  80 9F 80 00  40 9F 78 00  00 00 00 00  00 00 00 00  ........P...$.@.. . ......~.....@.x.........
...
00001F48   00 00 00 00  00 00 00 00  03 00 04 00  02 09 18 00  03 00 00 00  1D 43 68 75  63 6B 20 53  63 68 75 6D  65 72 13 4E  65 77 20 59  6F 72 6B 13  .....................Chuck Schumer.New York.
00001F74   44 65 6D 6F  63 72 61 74  00 00 00 00  E1 02 00 00  00 00 00 00  00 00 00 00  00 00 00 00  02 00 04 00  02 09 18 00  02 00 00 00  21 4D 69 74  Democrat................................!Mit
00001FA0   63 68 20 4D  63 43 6F 6E  6E 65 6C 6C  13 4B 65 6E  74 75 63 6B  79 17 52 65  70 75 62 6C  69 63 61 6E  E1 02 00 00  00 00 00 00  00 00 00 00  ch McConnell.Kentucky.Republican............
00001FCC   00 00 00 00  01 00 04 00  02 09 18 00  01 00 00 00  1F 42 65 72  6E 69 65 20  53 61 6E 64  65 72 73 11  56 65 72 6D  6F 6E 74 19  49 6E 64 65  .................Bernie Sanders.Vermont.Inde
00001FF8   70 65 6E 64  65 6E 74 00                                                                                                                       pendent.
```
сделаю правку в этом коде изменив 1D на 1E в строке 00001F48, сохраню это и перезагружу postgresql
```
karussia@kntxt-vm:~$ sudo systemctl restart postgresql
karussia@kntxt-vm:~$ sudo -i -u postgres
postgres@kntxt-vm:~$ psql -d usagov -c "SELECT * FROM senators;"
WARNING:  page verification failed, calculated checksum 47653 but expected 37968
ERROR:  invalid page in block 0 of relation base/16388/16390
```
***Моя база данных обнаружила повреждение данных из-за изменения, которое я внес непосредственно в файл таблицы.***
Сообщение `WARNING: page verification failed, calculated checksum 47653 but expected 37968` указывает на то, что контрольная сумма страницы не соответствует ожидаемой, что является признаком повреждения данных.
Сообщение `ERROR: invalid page in block 0 of relation base/16388/16390` указывает на то, что первая страница файла данных таблицы senators не может быть прочитана из-за ошибки контрольной суммы.

***Тем не менее можно скорректировать значение ignore_checksum_failure в файле /etc/postgresql/15/main/postgresql.conf сделав его***
```
ignore_checksum_failure = on
```
***Это позволит серверу игнорировать ошибки контрольных сумм при чтении данных.***
***Перезагружаю сервер PostgreSQL и проверяю после правки ignore_checksum_failure:***
```
karussia@kntxt-vm:~$ sudo systemctl restart postgresql
karussia@kntxt-vm:~$ sudo -i -u postgres
postgres@kntxt-vm:~$ psql -d usagov -c "SELECT * FROM senators;"
WARNING:  page verification failed, calculated checksum 47653 but expected 37968
server closed the connection unexpectedly
  This probably means the server terminated abnormally
  before or while processing the request.
connection to server was lost
postgres@kntxt-vm:~$ psql
psql (15.6 (Ubuntu 15.6-1.pgdg22.04+1))
Type "help" for help.

postgres=# \c usagov
You are now connected to database "usagov" as user "postgres".
usagov=# SELECT * FROM senators;
WARNING:  page verification failed, calculated checksum 47653 but expected 37968
server closed the connection unexpectedly
  This probably means the server terminated abnormally
  before or while processing the request.
The connection to the server was lost. Attempting reset: Failed.
The connection to the server was lost. Attempting reset: Failed.
!?>
```
***В данном случае вижу, что изменение данных в hex редакторе привело к серьезному повреждению файлов данных, из-за которого сервер PostgreSQL не может корректно обработать запросы. Ошибка контрольной суммы обычно указывает на нарушение целостности данных, и, несмотря на включение параметра ignore_checksum_failure, сервер все равно завершает работу, потому что ошибка может быть настолько серьезной, что предотвращает дальнейшее выполнение.***
В таких случаях имеет смысл восстановить данных из резервной копии или пересоздать таблицу, если это возможно.
***Теперь же верну все на свои места и проверю если сделаю правку в другом месте, поменяв в первой строке 1F на 1E:***
```
00000000   00 00 00 00  E8 0B BD 01  50 94 00 00  24 00 40 1F  00 20 04 20  00 00 00 00  C0 9F 7E 00  80 9F 80 00  40 9F 78 00  00 00 00 00  ........P...$.@.. . ......~.....@.x.....
```
Делаю проверку:
```
karussia@kntxt-vm:~$ sudo systemctl restart postgresql
postgres@kntxt-vm:~$ psql -d usagov -c "SELECT * FROM senators;"
WARNING:  page verification failed, calculated checksum 25499 but expected 37968
 id |     senator     |  state   |    party
----+-----------------+----------+-------------
  1 | Bernie Sanders  | Vermont  | Independent
  2 | Mitch McConnell | Kentucky | Republican
  3 | Chuck Schumer   | New York | Democrat
(3 rows)
```
Теперь же вижу все успешно, получилось увидеть записи в таблице проигнорирова ошибку контрольной суммы, то есть данная ошибка была допустима для продолжения работы.
