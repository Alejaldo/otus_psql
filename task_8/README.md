# Задание №8 "Нагрузочное тестирование и тюнинг PostgreSQL"

## 1. Развернуть виртуальную машину любым удобным способом + 2. поставить на неё PostgreSQL 15 любым способом
***Буду использовать виртуальную машину, ранее созданную в Yandex Cloud, со следующими характеристиками:***
```
OS: Ubuntu 22.04
vCPU: 4
RAM: 8 GB
Disk space: 20 GB
```
где уже есть установленный ранее PostgreSQL 15, проверяю на всякий случай его статус:
```
karussia@kntxt-vm:~$ sudo systemctl status postgresql
● postgresql.service - PostgreSQL RDBMS
     Loaded: loaded (/lib/systemd/system/postgresql.service; enabled; vendor pr>
     Active: active (exited) since Fri 2024-05-10 07:40:00 UTC; 23h ago
    Process: 6158 ExecStart=/bin/true (code=exited, status=0/SUCCESS)
   Main PID: 6158 (code=exited, status=0/SUCCESS)
        CPU: 1ms

May 10 07:40:00 kntxt-vm systemd[1]: Starting PostgreSQL RDBMS...
May 10 07:40:00 kntxt-vm systemd[1]: Finished PostgreSQL RDBMS.
```

# Шаги с 3 по 6 буду тестировать сценарно с разным набором характеристик, которые условно обозначу так: Minimal, High Concurrency и Aggressive Performance.
Aggressive Performance - максимизация пропускной способности и скорости, за счет более высокого риска и потребления ресурсов на соединение, High Concurrency - баланс пропускной способность и возможности надежно обслуживать множество пользователей.\
В каждом из сценариев буду использовать один и тот же набор параметров значения которых будут корректировать в файле /etc/postgresql/15/main/postgresql.conf:

- Настройки памяти:
    - shared_buffers: Увеличение значения должно улучшить производительность за счет использования большего количества памяти для кэширования данных, согласно документации рекоммендуемое возможное значение должно быть до примерно 25% от RAM.
    - work_mem: Используется для операций запроса, таких как сортировки и соединения, слишком большое значение может вызвать проблемы, если много соединений используют много памяти, начальной точкой выберу 64 МБ.
    - maintenance_work_mem: Увеличение значения этого параметра также должно улучшить производительности во время обслуживания (например, при VACUUM) до примерно 512 МБ.

- Производительность записи:
    - synchronous_commit: Значение off дорлжно улучшить производительности записи.
    - checkpoint_timeout и max_wal_size: Увеличение этих параметров должно уменьшить частоту синхронизации диска.

- Соединение:
    - max_connections: Каждое соединение требует памяти и других ресурсов.


Итак,
- параметры для Minimal:
```
shared_buffers = 2GB           # Этот параметр определяет объем памяти, выделенный для кэширования блоков данных в RAM. Установка его в 2GB (около 25% от общего объема RAM 8GB) балансирует потребность в значительном размере кэша с оставлением достаточного объема памяти для операционной системы и других процессов PostgreSQL. Это помогает уменьшить дисковый ввод-вывод, сохраняя часто используемые данные в памяти.
work_mem = 64MB                # Этот параметр контролирует объем памяти, используемый для внутренних операций сортировки, таких как ORDER BY или операции с индексами. Установка его в 64MB направлена на предоставление достаточного объема памяти для обработки умеренных нагрузок без необходимости записи PostgreSQL во временные дисковые файлы, тем самым ускоряя выполнение запросов.
maintenance_work_mem = 512MB   # Этот параметр указывает максимальный объем памяти для операций обслуживания, таких как VACUUM, CREATE INDEX и ALTER TABLE ADD FOREIGN KEY. Он установлен выше, чем work_mem, чтобы ускорить рутинное обслуживание базы данных, которое может быть ресурсоемким, но необходимым для долгосрочной производительности и стабильности.
synchronous_commit = off       # Отключение синхронного коммита позволяет завершать транзакции до того, как их изменения будут записаны на диск, что улучшает производительность записи. Эта настройка особенно полезна при интенсивных операциях записи, поскольку она снижает задержку коммита, но с риском потери данных в случае внезапного сбоя.
checkpoint_timeout = 10min     # Этот параметр определяет максимальное время между автоматическими контрольными точками WAL. Его увеличение до 10 минут снижает частоту дисковых записей, связанных с контрольными точками, что может улучшить общую производительность за счет снижения нагрузки на ввод-вывод, хотя и с немного увеличенным риском потери данных в случае сбоя системы.
max_wal_size = 2GB             # Этот параметр устанавливает максимальный размер файлов WAL между автоматическими контрольными точками. Больший размер WAL позволяет проводить больше транзакций между контрольными точками, снижая необходимость в частом контрольном точечном управлении и тем самым уменьшая операции ввода-вывода, что может повысить производительность в периоды высокой рабочей нагрузки.
max_connections = 100          # Это максимальное количество одновременных соединений, которое может обработать PostgreSQL. Установил в 100, чтобы позволить значительное количество одновременных клиентских соединений, обеспечивая хороший баланс между использованием системных ресурсов и способностью обрабатывать множество пользователей или приложений, подключающихся к базе данных.
#effective_io_concurrency = 1  # Используется для установки количества одновременных дисковых операций ввода-вывода, которые PostgreSQL ожидает, что можно будет выполнять одновременно. В сценарии Minimal использую дефолтное значение
```
- параметры для High Concurrency
```
shared_buffers = 4GB           # Сбалансированный подход к распределению памяти для общих буферов.
work_mem = 128MB               # Умеренно высокий для обработки большего числа запросов, выполняющих сортировки и разные JOIN-ы.
maintenance_work_mem = 2GB     # Позволяет проводить агрессивные операции по обслуживанию, что важно для загруженной базы данных.
synchronous_commit = on        # Поддерживает целостность данных за счет некоторой задержки записи.
checkpoint_timeout = 5min      # Более частые контрольные точки для лучшей надежности и более короткого восстановления.
max_wal_size = 1GB             # Меньшие сегменты WAL для управления дисковым пространством и обеспечения частых очисток.
max_connections = 300          # Высокое количество соединений для поддержки одновременных пользователей.
effective_io_concurrency = 100 # Предполагает умеренные возможности ввода-вывода, подходящие для среднего оборудования.
```
- параметры для Aggressive Performance
```
shared_buffers = 6GB           # Увеличено для использования большего объема RAM для кэширования, улучшая производительность чтения.
work_mem = 256MB               # Позволяет использовать больше памяти для сортировки и хэш-таблиц, полезно для сложных запросов.
maintenance_work_mem = 1GB     # Больше памяти для задач по обслуживанию может ускорить операции, такие как создание индексов.
synchronous_commit = off       # Улучшает производительность записи, не ожидая подтверждения записей WAL.
checkpoint_timeout = 30min     # Менее частые контрольные точки могут улучшить производительность, но увеличивают риск потери данных при сбое.
max_wal_size = 4GB             # Позволяет записывать больше данных до принудительной контрольной точки, сокращая дисковые записи.
max_connections = 200          # Увеличено для разрешения большего числа одновременных соединений.
effective_io_concurrency = 200 # Более высокая настройка для лучшего использования SSD или нескольких дисков.
```
После правок в файле каждый раз буду делать "sudo systemctl restart postgresql"

# Сценарий Minimal

## 3. Настроить кластер PostgreSQL 15 на максимальную производительность не обращая внимание на возможные проблемы с надежностью в случае аварийной перезагрузки виртуальной машины
Применяю обозначенные выше значения параметров и перезагружаю:
```
karussia@kntxt-vm:~$ sudo nano /etc/postgresql/15/main/postgresql.conf
karussia@kntxt-vm:~$ sudo systemctl restart postgresql
```
## 4. нагрузить кластер через утилиту через утилиту pgbench (https://postgrespro.ru/docs/postgrespro/14/pgbench) + 5. написать какого значения tps удалось достичь, показать какие параметры в какие значения устанавливали и почему
Буду использовать "pgbench -i -s 50 postgres" для инициализации и затем запущу с конфигурацией 10 Клиентов (количество одновременных клиентов базы данных (или соединений), которые будет симулировать pgbench), 2 Потока (сколько потоков использует pgbench для управления клиентами), 600 Секунд - "pgbench -c 10 -j 2 -T 600 postgres"
```
karussia@kntxt-vm:~$ sudo -i -u postgres
postgres@kntxt-vm:~$ pgbench -i -s 50 postgres
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
5000000 of 5000000 tuples (100%) done (elapsed 110.44 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 130.53 s (drop tables 0.00 s, create tables 0.00 s, client-side generate 111.14 s, vacuum 0.13 s, primary keys 19.25 s).
postgres@kntxt-vm:~$ pgbench -c 10 -j 2 -T 600 postgres
pgbench (15.7 (Ubuntu 15.7-1.pgdg22.04+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 50
query mode: simple
number of clients: 10
number of threads: 2
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 3131130
number of failed transactions: 0 (0.000%)
latency average = 1.916 ms
initial connection time = 11.172 ms
tps = 5218.492266 (without initial connection time)
```
Результат содержит следующие параметры:
- "transaction type": TPC-B - это бенчмарк для оценки производительности баз данных с точки зрения количества транзакций, которые система может обработать за определенное время, pgbench использует упрощенную версию этого.
- "scaling factor": 50 - размер базы данных, используемой в тесте ("pgbench -i -s 50 postgres").
- "query mode": simple - то есть запросы отправляются на сервер без какой-либо дополнительной разборки или подготовки, что может влиять на скорость и накладные расходы обработки.
- "number of clients", "number of threads" и "duration" - это параметры ранее указанные в команде "pgbench -c 10 -j 2 -T 600 postgres"
- "number of transactions actually processed": 3131130 - общее количество транзакций, завершенных за период тестирования в 600 секунд.
- "number of failed transactions": 0 - указывает на высокую надежность в течение тестового периода при данной нагрузке.
- "latency average": 1.916 мс. - среднее время ответа на транзакцию, показывающее, насколько быстро система реагирует на запросы.
- "initial connection time": 11.172 мс. - время, необходимое для установления первоначальных соединений с базой данных, которое обычно выше, чем при последующих попытках соединения.
- "tps" (Транзакции в Секунду): 5218.492266. - сколько транзакций PostgreSQL может обработать в среднем за секунду в течение тестового периода (не включается время первоначальной настройки соединения, только в части операционной пропускной способности).

