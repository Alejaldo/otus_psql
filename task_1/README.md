# Задание №1

## 1. Создать новый проект в Google Cloud Platform, Яндекс облако или на любых ВМ, докер, далее создать инстанс виртуальной машины с дефолтными параметрами и добавить свой ssh ключ в metadata ВМ
***Создана виртуальная машина (ВМ) в Yandex Cloud (4 GB RAM, 15 GB HDD, vCPU 2, Ubuntu 22.04 LTS) с привязкой ssh локальной машины***
## 2. Зайти удаленным ssh (первая сессия), не забывайте про ssh-add
***На моей локальной машине уже есть ssh, без использования passphrase, поэтому не требуется использования агента и ssh-add***
```
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_rsa
```
***Захожу на ВМ через команду в первом открытом терминале локальной машины***
```
ssh -i ~/.ssh/id_rsa chamonix@158.160.136.77
```
## 3. Поставить PostgreSQL
***Устанавливаю PostgreSQL через команду***
```
sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql && sudo apt install unzip && sudo apt -y install mc
```
## 4. Зайти вторым ssh (вторая сессия)
***Захожу на ВМ через команду во втором открытом терминале локальной машины***
```
ssh -i ~/.ssh/id_rsa chamonix@158.160.136.77
```
## 5. Запустить везде psql из под пользователя postgres
***Исполняю команду `sudo -u postgres psql` в обоих терминалах:***
```
chamonix@chamonix:~$ sudo -u postgres psql
psql (16.1 (Ubuntu 16.1-1.pgdg22.04+1))
Type "help" for help.

postgres=#
```
## 6. Выключить auto commit
***Для отключения начинаю вход в транзакции в обоих терминалах через команду `BEGIN;` после нее auto-commit отключен ([согласно документации](https://www.postgresql.org/docs/current/sql-begin.html)):***
```
postgres=# BEGIN;
BEGIN
```
## 7. Сделать в первой сессии новую таблицу и наполнить ее данными create table persons(id serial, first_name text, second_name text); insert into persons(first_name, second_name) values('ivan', 'ivanov'); insert into persons(first_name, second_name) values('petr', 'petrov'); commit;

***Исполняю в первом терминале последовательно команды***
```
CREATE TABLE persons (
    id SERIAL,
    first_name TEXT,
    second_name TEXT
);

INSERT INTO persons (first_name, second_name) VALUES ('ivan', 'ivanov');
INSERT INTO persons (first_name, second_name) VALUES ('petr', 'petrov');
COMMIT;
```
***Проверяю что все сохранилось***
```
postgres=# SELECT * FROM persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)

```

## 8. Посмотреть текущий уровень изоляции: show transaction isolation level
***Также в первом терминале исполняю комманду `SHOW TRANSACTION ISOLATION LEVEL;` которая выдает результат:***
```
postgres=# SHOW TRANSACTION ISOLATION LEVEL;
 transaction_isolation
-----------------------
 read committed
(1 row)

```
***"Read Committed" - стандартный уровень изоляции используемый в PostgreSQL***
## 9. Начать новую транзакцию в обоих сессиях с дефолтным (не меняя) уровнем изоляции
***Начинаю транзации в обоих окнах терминала с команды `BEGIN;`***
```
postgres=# BEGIN;
BEGIN

```
## 10. В первой сессии добавить новую запись insert into persons(first_name, second_name) values('sergey', 'sergeev');
***Затем исполняю добавление новой записи в таблицу в первом терминале/сессии:***
```
postgres=*# INSERT INTO persons (first_name, second_name) VALUES ('sergey', 'sergeev');
INSERT 0 1
```
## 11. сделать select from persons во второй сессии
***Исполняю команду просмотра записей в таблице из второго терминала:***
```
postgres=*# SELECT * FROM persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
```
## 12. Видите ли вы новую запись и если да то почему?
***Новую запись про добавление sergey sergeev не вижу во второй сессии, поскольку не исполнен коммит данной транзакции в первой серии (транзакция не была завершена). На уровне изоляции 'Read Committed' транзакция видит только данные, которые были зафиксированы до ее начала. Поскольку транзакция в первой сессии все еще открыта (не завершена), ее изменения не видны в других транзакциях, включая транзакцию во второй сессии.***
## 13. Завершить первую транзакцию - commit;
***Исполняю в первой сессии (первом терминале) команду COMMIT;***
```
postgres=*# COMMIT;
COMMIT
```
## 14. Сделать select from persons во второй сессии
***Во второй сессии проверяю записи в таблице:***
```
postgres=*# SELECT * FROM persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```
## 15. Видите ли вы новую запись и если да то почему?
***Теперь новая запись про добавление sergey sergeev отображается во второй сессии. Эта видимость обусловлена уровнем изоляции 'Read Committed'. Как только транзакция завершена, все ее изменения становятся видимыми для всех других транзакций, которые начинаются после этого коммита.***
## 16. Завершите транзакцию во второй сессии
***Во второй сессии исполняю команду `COMMIT;` которая формально завершает транзацию во второй сессии (не было ничего изменено в таблице, но поскольку транзакция была начата, ее нужно завершить)***
```
postgres=*# COMMIT;
COMMIT
```
## 17. Начать новые но уже repeatable read транзации - set transaction isolation level repeatable read;
***В каждой сессии начинаю новую транзакцию с уровнем изоляции REPEATABLE READ***
***Первая сессия:***
```
postgres=# BEGIN;
BEGIN
postgres=*# SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SET
postgres=*# SHOW TRANSACTION ISOLATION LEVEL;
 transaction_isolation
-----------------------
 repeatable read
(1 row)

```
***Вторая сессия:***
```
postgres=# BEGIN;
BEGIN
postgres=*# SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SET
postgres=*# SHOW TRANSACTION ISOLATION LEVEL;
 transaction_isolation
-----------------------
 repeatable read
(1 row)

```
## 18. В первой сессии добавить новую запись insert into persons(first_name, second_name) values('sveta', 'svetova');
***Делаю добавление новой записи в первой сессии:***
```
postgres=*# INSERT INTO persons (first_name, second_name) VALUES ('sveta', 'svetova');
INSERT 0 1
```
## 19. сделать select* from persons во второй сессии
***Проверяю добавилась ли новая запись через терминал второй сессии:***
```
postgres=*# SELECT * FROM persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```
## 20. Видите ли вы новую запись и если да то почему?
***Во второй сессии не отображена новая запись, поскольку эта сессия не должна видеть новую запись, добавленную в первой сессии, до завершения транзакции в этой сессии. На уровне изоляции REPEATABLE READ текущая транзакция не видит изменений, сделанных другими транзакциями после начала текущей транзакции. Поэтому, если во второй сессии запрос SELECT выполнен до завершения транзакции в первой сессии, новая запись ('sveta', 'svetova') не будет видна.***
## 21. завершить первую транзакцию - commit;
***В первой сессии завершаю транзакцию***
```
postgres=*# COMMIT;
COMMIT
```
## 22. Сделать select from persons во второй сессии
***Проверяю записи в таблице во второй сессии:***
```
postgres=*# SELECT * FROM persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)

```
## 23. Видите ли вы новую запись и если да то почему?
***Из терминала второй сессии новая запись не отображается, это происходит из-за уровня изоляции REPEATABLE READ, при котором транзакция видит данные в том состоянии, в котором они были в момент ее начала. Поскольку транзакция во второй сессии началась до фиксации изменений в первой сессии, новые данные не видны.***
## 24. завершить вторую транзакцию
***Завершаю вторую транзакцию***
```
postgres=*# COMMIT;
COMMIT
```
## 25. Сделать select * from persons во второй сессии
***Проверяю теперь записи в таблице во второй сесии:***
```
postgres=# SELECT * FROM persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
(4 rows)
```
## 26. Видите ли вы новую запись и если да то почему?
***И вижу теперь новую запись, поскольку на уровне изоляции REPEATABLE READ, новые данные становятся видимыми только после завершения текущей транзакции. Так как теперь транзакция во второй сессии завершена, все изменения, включая добавленные в первой сессии, видны.***
