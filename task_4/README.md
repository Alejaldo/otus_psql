# Задание №4

## 1. Создайте новый кластер PostgresSQL 14
***В своем случае будут использовать локально установленный docker и PSQL версии 15:***
```
[~]$ psql --version
psql (PostgreSQL) 15.3 (Ubuntu 15.3-1.pgdg18.04+1)
[~]$ docker network create kntxt-psql-network
10bea201521afb397efe855712717b68904f945c1c3e0867e6175f8e08bc8266
```
## 2. Зайдите в созданный кластер под пользователем postgres
***Захожу в кластер под юзером postgres:***
```
[~]$ docker run -d --name pg-cluster --network kntxt-psql-network -e POSTGRES_PASSWORD=cdewitte -p 5477:5432 postgres:15
bcf6240d7a5f7c35567c16be146de8f09fac93476a729a4fe753adee20ae6b51
[~]$ docker exec -it pg-cluster psql -U postgres
psql (15.6 (Debian 15.6-1.pgdg120+2))
Type "help" for help.

postgres=#
```
## 3. Создайте новую базу данных testdb
***Создаю новую  базу данных testdb:***
```
postgres=# CREATE DATABASE testdb;
CREATE DATABASE
```
## 4. Зайдите в созданную базу данных под пользователем postgres
***Захожу в базу данных testdb:***
```
postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
testdb=#
```
## 5. Создайте новую схему testnm
***Создаю схему testnm:***
```
testdb=# CREATE SCHEMA testnm;
CREATE SCHEMA
```
## 6. Создайте новую таблицу t1 с одной колонкой c1 типа integer
***Создаю таблицу t1 с одной колонкой c1 типа integer:***
```
testdb=# CREATE TABLE testnm.t1 (c1 integer);
CREATE TABLE
```
## 7. Вставьте строку со значением c1=1
***Добавляю запись:***
```
testdb=# INSERT INTO testnm.t1 VALUES (1);
INSERT 0 1
```
## 8. Создайте новую роль readonly
***Создаю новую роль:***
```
testdb=# CREATE ROLE readonly;
CREATE ROLE
```
## 9. Дайте новой роли право на подключение к базе данных testdb
***Даю для роли readonly право на подключение к базе данных testdb:***
```
testdb=# GRANT CONNECT ON DATABASE testdb TO readonly;
GRANT
```
## 10. Дайте новой роли право на использование схемы testnm
***Даю для роли readonly право на использование схемы testnm:***
```
testdb=# GRANT USAGE ON SCHEMA testnm TO readonly;
GRANT
```
## 11. Дайте новой роли право на select для всех таблиц схемы testnm
***Даю для роли readonly право на SELECT для всех таблиц схемы testnm:***
```
testdb=# ALTER DEFAULT PRIVILEGES IN SCHEMA testnm GRANT SELECT ON TABLES TO readonly;
ALTER DEFAULT PRIVILEGES
testdb=# GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;
GRANT
```
## 12. Создайте пользователя testread с паролем test123
***Создаю пользователя testread с паролем test123:***
```
testdb=# CREATE USER testread WITH PASSWORD 'test123';
CREATE ROLE
```
## 13. Дайте роль readonly пользователю testread
***Даю роль readonly пользователю testread:***
```
testdb=# GRANT readonly TO testread;
GRANT ROLE
```
## 14. Зайдите под пользователем testread в базу данных testdb
***Выхожу из текущей сессии:***
```
testdb=# \q
[~]$
```
***Захожу под юзером testread***
```
[~]$ docker exec -it pg-cluster psql -U testread -d testdb
psql (15.6 (Debian 15.6-1.pgdg120+2))
Type "help" for help.

testdb=>
```
## 15. Сделайте select * from t1;
***Совершаю команду `SELECT * FROM testnm.t1;`:***
```
testdb=> SELECT * FROM testnm.t1;
 c1
----
  1
(1 row)

testdb=>
```
***Команда отработала без ошибок***
## 16. Получилось? (могло если вы делали сами не по шпаргалке и не упустили один существенный момент про который позже)
## 17. Напишите что именно произошло в тексте домашнего задания;
## 18. У вас есть идеи почему? ведь права то дали?
## 19. Посмотрите на список таблиц
## 20. Подсказка в шпаргалке под пунктом 20
## 21. А почему так получилось с таблицей (если делали сами и без шпаргалки то может у вас все нормально)
## 22. Вернитесь в базу данных testdb под пользователем postgres
## 23. Удалите таблицу t1
## 24. Создайте ее заново но уже с явным указанием имени схемы testnm
## 25. Вставьте строку со значением c1=1
## 26. Зайдите под пользователем testread в базу данных testdb
## 27. Сделайте select * from testnm.t1;
## 28. Получилось?
## 29. Есть идеи почему? если нет - смотрите шпаргалку
## 30. Как сделать так чтобы такое больше не повторялось? если нет идей - смотрите шпаргалку
## 31. Сделайте select * from testnm.t1;
## 32. Получилось?
## 33. Есть идеи почему? если нет - смотрите шпаргалку
## 34. Сделайте select * from testnm.t1;
## 35. Получилось?
## 36. Ура!
***Пояснения по пунктам 16 - 36:***
***Проблема с предоставлением прав может возникать действительно в случаях не явного указания схем.***
***Стандартная схема: По умолчанию PostgreSQL использует схему public. Если вы создаете таблицу без указания схемы, она будет создана в public, а не в testnm, что может привести к проблемам с правами доступа, если роль readonly не имеет доступа к ней.***
***Права роли: Роль readonly должна иметь правильные привилегии не только для доступа к базе данных и использования схемы, но и для выборки из таблиц внутри этой схемы. Если эти права не установлены корректно, пользователь testread не сможет выполнить команду SELECT как предполагалось.***
## 37. Теперь попробуйте выполнить команду create table t2(c1 integer); insert into t2 values (2);
## 38. А как так? нам же никто прав на создание таблиц и insert в них под ролью readonly?
## 39. Есть идеи как убрать эти права? если нет - смотрите шпаргалку
## 40. Если вы справились сами то расскажите что сделали и почему, если смотрели шпаргалку - объясните что сделали и почему выполнив указанные в ней команды
***Пояснения для шагов 37-40:***
***Пробую сначала исполнить команды***
```
CREATE TABLE t2(c1 integer);
INSERT INTO t2 VALUES (2);
```
***под юзером testread:***
```
testdb=> CREATE TABLE t2(c1 integer);
ERROR:  permission denied for schema public
LINE 1: CREATE TABLE t2(c1 integer);
                     ^
testdb=> CREATE TABLE testnm.t2(c1 integer);
ERROR:  permission denied for schema testnm
LINE 1: CREATE TABLE testnm.t2(c1 integer);

```
***Это дает ошибку прав как с указанием явно схемы testnm так и без, поскольку для юзера testread ранее не давались права кроме как чтения.***
***Теперь я выхожу из текущей сессии, чтобы зайти снова под юзером postgres, чтобы иметь возможность создавать таблицы и вставлять данные:***
```
testdb=> \q
[~]$
[~]$ docker exec -it pg-cluster psql -U postgres -d testdb
psql (15.6 (Debian 15.6-1.pgdg120+2))
Type "help" for help.

testdb=# CREATE TABLE t2(c1 integer);
CREATE TABLE
testdb=# INSERT INTO t2 VALUES (2);
INSERT 0 1
testdb=# SELECT * FROM t2;
 c1
----
  2
(1 row)
```
***Под юзером postgres у которого есть все права получилось создать таблицу t2 и запись в ней.***
***При этом, таблица t2 создалась в рамках схемы public покольку никакую другую схему я явно не указывал, а public пока является дефолтной:***
```
testdb=# SELECT schemaname, tablename FROM pg_tables WHERE tablename = 't2';
 schemaname | tablename
------------+-----------
 public     | t2
(1 row)

```
## 41. Теперь попробуйте выполнить команду create table t3(c1 integer); insert into t2 values (2);
## 42. Расскажите что получилось и почему
***Теперь исполню создание таблицы t3, но для интереса укажу для нее схему testnm***
```
testdb=# CREATE TABLE testnm.t3(c1 integer);
CREATE TABLE
testdb=# INSERT INTO t2 VALUES (2);
INSERT 0 1
testdb=# SELECT * FROM t2;
 c1
----
  2
  2
(2 rows)

testdb=# INSERT INTO testnm.t3 VALUES (2);
INSERT 0 1
testdb=# SELECT * FROM t3;
ERROR:  relation "t3" does not exist
LINE 1: SELECT * FROM t3;
                      ^
testdb=# SELECT * FROM testnm.t3;
 c1
----
  2
(1 row)

testdb=#
```
***В итоги, я не смог прочитать данные в таблице t3 без явного указания наименования схемы testnm, с указанием же успешно создалась таблица, добавлена запись в нее (как видно в таблицу t2 втора запись была добавлена без указания наименования схемы  public)***