## 6. аналогично протестировать через утилиту https://github.com/Percona-Lab/sysbench-tpcc (требует установки https://github.com/akopytov/sysbench)
Согласно инструкции в репозитории https://github.com/akopytov/sysbench делаю установку:
```
karussia@kntxt-vm:~$ curl -s https://packagecloud.io/install/repositories/akopytov/sysbench/script.deb.sh | sudo bash
Detected operating system as Ubuntu/jammy.
Checking for curl...
Detected curl...
Checking for gpg...
Detected gpg...
Detected apt version as 2.4.12
Running apt-get update... done.
Installing apt-transport-https... done.
Installing /etc/apt/sources.list.d/akopytov_sysbench.list...done.
Importing packagecloud gpg key... Packagecloud gpg key imported to /etc/apt/keyrings/akopytov_sysbench-archive-keyring.gpg
done.
Running apt-get update... done.

The repository is setup! You can now install packages.
karussia@kntxt-vm:~$ sudo apt -y install sysbench
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following additional packages will be installed:
  libluajit-5.1-2 libluajit-5.1-common libmysqlclient21 mysql-common
The following NEW packages will be installed:
  libluajit-5.1-2 libluajit-5.1-common libmysqlclient21 mysql-common sysbench
0 upgraded, 5 newly installed, 0 to remove and 6 not upgraded.
Need to get 1711 kB of archives.
After this operation, 8056 kB of additional disk space will be used.
Get:1 http://mirror.yandex.ru/ubuntu jammy/universe amd64 libluajit-5.1-common all 2.1.0~beta3+dfsg-6 [44.3 kB]
Get:2 http://mirror.yandex.ru/ubuntu jammy/universe amd64 libluajit-5.1-2 amd64 2.1.0~beta3+dfsg-6 [238 kB]
Get:3 http://mirror.yandex.ru/ubuntu jammy/main amd64 mysql-common all 5.8+1.0.8 [7212 B]
Get:4 http://mirror.yandex.ru/ubuntu jammy-updates/main amd64 libmysqlclient21 amd64 8.0.36-0ubuntu0.22.04.1 [1302 kB]
Get:5 http://mirror.yandex.ru/ubuntu jammy/universe amd64 sysbench amd64 1.0.20+ds-2 [120 kB]
Fetched 1711 kB in 0s (12.2 MB/s)
Selecting previously unselected package libluajit-5.1-common.
(Reading database ... 117313 files and directories currently installed.)
Preparing to unpack .../libluajit-5.1-common_2.1.0~beta3+dfsg-6_all.deb ...
Unpacking libluajit-5.1-common (2.1.0~beta3+dfsg-6) ...
Selecting previously unselected package libluajit-5.1-2:amd64.
Preparing to unpack .../libluajit-5.1-2_2.1.0~beta3+dfsg-6_amd64.deb ...
Unpacking libluajit-5.1-2:amd64 (2.1.0~beta3+dfsg-6) ...
Selecting previously unselected package mysql-common.
Preparing to unpack .../mysql-common_5.8+1.0.8_all.deb ...
Unpacking mysql-common (5.8+1.0.8) ...
Selecting previously unselected package libmysqlclient21:amd64.
Preparing to unpack .../libmysqlclient21_8.0.36-0ubuntu0.22.04.1_amd64.deb ...
Unpacking libmysqlclient21:amd64 (8.0.36-0ubuntu0.22.04.1) ...
Selecting previously unselected package sysbench.
Preparing to unpack .../sysbench_1.0.20+ds-2_amd64.deb ...
Unpacking sysbench (1.0.20+ds-2) ...
Setting up mysql-common (5.8+1.0.8) ...
update-alternatives: using /etc/mysql/my.cnf.fallback to provide /etc/mysql/my.cnf (my.cnf) in auto mode
Setting up libmysqlclient21:amd64 (8.0.36-0ubuntu0.22.04.1) ...
Setting up libluajit-5.1-common (2.1.0~beta3+dfsg-6) ...
Setting up libluajit-5.1-2:amd64 (2.1.0~beta3+dfsg-6) ...
Setting up sysbench (1.0.20+ds-2) ...
Processing triggers for man-db (2.10.2-1) ...
Processing triggers for libc-bin (2.35-0ubuntu3.7) ...
Scanning processes...
Scanning candidates...
Scanning linux images...

Restarting services...
Service restarts being deferred:
 systemctl restart networkd-dispatcher.service
 systemctl restart unattended-upgrades.service

No containers need to be restarted.

No user sessions are running outdated binaries.

No VM guests are running outdated hypervisor (qemu) binaries on this host.
```
Затем создаю юзера PostgreSQL для дальнейших тестов sysbench:
```
karussia@kntxt-vm:~$ sudo -u postgres psql
psql (15.7 (Ubuntu 15.7-1.pgdg22.04+1))
Type "help" for help.

postgres=# CREATE USER "kntxt-user" WITH PASSWORD 'charlottedewitte';
CREATE ROLE
postgres=# GRANT USAGE, CREATE ON SCHEMA public TO "kntxt-user";
GRANT
postgres=# ALTER ROLE "kntxt-user" CREATEDB;
ALTER ROLE
postgres=# GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO "kntxt-user";
GRANT
postgres=# ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL PRIVILEGES ON TABLES TO "kntxt-user";
ALTER DEFAULT PRIVILEGES
postgres=# \q
```
Теперь подготавливаю базу данных используя аргументы --tables=10 и --table-size=1000000 указывают, что должно быть создано 10 таблиц, каждая из которых содержит 1 миллион строк.
```
karussia@kntxt-vm:~$ sysbench /usr/share/sysbench/oltp_read_write.lua --db-driver=pgsql --pgsql-db=postgres --pgsql-user="kntxt-user" --pgsql-password='charlottedewitte' --tables=10 --table-size=1000000 prepare
sysbench 1.0.20 (using system LuaJIT 2.1.0-beta3)

Creating table 'sbtest1'...
Inserting 1000000 records into 'sbtest1'
Creating a secondary index on 'sbtest1'...
Creating table 'sbtest2'...
Inserting 1000000 records into 'sbtest2'
Creating a secondary index on 'sbtest2'...
Creating table 'sbtest3'...
Inserting 1000000 records into 'sbtest3'
Creating a secondary index on 'sbtest3'...
Creating table 'sbtest4'...
Inserting 1000000 records into 'sbtest4'
Creating a secondary index on 'sbtest4'...
Creating table 'sbtest5'...
Inserting 1000000 records into 'sbtest5'
Creating a secondary index on 'sbtest5'...
Creating table 'sbtest6'...
Inserting 1000000 records into 'sbtest6'
Creating a secondary index on 'sbtest6'...
Creating table 'sbtest7'...
Inserting 1000000 records into 'sbtest7'
Creating a secondary index on 'sbtest7'...
Creating table 'sbtest8'...
Inserting 1000000 records into 'sbtest8'
Creating a secondary index on 'sbtest8'...
Creating table 'sbtest9'...
Inserting 1000000 records into 'sbtest9'
Creating a secondary index on 'sbtest9'...
Creating table 'sbtest10'...
Inserting 1000000 records into 'sbtest10'
Creating a secondary index on 'sbtest10'...
```
Теперь буду запускать тест командой
```
sysbench /usr/share/sysbench/oltp_read_write.lua --db-driver=pgsql --pgsql-db=postgres --pgsql-user="kntxt-user" --pgsql-password='charlottedewitte' --tables=10 --table-size=1000000 --threads=4 --time=600 --report-interval=10 run
```
Эта команда будет симулировать нагрузку на базе данных PostgreSQL, выполняя смешанные операции чтения и записи с данными.
- "--threads=4": Этот параметр указывает, что для бенчмарка должно использоваться 4 потока. Вы можете скорректировать это число в зависимости от количества доступных ядер процессора и желаемой интенсивности симулируемой нагрузки.
- "--time=600": Бенчмарк будет выполняться в течение 600 секунд (10 минут). Скорректируйте эту продолжительность в зависимости от того, как долго вы хотите провести тест и насколько полными вы хотите получить результаты.
- "--report-interval=10": Sysbench будет сообщать промежуточные результаты каждые 10 секунд, предоставляя вам детализированное представление о производительности в течение всего теста.

```
karussia@kntxt-vm:~$ sysbench /usr/share/sysbench/oltp_read_write.lua --db-driver=pgsql --pgsql-db=postgres --pgsql-user="kntxt-user" --pgsql-password='charlottedewitte' --tables=10 --table-size=1000000 --threads=4 --time=600 --report-interval=10 run
sysbench 1.0.20 (using system LuaJIT 2.1.0-beta3)

Running the test with following options:
Number of threads: 4
Report intermediate results every 10 second(s)
Initializing random number generator from current time


Initializing worker threads...

Threads started!

[ 10s ] thds: 4 tps: 320.64 qps: 6416.38 (r/w/o: 4492.12/1282.58/641.69) lat (ms,95%): 81.48 err/s: 0.00 reconn/s: 0.00
[ 20s ] thds: 4 tps: 502.59 qps: 10053.37 (r/w/o: 7037.81/2010.37/1005.19) lat (ms,95%): 46.63 err/s: 0.00 reconn/s: 0.00
[ 30s ] thds: 4 tps: 702.11 qps: 14042.86 (r/w/o: 9829.71/2808.93/1404.22) lat (ms,95%): 21.89 err/s: 0.00 reconn/s: 0.00
[ 40s ] thds: 4 tps: 1061.99 qps: 21238.18 (r/w/o: 14866.62/4247.58/2123.99) lat (ms,95%): 5.18 err/s: 0.00 reconn/s: 0.00
[ 50s ] thds: 4 tps: 1125.70 qps: 22514.97 (r/w/o: 15760.65/4502.91/2251.41) lat (ms,95%): 4.49 err/s: 0.00 reconn/s: 0.00
[ 60s ] thds: 4 tps: 1040.18 qps: 20802.04 (r/w/o: 14561.18/4160.51/2080.35) lat (ms,95%): 5.88 err/s: 0.00 reconn/s: 0.00
[ 70s ] thds: 4 tps: 1078.83 qps: 21578.63 (r/w/o: 15105.67/4315.31/2157.65) lat (ms,95%): 5.37 err/s: 0.00 reconn/s: 0.00
[ 80s ] thds: 4 tps: 1195.20 qps: 23904.66 (r/w/o: 16732.74/4781.51/2390.41) lat (ms,95%): 4.18 err/s: 0.00 reconn/s: 0.00
[ 90s ] thds: 4 tps: 1433.40 qps: 28665.95 (r/w/o: 20066.34/5732.91/2866.71) lat (ms,95%): 4.03 err/s: 0.00 reconn/s: 0.00
[ 100s ] thds: 4 tps: 1114.19 qps: 22285.11 (r/w/o: 15599.46/4457.16/2228.48) lat (ms,95%): 3.89 err/s: 0.00 reconn/s: 0.00
[ 110s ] thds: 4 tps: 596.99 qps: 11936.63 (r/w/o: 8355.08/2387.57/1193.98) lat (ms,95%): 3.36 err/s: 0.00 reconn/s: 0.00
[ 120s ] thds: 4 tps: 793.09 qps: 15863.84 (r/w/o: 11105.22/3172.45/1586.17) lat (ms,95%): 5.18 err/s: 0.00 reconn/s: 0.00
[ 130s ] thds: 4 tps: 726.71 qps: 14535.83 (r/w/o: 10175.39/2907.03/1453.41) lat (ms,95%): 5.77 err/s: 0.00 reconn/s: 0.00
[ 140s ] thds: 4 tps: 1042.13 qps: 20841.19 (r/w/o: 14588.68/4168.24/2084.27) lat (ms,95%): 4.82 err/s: 0.00 reconn/s: 0.00
[ 150s ] thds: 4 tps: 1154.10 qps: 23083.25 (r/w/o: 16158.17/4616.69/2308.40) lat (ms,95%): 3.62 err/s: 0.10 reconn/s: 0.00
[ 160s ] thds: 4 tps: 1062.60 qps: 21251.43 (r/w/o: 14875.92/4250.31/2125.20) lat (ms,95%): 3.68 err/s: 0.00 reconn/s: 0.00
[ 170s ] thds: 4 tps: 1153.96 qps: 23080.10 (r/w/o: 16156.34/4615.84/2307.92) lat (ms,95%): 4.03 err/s: 0.00 reconn/s: 0.00
[ 180s ] thds: 4 tps: 891.59 qps: 17832.16 (r/w/o: 12482.60/3566.37/1783.19) lat (ms,95%): 5.77 err/s: 0.00 reconn/s: 0.00
[ 190s ] thds: 4 tps: 1021.74 qps: 20433.22 (r/w/o: 14302.81/4086.94/2043.47) lat (ms,95%): 5.28 err/s: 0.00 reconn/s: 0.00
[ 200s ] thds: 4 tps: 940.28 qps: 18806.75 (r/w/o: 13165.05/3761.13/1880.56) lat (ms,95%): 4.91 err/s: 0.00 reconn/s: 0.00
[ 210s ] thds: 4 tps: 885.69 qps: 17715.37 (r/w/o: 12401.21/3542.77/1771.39) lat (ms,95%): 4.41 err/s: 0.00 reconn/s: 0.00
[ 220s ] thds: 4 tps: 1219.72 qps: 24393.67 (r/w/o: 17075.16/4878.97/2439.54) lat (ms,95%): 4.49 err/s: 0.00 reconn/s: 0.00
[ 230s ] thds: 4 tps: 1374.89 qps: 27498.71 (r/w/o: 19249.37/5499.56/2749.78) lat (ms,95%): 4.18 err/s: 0.00 reconn/s: 0.00
[ 240s ] thds: 4 tps: 1207.60 qps: 24150.30 (r/w/o: 16904.90/4830.20/2415.20) lat (ms,95%): 4.74 err/s: 0.00 reconn/s: 0.00
[ 250s ] thds: 4 tps: 1151.30 qps: 23027.35 (r/w/o: 16119.13/4605.41/2302.80) lat (ms,95%): 4.82 err/s: 0.10 reconn/s: 0.00
[ 260s ] thds: 4 tps: 1226.60 qps: 24532.63 (r/w/o: 17172.92/4906.51/2453.20) lat (ms,95%): 4.74 err/s: 0.00 reconn/s: 0.00
[ 270s ] thds: 4 tps: 1014.50 qps: 20290.92 (r/w/o: 14204.04/4057.88/2028.99) lat (ms,95%): 5.37 err/s: 0.00 reconn/s: 0.00
[ 280s ] thds: 4 tps: 1048.99 qps: 20978.15 (r/w/o: 14684.19/4195.97/2097.98) lat (ms,95%): 5.18 err/s: 0.00 reconn/s: 0.00
[ 290s ] thds: 4 tps: 956.11 qps: 19122.56 (r/w/o: 13385.91/3824.43/1912.22) lat (ms,95%): 5.67 err/s: 0.00 reconn/s: 0.00
[ 300s ] thds: 4 tps: 892.68 qps: 17853.97 (r/w/o: 12497.87/3570.73/1785.37) lat (ms,95%): 6.43 err/s: 0.00 reconn/s: 0.00
[ 310s ] thds: 4 tps: 997.42 qps: 19948.82 (r/w/o: 13964.30/3989.68/1994.84) lat (ms,95%): 5.88 err/s: 0.00 reconn/s: 0.00
[ 320s ] thds: 4 tps: 1096.10 qps: 21922.12 (r/w/o: 15345.24/4384.68/2192.19) lat (ms,95%): 5.28 err/s: 0.00 reconn/s: 0.00
[ 330s ] thds: 4 tps: 1133.51 qps: 22669.58 (r/w/o: 15868.82/4533.74/2267.02) lat (ms,95%): 5.18 err/s: 0.00 reconn/s: 0.00
[ 340s ] thds: 4 tps: 1183.07 qps: 23663.95 (r/w/o: 16565.14/4732.37/2366.43) lat (ms,95%): 2.97 err/s: 0.10 reconn/s: 0.00
[ 350s ] thds: 4 tps: 415.29 qps: 8306.99 (r/w/o: 5814.95/1661.46/830.58) lat (ms,95%): 4.65 err/s: 0.00 reconn/s: 0.00
[ 360s ] thds: 4 tps: 763.04 qps: 15258.82 (r/w/o: 10680.90/3051.84/1526.07) lat (ms,95%): 4.33 err/s: 0.00 reconn/s: 0.00
[ 370s ] thds: 4 tps: 1241.73 qps: 24834.74 (r/w/o: 17384.38/4966.91/2483.45) lat (ms,95%): 3.19 err/s: 0.00 reconn/s: 0.00
[ 380s ] thds: 4 tps: 1588.09 qps: 31762.99 (r/w/o: 22234.32/6352.48/3176.19) lat (ms,95%): 3.30 err/s: 0.00 reconn/s: 0.00
[ 390s ] thds: 4 tps: 1466.39 qps: 29327.05 (r/w/o: 20528.40/5865.87/2932.79) lat (ms,95%): 3.68 err/s: 0.00 reconn/s: 0.00
[ 400s ] thds: 4 tps: 1454.00 qps: 29078.79 (r/w/o: 20355.06/5815.72/2908.01) lat (ms,95%): 3.75 err/s: 0.00 reconn/s: 0.00
[ 410s ] thds: 4 tps: 1404.98 qps: 28100.90 (r/w/o: 19671.15/5619.80/2809.95) lat (ms,95%): 4.03 err/s: 0.00 reconn/s: 0.00
[ 420s ] thds: 4 tps: 1042.69 qps: 20852.62 (r/w/o: 14596.41/4170.84/2085.37) lat (ms,95%): 5.57 err/s: 0.00 reconn/s: 0.00
[ 430s ] thds: 4 tps: 1098.84 qps: 21980.36 (r/w/o: 15386.80/4395.67/2197.89) lat (ms,95%): 5.47 err/s: 0.10 reconn/s: 0.00
[ 440s ] thds: 4 tps: 1390.29 qps: 27805.32 (r/w/o: 19463.58/5561.16/2780.58) lat (ms,95%): 3.89 err/s: 0.00 reconn/s: 0.00
[ 450s ] thds: 4 tps: 1362.19 qps: 27242.28 (r/w/o: 19069.35/5448.56/2724.38) lat (ms,95%): 3.75 err/s: 0.00 reconn/s: 0.00
[ 460s ] thds: 4 tps: 1329.22 qps: 26584.66 (r/w/o: 18609.15/5317.17/2658.34) lat (ms,95%): 3.89 err/s: 0.00 reconn/s: 0.00
[ 470s ] thds: 4 tps: 1291.98 qps: 25840.93 (r/w/o: 18089.27/5167.61/2584.05) lat (ms,95%): 3.89 err/s: 0.00 reconn/s: 0.00
[ 480s ] thds: 4 tps: 1059.99 qps: 21200.15 (r/w/o: 14839.93/4240.15/2120.08) lat (ms,95%): 5.57 err/s: 0.10 reconn/s: 0.00
[ 490s ] thds: 4 tps: 942.81 qps: 18856.10 (r/w/o: 13198.97/3771.42/1885.71) lat (ms,95%): 6.09 err/s: 0.00 reconn/s: 0.00
[ 500s ] thds: 4 tps: 1342.53 qps: 26853.32 (r/w/o: 18797.63/5370.62/2685.06) lat (ms,95%): 4.91 err/s: 0.00 reconn/s: 0.00
[ 510s ] thds: 4 tps: 1808.89 qps: 36176.28 (r/w/o: 25323.42/7235.08/3617.79) lat (ms,95%): 2.48 err/s: 0.00 reconn/s: 0.00
[ 520s ] thds: 4 tps: 1504.30 qps: 30086.20 (r/w/o: 21060.30/6017.30/3008.60) lat (ms,95%): 3.13 err/s: 0.00 reconn/s: 0.00
[ 530s ] thds: 4 tps: 475.49 qps: 9509.18 (r/w/o: 6656.52/1901.68/950.99) lat (ms,95%): 3.43 err/s: 0.00 reconn/s: 0.00
[ 540s ] thds: 4 tps: 770.78 qps: 15415.86 (r/w/o: 10791.06/3083.23/1541.57) lat (ms,95%): 5.00 err/s: 0.00 reconn/s: 0.00
[ 550s ] thds: 4 tps: 1280.05 qps: 25597.86 (r/w/o: 17917.77/5120.09/2560.00) lat (ms,95%): 3.75 err/s: 0.00 reconn/s: 0.00
[ 560s ] thds: 4 tps: 1377.40 qps: 27552.20 (r/w/o: 19287.30/5509.80/2755.10) lat (ms,95%): 3.75 err/s: 0.10 reconn/s: 0.00
[ 570s ] thds: 4 tps: 1273.70 qps: 25475.24 (r/w/o: 17833.03/5094.81/2547.40) lat (ms,95%): 3.96 err/s: 0.00 reconn/s: 0.00
[ 580s ] thds: 4 tps: 1277.31 qps: 25544.30 (r/w/o: 17880.47/5109.22/2554.61) lat (ms,95%): 3.82 err/s: 0.00 reconn/s: 0.00
[ 590s ] thds: 4 tps: 1142.69 qps: 22852.98 (r/w/o: 15996.84/4570.76/2285.38) lat (ms,95%): 4.49 err/s: 0.00 reconn/s: 0.00
[ 600s ] thds: 4 tps: 991.89 qps: 19840.69 (r/w/o: 13888.93/3967.98/1983.79) lat (ms,95%): 6.21 err/s: 0.00 reconn/s: 0.00
SQL statistics:
    queries performed:
        read:                            9162132
        write:                           2617738
        other:                           1308878
        total:                           13088748
    transactions:                        654432 (1090.70 per sec.)
    queries:                             13088748 (21814.15 per sec.)
    ignored errors:                      6      (0.01 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          600.0102s
    total number of events:              654432

Latency (ms):
         min:                                    1.11
         avg:                                    3.67
         max:                                 1481.72
         95th percentile:                        4.91
         sum:                              2398616.00

Threads fairness:
    events (avg/stddev):           163608.0000/833.19
    execution time (avg/stddev):   599.6540/0.00
```
Итоги тестирования:
- "total number of events": За 600-секундный период тестирования было обработано 654,432 транзакции. Это общее количество транзакций, которые Sysbench смог завершить.
- "transactions": В среднем система обрабатывала около 1090.70 транзакций в секунду. Это ключевой показатель, так как он демонстрирует пропускную способность транзакций вашей системы.
- "queries": В секунду выполнялось 21814.15 запросов. Это включает чтение, запись и другие типы запросов.
- "ignored errors": 6 ошибок были проигнорированы. По идее это не критические ошибки, которые не мешают продолжению теста.
- "reconnects": Переподключений не было, что указывает на стабильное подключение на протяжении всего теста.
Latency Метрики :
- "min" Минимальная задержка: 1.11 мс
- "avg" Средняя задержка: 3.67 мс
- "max" Максимальная задержка: 1481.72 мс
- "95th percentile" Задержка 95-го процентиля: 4.91 мс
- Latency (задержка) — мера времени, необходимого базе данных для завершения транзакции. Моя средняя задержка в 3.67 мс и задержка 95-го процентиля в 4.91 мс свидетельствуют о отзывчивости системы под нагрузкой. Однако максимальная задержка в 1481.72 мс указывает на случайные задержки, возможно, из-за сложных запросов или конкуренции системы.


# Сценарий High Concurrency
## 3. Настроить кластер PostgreSQL 15 на максимальную производительность не обращая внимание на возможные проблемы с надежностью в случае аварийной перезагрузки виртуальной машины
Применяю обозначенные выше значения параметров для High Concurrency и перезагружаю:
```
karussia@kntxt-vm:~$ sudo nano /etc/postgresql/15/main/postgresql.conf
karussia@kntxt-vm:~$ sudo systemctl restart postgresql
```
## 4. Нагрузить кластер через утилиту через утилиту pgbench (https://postgrespro.ru/docs/postgrespro/14/pgbench) + 5. написать какого значения tps удалось достичь, показать какие параметры в какие значения устанавливали и почему
Использую "pgbench -i -s 50 postgres" для инициализации и "pgbench -c 10 -j 2 -T 600 postgres" для запуска теста:
```
karussia@kntxt-vm:~$ sudo -i -u postgres
postgres@kntxt-vm:~$ pgbench -i -s 50 postgres
dropping old tables...
creating tables...
generating data (client-side)...
5000000 of 5000000 tuples (100%) done (elapsed 54.91 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 104.97 s (drop tables 0.12 s, create tables 0.04 s, client-side generate 56.44 s, vacuum 0.97 s, primary keys 47.40 s).
postgres@kntxt-vm:~$ pgbench -c 10 -j 2 -T 600 postgres
pgbench (15.7 (Ubuntu 15.7-1.pgdg22.04+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 50
query mode: simple
number of clients: 10
number of threads: 2
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 572099
number of failed transactions: 0 (0.000%)
latency average = 10.488 ms
initial connection time = 11.509 ms
tps = 953.473235 (without initial connection time)
```
Итоги:
- "transaction type": также TPC-B
- "scaling factor": 50.
- "query mode": simple. В этом режиме запросы отправляются на сервер без предварительной обработки или разбора. Этот режим тестирует способность базы данных обрабатывать запросы в исходном виде.
- "number of clients", "number of threads" и "duration" - это параметры ранее указанные в команде "pgbench -c 10 -j 2 -T 600 postgres"
- "number of transactions actually processed": 572,099. Это число представляет общее количество транзакций, завершенных во время теста. Это критически важно для понимания пропускной способности транзакций базы данных при настроенных параметрах.
- "number of failed transactions": 0 - свидетельствует о высокой надежности во время теста, без сбоев транзакций, несмотря на высокую нагрузку и уровень одновременности.
- "latency average": 10.488 мс. - по сравнению с минимальным сценарием эта задержка выше, вероятно, из-за увеличенной нагрузки и сложности операций при более высоких настройках одновременности.
- "initial connection time": 11.509 мс. - немного выше, чем в Minimal сценарии, возможно, из-за большего количества одновременно инициализируемых соединений.
- "tps" (Транзакции в Секунду): 953.473 - Более низкий TPS по сравнению с минимальным сценарием может быть вызван более высокой системной нагрузкой и увеличенной сложностью управления большим количеством соединений и большими объемами данных.

## 6. аналогично протестировать через утилиту sysbench:
подготавливаю базу данных c предварительным удалением прошлых тестовых таблиц
```
karussia@kntxt-vm:~$ sudo -u postgres psql
psql (15.7 (Ubuntu 15.7-1.pgdg22.04+1))
Type "help" for help.

postgres=# DROP TABLE IF EXISTS sbtest1, sbtest2, sbtest3, sbtest4, sbtest5, sbtest6, sbtest7, sbtest8, sbtest9, sbtest10;
DROP TABLE
postgres=# SELECT tablename FROM pg_tables WHERE tablename LIKE 'sbtest%';
 tablename
-----------
(0 rows)

postgres=# \q

karussia@kntxt-vm:~$ sysbench /usr/share/sysbench/oltp_read_write.lua --db-driver=pgsql --pgsql-db=postgres --pgsql-user="kntxt-user" --pgsql-password='charlottedewitte' cleanup
sysbench 1.0.20 (using system LuaJIT 2.1.0-beta3)

Dropping table 'sbtest1'...
karussia@kntxt-vm:~$ sysbench /usr/share/sysbench/oltp_read_write.lua --db-driver=pgsql --pgsql-db=postgres --pgsql-user="kntxt-user" --pgsql-password='charlottedewitte' --tables=10 --table-size=1000000 prepare
sysbench 1.0.20 (using system LuaJIT 2.1.0-beta3)

Creating table 'sbtest1'...
Inserting 1000000 records into 'sbtest1'
Creating a secondary index on 'sbtest1'...
Creating table 'sbtest2'...
Inserting 1000000 records into 'sbtest2'
Creating a secondary index on 'sbtest2'...
Creating table 'sbtest3'...
Inserting 1000000 records into 'sbtest3'
Creating a secondary index on 'sbtest3'...
Creating table 'sbtest4'...
Inserting 1000000 records into 'sbtest4'
Creating a secondary index on 'sbtest4'...
Creating table 'sbtest5'...
Inserting 1000000 records into 'sbtest5'
Creating a secondary index on 'sbtest5'...
Creating table 'sbtest6'...
Inserting 1000000 records into 'sbtest6'
Creating a secondary index on 'sbtest6'...
Creating table 'sbtest7'...
Inserting 1000000 records into 'sbtest7'
Creating a secondary index on 'sbtest7'...
Creating table 'sbtest8'...
Inserting 1000000 records into 'sbtest8'
Creating a secondary index on 'sbtest8'...
Creating table 'sbtest9'...
Inserting 1000000 records into 'sbtest9'
Creating a secondary index on 'sbtest9'...
Creating table 'sbtest10'...
Inserting 1000000 records into 'sbtest10'
Creating a secondary index on 'sbtest10'...

```
Запускаю тест:

```
karussia@kntxt-vm:~$ sysbench /usr/share/sysbench/oltp_read_write.lua --db-driver=pgsql --pgsql-db=postgres --pgsql-user="kntxt-user" --pgsql-password='charlottedewitte' --tables=10 --table-size=1000000 --threads=4 --time=600 --report-interval=10 run
sysbench 1.0.20 (using system LuaJIT 2.1.0-beta3)

Running the test with following options:
Number of threads: 4
Report intermediate results every 10 second(s)
Initializing random number generator from current time


Initializing worker threads...

Threads started!

[ 10s ] thds: 4 tps: 283.35 qps: 5674.61 (r/w/o: 3972.51/1135.00/567.10) lat (ms,95%): 46.63 err/s: 0.00 reconn/s: 0.00
[ 20s ] thds: 4 tps: 481.60 qps: 9631.96 (r/w/o: 6742.37/1926.39/963.20) lat (ms,95%): 17.63 err/s: 0.00 reconn/s: 0.00
[ 30s ] thds: 4 tps: 146.40 qps: 2927.94 (r/w/o: 2049.56/585.59/292.79) lat (ms,95%): 87.56 err/s: 0.00 reconn/s: 0.00
[ 40s ] thds: 4 tps: 215.61 qps: 4312.11 (r/w/o: 3018.48/862.42/431.21) lat (ms,95%): 74.46 err/s: 0.00 reconn/s: 0.00
[ 50s ] thds: 4 tps: 278.00 qps: 5560.00 (r/w/o: 3892.00/1112.00/556.00) lat (ms,95%): 63.32 err/s: 0.00 reconn/s: 0.00
[ 60s ] thds: 4 tps: 410.10 qps: 8199.73 (r/w/o: 5740.22/1639.31/820.20) lat (ms,95%): 38.94 err/s: 0.00 reconn/s: 0.00
[ 70s ] thds: 4 tps: 466.20 qps: 9323.18 (r/w/o: 6526.48/1864.30/932.40) lat (ms,95%): 27.17 err/s: 0.00 reconn/s: 0.00
[ 80s ] thds: 4 tps: 528.60 qps: 10575.06 (r/w/o: 7401.87/2115.99/1057.20) lat (ms,95%): 22.28 err/s: 0.00 reconn/s: 0.00
[ 90s ] thds: 4 tps: 481.19 qps: 9621.28 (r/w/o: 6735.35/1923.56/962.38) lat (ms,95%): 22.28 err/s: 0.00 reconn/s: 0.00
[ 100s ] thds: 4 tps: 558.99 qps: 11180.53 (r/w/o: 7826.18/2236.37/1117.98) lat (ms,95%): 13.46 err/s: 0.00 reconn/s: 0.00
[ 110s ] thds: 4 tps: 656.73 qps: 13136.42 (r/w/o: 9195.24/2627.72/1313.46) lat (ms,95%): 13.95 err/s: 0.00 reconn/s: 0.00
[ 120s ] thds: 4 tps: 591.10 qps: 11818.91 (r/w/o: 8273.74/2362.98/1182.19) lat (ms,95%): 18.95 err/s: 0.00 reconn/s: 0.00
[ 130s ] thds: 4 tps: 715.30 qps: 14306.34 (r/w/o: 10013.96/2861.79/1430.59) lat (ms,95%): 12.30 err/s: 0.00 reconn/s: 0.00
[ 140s ] thds: 4 tps: 722.89 qps: 14456.78 (r/w/o: 10119.45/2891.56/1445.78) lat (ms,95%): 11.45 err/s: 0.00 reconn/s: 0.00
[ 150s ] thds: 4 tps: 498.70 qps: 9975.58 (r/w/o: 6983.36/1994.82/997.41) lat (ms,95%): 17.01 err/s: 0.00 reconn/s: 0.00
[ 160s ] thds: 4 tps: 155.70 qps: 3116.10 (r/w/o: 2181.10/623.60/311.40) lat (ms,95%): 99.33 err/s: 0.00 reconn/s: 0.00
[ 170s ] thds: 4 tps: 172.50 qps: 3451.85 (r/w/o: 2416.43/690.21/345.20) lat (ms,95%): 84.47 err/s: 0.10 reconn/s: 0.00
[ 180s ] thds: 4 tps: 227.50 qps: 4549.96 (r/w/o: 3184.97/909.99/455.00) lat (ms,95%): 66.84 err/s: 0.00 reconn/s: 0.00
[ 190s ] thds: 4 tps: 286.50 qps: 5729.95 (r/w/o: 4010.97/1145.99/573.00) lat (ms,95%): 59.99 err/s: 0.00 reconn/s: 0.00
[ 200s ] thds: 4 tps: 343.30 qps: 6866.05 (r/w/o: 4806.23/1373.21/686.60) lat (ms,95%): 54.83 err/s: 0.00 reconn/s: 0.00
[ 210s ] thds: 4 tps: 325.70 qps: 6508.70 (r/w/o: 4555.73/1301.48/651.49) lat (ms,95%): 43.39 err/s: 0.00 reconn/s: 0.00
[ 220s ] thds: 4 tps: 447.90 qps: 8963.30 (r/w/o: 6274.67/1792.82/895.81) lat (ms,95%): 34.95 err/s: 0.00 reconn/s: 0.00
[ 230s ] thds: 4 tps: 482.61 qps: 9652.13 (r/w/o: 6756.49/1930.43/965.21) lat (ms,95%): 31.94 err/s: 0.00 reconn/s: 0.00
[ 240s ] thds: 4 tps: 475.70 qps: 9512.55 (r/w/o: 6659.17/1901.99/951.40) lat (ms,95%): 24.83 err/s: 0.00 reconn/s: 0.00
[ 250s ] thds: 4 tps: 529.70 qps: 10593.68 (r/w/o: 7415.25/2119.02/1059.41) lat (ms,95%): 25.74 err/s: 0.00 reconn/s: 0.00
[ 260s ] thds: 4 tps: 556.80 qps: 11137.77 (r/w/o: 7796.38/2227.79/1113.60) lat (ms,95%): 20.74 err/s: 0.00 reconn/s: 0.00
[ 270s ] thds: 4 tps: 503.99 qps: 10078.01 (r/w/o: 7054.89/2015.14/1007.97) lat (ms,95%): 22.28 err/s: 0.00 reconn/s: 0.00
[ 280s ] thds: 4 tps: 569.00 qps: 11377.27 (r/w/o: 7964.05/2275.21/1138.01) lat (ms,95%): 12.98 err/s: 0.00 reconn/s: 0.00
[ 290s ] thds: 4 tps: 481.81 qps: 9640.13 (r/w/o: 6748.06/1928.45/963.62) lat (ms,95%): 16.12 err/s: 0.00 reconn/s: 0.00
[ 300s ] thds: 4 tps: 149.90 qps: 2998.60 (r/w/o: 2098.80/600.00/299.80) lat (ms,95%): 89.16 err/s: 0.00 reconn/s: 0.00
[ 310s ] thds: 4 tps: 186.60 qps: 3731.48 (r/w/o: 2612.28/745.90/373.30) lat (ms,95%): 89.16 err/s: 0.00 reconn/s: 0.00
[ 320s ] thds: 4 tps: 243.60 qps: 4872.42 (r/w/o: 3410.44/974.78/487.19) lat (ms,95%): 73.13 err/s: 0.00 reconn/s: 0.00
[ 330s ] thds: 4 tps: 293.40 qps: 5865.46 (r/w/o: 4106.67/1171.99/586.80) lat (ms,95%): 57.87 err/s: 0.00 reconn/s: 0.00
[ 340s ] thds: 4 tps: 338.30 qps: 6764.14 (r/w/o: 4734.32/1353.21/676.60) lat (ms,95%): 44.98 err/s: 0.00 reconn/s: 0.00
[ 350s ] thds: 4 tps: 384.41 qps: 7692.61 (r/w/o: 5384.55/1539.24/768.82) lat (ms,95%): 39.65 err/s: 0.00 reconn/s: 0.00
[ 360s ] thds: 4 tps: 414.90 qps: 8298.00 (r/w/o: 5808.60/1659.60/829.80) lat (ms,95%): 38.94 err/s: 0.00 reconn/s: 0.00
[ 370s ] thds: 4 tps: 469.79 qps: 9395.80 (r/w/o: 6577.06/1879.16/939.58) lat (ms,95%): 33.72 err/s: 0.00 reconn/s: 0.00
[ 380s ] thds: 4 tps: 514.41 qps: 10288.18 (r/w/o: 7201.73/2057.54/1028.92) lat (ms,95%): 26.20 err/s: 0.00 reconn/s: 0.00
[ 390s ] thds: 4 tps: 471.49 qps: 9427.12 (r/w/o: 6599.01/1885.14/942.97) lat (ms,95%): 22.28 err/s: 0.00 reconn/s: 0.00
[ 400s ] thds: 4 tps: 546.82 qps: 10937.21 (r/w/o: 7656.71/2186.86/1093.63) lat (ms,95%): 19.65 err/s: 0.00 reconn/s: 0.00
[ 410s ] thds: 4 tps: 724.30 qps: 14483.06 (r/w/o: 10137.67/2896.79/1448.60) lat (ms,95%): 11.24 err/s: 0.00 reconn/s: 0.00
[ 420s ] thds: 4 tps: 502.40 qps: 10052.67 (r/w/o: 7036.65/2011.21/1004.81) lat (ms,95%): 20.00 err/s: 0.00 reconn/s: 0.00
[ 430s ] thds: 4 tps: 149.40 qps: 2987.99 (r/w/o: 2091.59/597.60/298.80) lat (ms,95%): 94.10 err/s: 0.00 reconn/s: 0.00
[ 440s ] thds: 4 tps: 189.40 qps: 3787.99 (r/w/o: 2651.59/757.60/378.80) lat (ms,95%): 86.00 err/s: 0.00 reconn/s: 0.00
[ 450s ] thds: 4 tps: 240.39 qps: 4806.65 (r/w/o: 3365.40/960.47/480.79) lat (ms,95%): 65.65 err/s: 0.00 reconn/s: 0.00
[ 460s ] thds: 4 tps: 286.50 qps: 5731.20 (r/w/o: 4011.10/1147.10/573.00) lat (ms,95%): 53.85 err/s: 0.00 reconn/s: 0.00
[ 470s ] thds: 4 tps: 336.61 qps: 6732.20 (r/w/o: 4712.54/1346.34/673.32) lat (ms,95%): 45.79 err/s: 0.00 reconn/s: 0.00
[ 480s ] thds: 4 tps: 391.10 qps: 7822.07 (r/w/o: 5475.45/1564.41/782.21) lat (ms,95%): 38.25 err/s: 0.00 reconn/s: 0.00
[ 490s ] thds: 4 tps: 426.60 qps: 8531.90 (r/w/o: 5972.33/1706.38/853.19) lat (ms,95%): 38.25 err/s: 0.00 reconn/s: 0.00
[ 500s ] thds: 4 tps: 454.09 qps: 9078.98 (r/w/o: 6356.02/1814.78/908.19) lat (ms,95%): 34.33 err/s: 0.00 reconn/s: 0.00
[ 510s ] thds: 4 tps: 439.00 qps: 8781.53 (r/w/o: 6146.75/1756.79/877.99) lat (ms,95%): 25.74 err/s: 0.00 reconn/s: 0.00
[ 520s ] thds: 4 tps: 537.00 qps: 10737.00 (r/w/o: 7515.70/2147.40/1073.90) lat (ms,95%): 13.46 err/s: 0.00 reconn/s: 0.00
[ 530s ] thds: 4 tps: 546.61 qps: 10936.60 (r/w/o: 7655.41/2187.86/1093.33) lat (ms,95%): 18.61 err/s: 0.00 reconn/s: 0.00
[ 540s ] thds: 4 tps: 705.00 qps: 14098.30 (r/w/o: 9869.47/2818.82/1410.01) lat (ms,95%): 12.75 err/s: 0.00 reconn/s: 0.00
[ 550s ] thds: 4 tps: 425.28 qps: 8507.47 (r/w/o: 5954.57/1702.33/850.57) lat (ms,95%): 21.89 err/s: 0.00 reconn/s: 0.00
[ 560s ] thds: 4 tps: 154.20 qps: 3081.20 (r/w/o: 2157.60/615.20/308.40) lat (ms,95%): 95.81 err/s: 0.00 reconn/s: 0.00
[ 570s ] thds: 4 tps: 195.10 qps: 3904.80 (r/w/o: 2732.60/782.00/390.20) lat (ms,95%): 77.19 err/s: 0.00 reconn/s: 0.00
[ 580s ] thds: 4 tps: 238.81 qps: 4774.38 (r/w/o: 3341.93/954.84/477.62) lat (ms,95%): 77.19 err/s: 0.00 reconn/s: 0.00
[ 590s ] thds: 4 tps: 290.60 qps: 5812.56 (r/w/o: 4068.97/1162.39/581.20) lat (ms,95%): 62.19 err/s: 0.00 reconn/s: 0.00
[ 600s ] thds: 4 tps: 345.80 qps: 6917.22 (r/w/o: 4842.01/1383.60/691.60) lat (ms,95%): 43.39 err/s: 0.00 reconn/s: 0.00
SQL statistics:
    queries performed:
        read:                            3386012
        write:                           967426
        other:                           483720
        total:                           4837158
    transactions:                        241857 (403.08 per sec.)
    queries:                             4837158 (8061.66 per sec.)
    ignored errors:                      1      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          600.0186s
    total number of events:              241857

Latency (ms):
         min:                                    1.97
         avg:                                    9.92
         max:                                  425.02
         95th percentile:                       35.59
         sum:                              2399513.76

Threads fairness:
    events (avg/stddev):           60464.2500/65.50
    execution time (avg/stddev):   599.8784/0.00
```
Итоги тестирования:

Ключевые метрики:
- Transactions Per Second (TPS): 403.08
- Queries Per Second (QPS): 8061.66
- Average Latency: 9.92 ms
- Maximum Latency: 425.02 ms
- 95th Percentile Latency: 35.59 ms

TPS значительно снизился с более чем 5000 в сценарии "Minimal" до около 400 в сценарии "High Concurrency". Это может свидетельствовать о том, что более высокая одновременность и большая нагрузка приводят к большему количеству конфликтов и более медленным временам обработки.
Задержка в сценарии "High Concurrency" намного выше, в среднем около 10 мс, по сравнению с менее чем 2 мс в сценарии "Minimal". Это отражает увеличенную сложность и потребности в ресурсах при обработке большего количества одновременных транзакций.
Средняя задержка значительно увеличилась по сравнению со сценарием "Minimal", что предполагает, что каждая транзакция требует больше времени для завершения.
Максимальная задержка и задержка 95-го процентиля также выше, что может указывать на большую изменчивость и потенциальные узкие места при более высоких нагрузках одновременности.


# Сценарий Aggressive Performance
## 3. Настроить кластер PostgreSQL 15 на максимальную производительность не обращая внимание на возможные проблемы с надежностью в случае аварийной перезагрузки виртуальной машины
Применяю обозначенные выше значения параметров для High Concurrency и перезагружаю:
```
karussia@kntxt-vm:~$ sudo nano /etc/postgresql/15/main/postgresql.conf
karussia@kntxt-vm:~$ sudo systemctl restart postgresql
```
## 4. Нагрузить кластер через утилиту через утилиту pgbench (https://postgrespro.ru/docs/postgrespro/14/pgbench) + 5. написать какого значения tps удалось достичь, показать какие параметры в какие значения устанавливали и почему
Использую "pgbench -i -s 50 postgres" для инициализации и "pgbench -c 10 -j 2 -T 600 postgres" для запуска теста:
```
karussia@kntxt-vm:~$ sudo -i -u postgres
postgres@kntxt-vm:~$ pgbench -i -s 50 postgres
dropping old tables...
creating tables...
generating data (client-side)...
5000000 of 5000000 tuples (100%) done (elapsed 68.66 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 93.93 s (drop tables 0.05 s, create tables 0.00 s, client-side generate 71.10 s, vacuum 2.09 s, primary keys 20.68 s).
postgres@kntxt-vm:~$ pgbench -c 10 -j 2 -T 600 postgres
pgbench (15.7 (Ubuntu 15.7-1.pgdg22.04+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 50
query mode: simple
number of clients: 10
number of threads: 2
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 4044947
number of failed transactions: 0 (0.000%)
latency average = 1.483 ms
initial connection time = 11.326 ms
tps = 6741.589345 (without initial connection time)
```
Итоги:

Ключевые метрики:
- Transactions Per Second (TPS): 6741.589345
- Average Latency: 1.483 ms

TPS значительно увеличился по сравнению со сценариями "High Concurrency" (403.08 TPS) и "Minimal" (5218.492266 TPS).\
Средняя задержка 1.483 мс исключительно низка, особенно по сравнению с 10.488 мс в сценарии "High Concurrency", что указывает на значительное улучшение времени отклика.\
Нулевое количество неудачных транзакций указывает на высокую надежность даже при агрессивных настройках производительности.

Настройки "Aggressive Performance" показывают лучшую производительность среди всех сценариев с точки зрения пропускной способности (TPS) и отклика (задержки). Ключевые изменения, которые, вероятно, способствовали этому улучшению производительности, включают:
- Увеличение shared_buffers и work_mem: Больший объем выделенной памяти позволяет обрабатывать больше данных в памяти, что снижает дисковый ввод-вывод и ускоряет обработку запросов.
- Увеличение max_wal_size и checkpoint_timeout: Эти настройки снижают частоту дисковых записей для WAL и контрольных точек, что может значительно улучшить производительность, особенно в средах с интенсивной записью, хотя и с немного увеличенным риском потери данных при сбое.
- Отключение synchronous_commit: Это повышает производительность записи, не дожидаясь подтверждения записей WAL перед завершением транзакций, улучшая TPS, но с риском потери очень последних транзакций в случае сбоя.
- Увеличение effective_io_concurrency: Лучшее использование возможностей SSD или RAID-конфигураций, позволяющее PostgreSQL обрабатывать больше операций ввода-вывода параллельно.

## 6. аналогично протестировать через утилиту sysbench:
подготавливаю базу данных c предварительным удалением прошлых тестовых таблиц
```
karussia@kntxt-vm:~$ sudo -u postgres psql
psql (15.7 (Ubuntu 15.7-1.pgdg22.04+1))
Type "help" for help.

postgres=# DROP TABLE IF EXISTS sbtest1, sbtest2, sbtest3, sbtest4, sbtest5, sbtest6, sbtest7, sbtest8, sbtest9, sbtest10;
DROP TABLE
postgres=# SELECT tablename FROM pg_tables WHERE tablename LIKE 'sbtest%';
 tablename
-----------
(0 rows)

postgres=# \q
karussia@kntxt-vm:~$ sysbench /usr/share/sysbench/oltp_read_write.lua --db-driver=pgsql --pgsql-db=postgres --pgsql-user="kntxt-user" --pgsql-password='charlottedewitte' cleanup
sysbench 1.0.20 (using system LuaJIT 2.1.0-beta3)

Dropping table 'sbtest1'...
karussia@kntxt-vm:~$ sysbench /usr/share/sysbench/oltp_read_write.lua --db-driver=pgsql --pgsql-db=postgres --pgsql-user="kntxt-user" --pgsql-password='charlottedewitte' --tables=10 --table-size=1000000 prepare
sysbench 1.0.20 (using system LuaJIT 2.1.0-beta3)

Creating table 'sbtest1'...
Inserting 1000000 records into 'sbtest1'
Creating a secondary index on 'sbtest1'...
Creating table 'sbtest2'...
Inserting 1000000 records into 'sbtest2'
Creating a secondary index on 'sbtest2'...
Creating table 'sbtest3'...
Inserting 1000000 records into 'sbtest3'
Creating a secondary index on 'sbtest3'...
Creating table 'sbtest4'...
Inserting 1000000 records into 'sbtest4'
Creating a secondary index on 'sbtest4'...
Creating table 'sbtest5'...
Inserting 1000000 records into 'sbtest5'
Creating a secondary index on 'sbtest5'...
Creating table 'sbtest6'...
Inserting 1000000 records into 'sbtest6'
Creating a secondary index on 'sbtest6'...
Creating table 'sbtest7'...
Inserting 1000000 records into 'sbtest7'
Creating a secondary index on 'sbtest7'...
Creating table 'sbtest8'...
Inserting 1000000 records into 'sbtest8'
Creating a secondary index on 'sbtest8'...
Creating table 'sbtest9'...
Inserting 1000000 records into 'sbtest9'
Creating a secondary index on 'sbtest9'...
Creating table 'sbtest10'...
Inserting 1000000 records into 'sbtest10'
Creating a secondary index on 'sbtest10'...
```

Запускаю тест:

```
karussia@kntxt-vm:~$ sysbench /usr/share/sysbench/oltp_read_write.lua --db-driver=pgsql --pgsql-db=postgres --pgsql-user="kntxt-user" --pgsql-password='charlottedewitte' --tables=10 --table-size=1000000 --threads=4 --time=600 --report-interval=10 run
sysbench 1.0.20 (using system LuaJIT 2.1.0-beta3)

Running the test with following options:
Number of threads: 4
Report intermediate results every 10 second(s)
Initializing random number generator from current time


Initializing worker threads...

Threads started!

[ 10s ] thds: 4 tps: 411.83 qps: 8242.56 (r/w/o: 5771.19/1647.31/824.06) lat (ms,95%): 51.94 err/s: 0.00 reconn/s: 0.00
[ 20s ] thds: 4 tps: 977.80 qps: 19554.44 (r/w/o: 13687.63/3911.21/1955.60) lat (ms,95%): 7.04 err/s: 0.00 reconn/s: 0.00
[ 30s ] thds: 4 tps: 1647.11 qps: 32940.99 (r/w/o: 23058.30/6588.46/3294.23) lat (ms,95%): 3.43 err/s: 0.00 reconn/s: 0.00
[ 40s ] thds: 4 tps: 1432.26 qps: 28646.82 (r/w/o: 20053.08/5729.22/2864.51) lat (ms,95%): 3.62 err/s: 0.00 reconn/s: 0.00
[ 50s ] thds: 4 tps: 1142.49 qps: 22849.62 (r/w/o: 15994.70/4569.94/2284.97) lat (ms,95%): 5.09 err/s: 0.00 reconn/s: 0.00
[ 60s ] thds: 4 tps: 1787.98 qps: 35759.42 (r/w/o: 25031.76/7151.70/3575.95) lat (ms,95%): 3.25 err/s: 0.00 reconn/s: 0.00
[ 70s ] thds: 4 tps: 1977.29 qps: 39545.74 (r/w/o: 27681.99/7909.17/3954.58) lat (ms,95%): 2.18 err/s: 0.00 reconn/s: 0.00
[ 80s ] thds: 4 tps: 1854.59 qps: 37092.42 (r/w/o: 25964.67/7418.56/3709.18) lat (ms,95%): 2.86 err/s: 0.00 reconn/s: 0.00
[ 90s ] thds: 4 tps: 1895.62 qps: 37912.74 (r/w/o: 26539.24/7582.27/3791.23) lat (ms,95%): 2.35 err/s: 0.00 reconn/s: 0.00
[ 100s ] thds: 4 tps: 1637.64 qps: 32753.68 (r/w/o: 22927.02/6551.38/3275.29) lat (ms,95%): 3.43 err/s: 0.00 reconn/s: 0.00
[ 110s ] thds: 4 tps: 1582.65 qps: 31651.17 (r/w/o: 22156.08/6329.79/3165.30) lat (ms,95%): 3.55 err/s: 0.00 reconn/s: 0.00
[ 120s ] thds: 4 tps: 646.70 qps: 12935.70 (r/w/o: 9055.23/2587.08/1293.39) lat (ms,95%): 3.30 err/s: 0.00 reconn/s: 0.00
[ 130s ] thds: 4 tps: 624.00 qps: 12479.93 (r/w/o: 8735.95/2495.99/1247.99) lat (ms,95%): 2.48 err/s: 0.00 reconn/s: 0.00
[ 140s ] thds: 4 tps: 1277.61 qps: 25549.69 (r/w/o: 17884.34/5110.04/2555.32) lat (ms,95%): 2.39 err/s: 0.00 reconn/s: 0.00
[ 150s ] thds: 4 tps: 1568.59 qps: 31372.43 (r/w/o: 21960.91/6274.35/3137.17) lat (ms,95%): 3.43 err/s: 0.00 reconn/s: 0.00
[ 160s ] thds: 4 tps: 1682.78 qps: 33656.41 (r/w/o: 23559.43/6731.42/3365.56) lat (ms,95%): 3.36 err/s: 0.00 reconn/s: 0.00
[ 170s ] thds: 4 tps: 1263.67 qps: 25272.89 (r/w/o: 17691.14/5054.40/2527.35) lat (ms,95%): 4.74 err/s: 0.00 reconn/s: 0.00
[ 180s ] thds: 4 tps: 1084.30 qps: 21684.57 (r/w/o: 15178.75/4337.21/2168.61) lat (ms,95%): 5.28 err/s: 0.00 reconn/s: 0.00
[ 190s ] thds: 4 tps: 1709.67 qps: 34192.90 (r/w/o: 23934.98/6838.68/3419.24) lat (ms,95%): 3.96 err/s: 0.00 reconn/s: 0.00
[ 200s ] thds: 4 tps: 1778.58 qps: 35574.88 (r/w/o: 24903.11/7114.52/3557.26) lat (ms,95%): 2.30 err/s: 0.00 reconn/s: 0.00
[ 210s ] thds: 4 tps: 1443.72 qps: 28872.03 (r/w/o: 20209.93/5774.67/2887.43) lat (ms,95%): 3.62 err/s: 0.00 reconn/s: 0.00
[ 220s ] thds: 4 tps: 1496.36 qps: 29928.18 (r/w/o: 20949.53/5985.84/2992.82) lat (ms,95%): 3.62 err/s: 0.00 reconn/s: 0.00
[ 230s ] thds: 4 tps: 1048.99 qps: 20979.05 (r/w/o: 14685.59/4195.57/2097.88) lat (ms,95%): 3.68 err/s: 0.00 reconn/s: 0.00
[ 240s ] thds: 4 tps: 1238.84 qps: 24777.16 (r/w/o: 17344.13/4955.35/2477.68) lat (ms,95%): 3.68 err/s: 0.00 reconn/s: 0.00
[ 250s ] thds: 4 tps: 1650.80 qps: 33017.65 (r/w/o: 23112.53/6603.51/3301.60) lat (ms,95%): 3.49 err/s: 0.00 reconn/s: 0.00
[ 260s ] thds: 4 tps: 1815.92 qps: 36316.93 (r/w/o: 25421.60/7263.49/3631.84) lat (ms,95%): 2.61 err/s: 0.00 reconn/s: 0.00
[ 270s ] thds: 4 tps: 1866.09 qps: 37322.90 (r/w/o: 26126.39/7464.34/3732.17) lat (ms,95%): 2.39 err/s: 0.00 reconn/s: 0.00
[ 280s ] thds: 4 tps: 1434.48 qps: 28688.75 (r/w/o: 20081.88/5737.91/2868.95) lat (ms,95%): 3.89 err/s: 0.00 reconn/s: 0.00
[ 290s ] thds: 4 tps: 1400.43 qps: 28009.65 (r/w/o: 19606.75/5602.03/2800.86) lat (ms,95%): 3.89 err/s: 0.00 reconn/s: 0.00
[ 300s ] thds: 4 tps: 1860.09 qps: 37199.93 (r/w/o: 26039.81/7439.85/3720.27) lat (ms,95%): 2.35 err/s: 0.00 reconn/s: 0.00
[ 310s ] thds: 4 tps: 1869.60 qps: 37393.62 (r/w/o: 26176.01/7478.40/3739.20) lat (ms,95%): 2.39 err/s: 0.00 reconn/s: 0.00
[ 320s ] thds: 4 tps: 1778.21 qps: 35562.05 (r/w/o: 24892.78/7112.85/3556.43) lat (ms,95%): 2.91 err/s: 0.00 reconn/s: 0.00
[ 330s ] thds: 4 tps: 1528.50 qps: 30570.62 (r/w/o: 21399.61/6114.00/3057.00) lat (ms,95%): 3.96 err/s: 0.00 reconn/s: 0.00
[ 340s ] thds: 4 tps: 1284.47 qps: 25691.18 (r/w/o: 17984.37/5137.88/2568.94) lat (ms,95%): 4.74 err/s: 0.00 reconn/s: 0.00
[ 350s ] thds: 4 tps: 960.29 qps: 19205.19 (r/w/o: 13443.45/3841.16/1920.58) lat (ms,95%): 5.88 err/s: 0.00 reconn/s: 0.00
[ 360s ] thds: 4 tps: 1114.63 qps: 22291.62 (r/w/o: 15603.83/4458.52/2229.26) lat (ms,95%): 5.67 err/s: 0.00 reconn/s: 0.00
[ 370s ] thds: 4 tps: 1452.69 qps: 29055.06 (r/w/o: 20338.90/5810.77/2905.39) lat (ms,95%): 3.82 err/s: 0.00 reconn/s: 0.00
[ 380s ] thds: 4 tps: 1395.32 qps: 27904.73 (r/w/o: 19532.83/5581.27/2790.63) lat (ms,95%): 3.96 err/s: 0.00 reconn/s: 0.00
[ 390s ] thds: 4 tps: 1418.00 qps: 28361.38 (r/w/o: 19853.39/5672.00/2836.00) lat (ms,95%): 4.33 err/s: 0.00 reconn/s: 0.00
[ 400s ] thds: 4 tps: 1692.34 qps: 33848.11 (r/w/o: 23693.50/6769.94/3384.67) lat (ms,95%): 3.36 err/s: 0.00 reconn/s: 0.00
[ 410s ] thds: 4 tps: 1226.29 qps: 24521.85 (r/w/o: 17164.70/4904.57/2452.59) lat (ms,95%): 4.18 err/s: 0.00 reconn/s: 0.00
[ 420s ] thds: 4 tps: 1273.05 qps: 25462.36 (r/w/o: 17823.77/5092.59/2546.00) lat (ms,95%): 3.62 err/s: 0.00 reconn/s: 0.00
[ 430s ] thds: 4 tps: 1861.50 qps: 37230.09 (r/w/o: 26061.39/7445.60/3723.10) lat (ms,95%): 2.35 err/s: 0.00 reconn/s: 0.00
[ 440s ] thds: 4 tps: 1777.00 qps: 35540.57 (r/w/o: 24878.55/7108.01/3554.01) lat (ms,95%): 3.02 err/s: 0.00 reconn/s: 0.00
[ 450s ] thds: 4 tps: 1856.80 qps: 37136.14 (r/w/o: 25995.06/7427.49/3713.59) lat (ms,95%): 2.39 err/s: 0.00 reconn/s: 0.00
[ 460s ] thds: 4 tps: 1551.05 qps: 31018.98 (r/w/o: 21712.96/6203.92/3102.11) lat (ms,95%): 3.62 err/s: 0.00 reconn/s: 0.00
[ 470s ] thds: 4 tps: 1341.04 qps: 26821.74 (r/w/o: 18775.52/5364.15/2682.07) lat (ms,95%): 3.89 err/s: 0.00 reconn/s: 0.00
[ 480s ] thds: 4 tps: 1881.71 qps: 37635.89 (r/w/o: 26345.53/7526.84/3763.52) lat (ms,95%): 2.35 err/s: 0.00 reconn/s: 0.00
[ 490s ] thds: 4 tps: 1867.16 qps: 37342.83 (r/w/o: 26139.96/7468.55/3734.32) lat (ms,95%): 2.35 err/s: 0.00 reconn/s: 0.00
[ 500s ] thds: 4 tps: 1546.63 qps: 30931.19 (r/w/o: 21651.38/6186.54/3093.27) lat (ms,95%): 3.75 err/s: 0.00 reconn/s: 0.00
[ 510s ] thds: 4 tps: 1409.60 qps: 28193.26 (r/w/o: 19735.44/5638.61/2819.21) lat (ms,95%): 3.96 err/s: 0.00 reconn/s: 0.00
[ 520s ] thds: 4 tps: 1307.69 qps: 26153.83 (r/w/o: 18307.91/5230.55/2615.37) lat (ms,95%): 4.41 err/s: 0.00 reconn/s: 0.00
[ 530s ] thds: 4 tps: 1168.27 qps: 23364.08 (r/w/o: 16354.17/4673.38/2336.54) lat (ms,95%): 4.65 err/s: 0.00 reconn/s: 0.00
[ 540s ] thds: 4 tps: 1251.93 qps: 25040.24 (r/w/o: 17528.98/5007.41/2503.85) lat (ms,95%): 3.89 err/s: 0.00 reconn/s: 0.00
[ 550s ] thds: 4 tps: 1489.00 qps: 29780.38 (r/w/o: 20846.26/5956.12/2978.01) lat (ms,95%): 3.82 err/s: 0.00 reconn/s: 0.00
[ 560s ] thds: 4 tps: 1377.81 qps: 27554.15 (r/w/o: 19287.41/5511.13/2755.62) lat (ms,95%): 4.03 err/s: 0.00 reconn/s: 0.00
[ 570s ] thds: 4 tps: 1535.50 qps: 30711.88 (r/w/o: 21498.89/6142.00/3071.00) lat (ms,95%): 4.18 err/s: 0.00 reconn/s: 0.00
[ 580s ] thds: 4 tps: 1456.25 qps: 29126.18 (r/w/o: 20387.88/5825.80/2912.50) lat (ms,95%): 3.96 err/s: 0.00 reconn/s: 0.00
[ 590s ] thds: 4 tps: 1134.91 qps: 22696.84 (r/w/o: 15888.20/4538.83/2269.81) lat (ms,95%): 4.74 err/s: 0.00 reconn/s: 0.00
[ 600s ] thds: 4 tps: 1250.61 qps: 25010.09 (r/w/o: 17506.41/5002.46/2501.23) lat (ms,95%): 5.09 err/s: 0.00 reconn/s: 0.00
SQL statistics:
    queries performed:
        read:                            12221608
        write:                           3491885
        other:                           1745947
        total:                           17459440
    transactions:                        872972 (1454.94 per sec.)
    queries:                             17459440 (29098.72 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          600.0052s
    total number of events:              872972

Latency (ms):
         min:                                    1.14
         avg:                                    2.75
         max:                                 1650.86
         95th percentile:                        3.89
         sum:                              2398246.47

Threads fairness:
    events (avg/stddev):           218243.0000/495.71
    execution time (avg/stddev):   599.5616/0.00

```
Итоги тестирования:

Ключевые метрики:
- Transactions Per Second (TPS): 1454.94
- Queries Per Second (QPS): 29098.72
- Average Latency: 2.75 ms
- Maximum Latency: 1650.86 ms
- 95th Percentile Latency: 3.89 ms

Заметное увеличение TPS и QPS по сравнению с предыдущими сценариями, показывающее значительное улучшение в обработке одновременных операций.\
Средняя задержка умеренно низкая, 2.75 мс, хотя заметен резкий скачок максимальной задержки, что может указывать на периодические пики спроса или стресс системы.\
Постоянно низкая задержка 95-го процентиля предполагает, что система эффективно справляется с большинством операций в настройках "Aggressive Performance".

Сценарий Aggressive Performance:
- TPS: 1454.94
- Средняя задержка: 2.75 мс

Сценарий High Concurrency:
- TPS: 403.08
- Средняя задержка: 10.488 мс

Сценарий Minimal:
- TPS: 5218.492266 (pgbench)
- Средняя задержка: 1.916 мс

***Анализ:***
1. Эффективность и скорость:
Настройки "Aggressive Performance" показывают эффективный баланс между обработкой высоких нагрузок и поддержанием низкой задержки, значительно превосходя сценарий "High Concurrency".\
Хотя они не достигают минимальной задержки, наблюдаемой в сценарии "Minimal", они обеспечивают гораздо более надежную пропускную способность, особенно в среде sysbench, которая симулирует сложные операции чтения/записи.
2. Использование ресурсов:
Настройки "Aggressive Performance" кажутся более эффективными в использовании доступных ресурсов (таких как ЦП, память), особенно с увеличением shared_buffers, work_mem и корректировкой checkpoint_timeout и max_wal_size. Это предполагает лучшее управление большими объемами данных и более интенсивными операциями ввода-вывода.
3. Прочее:
Отключение synchronous_commit и увеличение checkpoint_timeout в сценарии "Aggressive Performance" может ввести более высокий риск потери данных в случае сбоя, но обеспечивает более высокий TPS и более низкие задержки во время нормальной работы. Этот компромисс критичен для приложений, где производительность является ключевой и может терпеть небольшое увеличение риска потери данных.



---

# Итоговый вывод
При сравнении всех трех сценариев:
- Сценарий Minimal: Лучше всего подходит для сред, где критична минимальная задержка, подходит для менее сложных запросов, где база данных не подвергается сильной нагрузке от множественных одновременных соединений.
- Сценарий High Concurrency: Лучше для сред с высоким числом одновременных соединений, но показывает значительное снижение производительности, указывая на потенциальные узкие места под нагрузкой.
- Сценарий Aggressive Performance: Балансирует между высокой производительностью и умеренным риском, делая его подходящим для высоких требований к производительности, где приемлемы случайные пики высокой задержки.

