# Задание №14 "Триггеры, поддержка заполнения витрин"

# Развернутое описание задачи:
```
-- ДЗ тема: триггеры, поддержка заполнения витрин

DROP SCHEMA IF EXISTS pract_functions CASCADE;
CREATE SCHEMA pract_functions;

SET search_path = pract_functions, publ

-- товары:
CREATE TABLE goods
(
    goods_id    integer PRIMARY KEY,
    good_name   varchar(63) NOT NULL,
    good_price  numeric(12, 2) NOT NULL CHECK (good_price > 0.0)
);
INSERT INTO goods (goods_id, good_name, good_price)
VALUES  (1, 'Спички хозайственные', .50),
    (2, 'Автомобиль Ferrari FXX K', 185000000.01);

-- Продажи
CREATE TABLE sales
(
    sales_id    integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    good_id     integer REFERENCES goods (goods_id),
    sales_time  timestamp with time zone DEFAULT now(),
    sales_qty   integer CHECK (sales_qty > 0)
);

INSERT INTO sales (good_id, sales_qty) VALUES (1, 10), (1, 1), (1, 120), (2, 1);

-- отчет:
SELECT G.good_name, sum(G.good_price * S.sales_qty)
FROM goods G
INNER JOIN sales S ON S.good_id = G.goods_id
GROUP BY G.good_name;

-- с увеличением объёма данных отчет стал создаваться медленно
-- Принято решение денормализовать БД, создать таблицу
CREATE TABLE good_sum_mart
(
  good_name   varchar(63) NOT NULL,
  sum_sale  numeric(16, 2)NOT NULL
);

-- Создать триггер (на таблице sales) для поддержки.
-- Подсказка: не забыть, что кроме INSERT есть еще UPDATE и DELETE

-- Чем такая схема (витрина+триггер) предпочтительнее отчета, создаваемого "по требованию" (кроме производительности)?
-- Подсказка: В реальной жизни возможны изменения цен.
```

В БД создана структура, описывающая товары (таблица goods) и продажи (таблица sales).

Есть запрос для генерации отчета – сумма продаж по каждому товару.

БД была денормализована, создана таблица (витрина), структура которой повторяет структуру отчета.

Создать триггер на таблице продаж, для поддержки данных в витрине в актуальном состоянии (вычисляющий при каждой продаже сумму и записывающий её в витрину)

Подсказка: не забыть, что кроме INSERT есть еще UPDATE и DELETE

Задание со звездочкой*
Чем такая схема (витрина+триггер) предпочтительнее отчета, создаваемого "по требованию" (кроме производительности)?
Подсказка: В реальной жизни возможны изменения цен.

# План выполнения
Для решения задачи необходимо выполнить следующие шаги:

1. Создать схему и таблицы.
2. Создание витрины и заполнение начальными данными
3. Создагние функций для обработки вставки данных, для обработки обновления данных, для обработки удаления данных
4. Создание триггеров
5. Проверить корректность работы триггера.
6. Объяснить преимущество использования витрины с триггерами ("Чем такая схема (витрина+триггер) предпочтительнее отчета, создаваемого "по требованию" (кроме производительности)? Подсказка: В реальной жизни возможны изменения цен.")

## 1. Создать схему и таблицы.
Буду работать локально со своим докер контейнером pg-cluster где создам базу данных testshopdb:
```
[~]$ docker start pg-cluster
pg-cluster
[~]$ docker exec -it pg-cluster psql -U postgres -c "CREATE DATABASE testshopdb;"
CREATE DATABASE
[~]$ docker exec -it pg-cluster psql -U postgres -d testshopdb
psql (15.6 (Debian 15.6-1.pgdg120+2))
Type "help" for help.

testshopdb=#

```
Теперь создаю схему и таблицы:
```
testshopdb=# DROP SCHEMA IF EXISTS pract_functions CASCADE;
NOTICE:  schema "pract_functions" does not exist, skipping
DROP SCHEMA
testshopdb=# CREATE SCHEMA pract_functions;
CREATE SCHEMA
testshopdb=# SET search_path = pract_functions, public;
SET
testshopdb=# CREATE TABLE goods
(
    goods_id    integer PRIMARY KEY,
    good_name   varchar(63) NOT NULL,
    good_price  numeric(12, 2) NOT NULL CHECK (good_price > 0.0)
);
CREATE TABLE
testshopdb=# INSERT INTO goods (goods_id, good_name, good_price)
VALUES  (1, 'Спички хозайственные', 0.50),
        (2, 'Автомобиль Ferrari FXX K', 185000000.01);
INSERT 0 2
testshopdb=# CREATE TABLE sales
(
    sales_id    integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    good_id     integer REFERENCES goods (goods_id),
    sales_time  timestamp with time zone DEFAULT now(),
    sales_qty   integer CHECK (sales_qty > 0)
);
CREATE TABLE
testshopdb=# INSERT INTO sales (good_id, sales_qty) VALUES (1, 10), (1, 1), (1, 120), (2, 1);
INSERT 0 4

```

## 2. Создание витрины и заполнение начальными данными
```
testshopdb=# CREATE TABLE good_sum_mart
(
  good_name   varchar(63) NOT NULL,
  sum_sale    numeric(16, 2) NOT NULL
);
CREATE TABLE
testshopdb=# INSERT INTO good_sum_mart (good_name, sum_sale)
SELECT G.good_name, SUM(G.good_price * S.sales_qty)
FROM goods G
INNER JOIN sales S ON S.good_id = G.goods_id
GROUP BY G.good_name;
INSERT 0 2

```

## 3. Создание триггеров и функций для обновления витрины

Функция для обработки вставки данных:
```
testshopdb=# CREATE OR REPLACE FUNCTION update_good_sum_mart_on_insert()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO good_sum_mart (good_name, sum_sale)
    VALUES (
        (SELECT good_name FROM goods WHERE goods_id = NEW.good_id),
        (SELECT good_price FROM goods WHERE goods_id = NEW.good_id) * NEW.sales_qty
    )
    ON CONFLICT (good_name) DO UPDATE SET
        sum_sale = good_sum_mart.sum_sale + EXCLUDED.sum_sale;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
CREATE FUNCTION

```
где:
- `RETURNS TRIGGER`: Указывает, что функция возвращает триггер.
- `NEW`: Содержит значения новых данных, вставленных в таблицу.
- `ON CONFLICT`: Указывает, что делать в случае конфликта, например, при попытке вставить запись с уже существующим значением good_name.

Функция для обработки обновления данных:
```
testshopdb=# CREATE OR REPLACE FUNCTION update_good_sum_mart_on_update()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE good_sum_mart
    SET sum_sale = sum_sale - (SELECT good_price FROM goods WHERE goods_id = OLD.good_id) * OLD.sales_qty
    WHERE good_name = (SELECT good_name FROM goods WHERE goods_id = OLD.good_id);

    INSERT INTO good_sum_mart (good_name, sum_sale)
    VALUES (
        (SELECT good_name FROM goods WHERE goods_id = NEW.good_id),
        (SELECT good_price FROM goods WHERE goods_id = NEW.good_id) * NEW.sales_qty
    )
    ON CONFLICT (good_name) DO UPDATE SET
        sum_sale = good_sum_mart.sum_sale + EXCLUDED.sum_sale;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
CREATE FUNCTION
```
- `OLD`: Содержит значения старых данных, которые были в таблице до обновления.
- Функция сначала уменьшает сумму продаж на значение, соответствующее старым данным, а затем увеличивает на значение, соответствующее новым данным.

Функция для обработки удаления данных:
```
testshopdb=# CREATE OR REPLACE FUNCTION update_good_sum_mart_on_delete()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE good_sum_mart
    SET sum_sale = sum_sale - (SELECT good_price FROM goods WHERE goods_id = OLD.good_id) * OLD.sales_qty
    WHERE good_name = (SELECT good_name FROM goods WHERE goods_id = OLD.good_id);

    DELETE FROM good_sum_mart WHERE sum_sale <= 0;
    RETURN OLD;
END;
$$ LANGUAGE plpgsql;
CREATE FUNCTION

```
Эта функция уменьшает сумму продаж на значение удаленных данных и удаляет запись из витрины, если сумма продаж становится меньше или равна нулю.

## 4. Создание триггеров
Триггеры вызывают функции при определенных событиях (вставка, обновление, удаление) в таблице sales.
```
testshopdb=# CREATE TRIGGER trg_insert_sales
AFTER INSERT ON sales
FOR EACH ROW
EXECUTE FUNCTION update_good_sum_mart_on_insert();
CREATE TRIGGER
testshopdb=# CREATE TRIGGER trg_update_sales
AFTER UPDATE ON sales
FOR EACH ROW
EXECUTE FUNCTION update_good_sum_mart_on_update();
CREATE TRIGGER
testshopdb=# CREATE TRIGGER trg_delete_sales
AFTER DELETE ON sales
FOR EACH ROW
EXECUTE FUNCTION update_good_sum_mart_on_delete();
CREATE TRIGGER

```
- `AFTER` `INSERT`/`UPDATE`/`DELETE`: Указывает, когда должен быть вызван триггер (после вставки, обновления или удаления).
- `FOR EACH ROW`: Указывает, что триггер должен срабатывать для каждой строки, измененной в таблице.


## 5. Проверка работы триггеров
Вставка новой продажи:

```
testshopdb=# INSERT INTO sales (good_id, sales_qty) VALUES (1, 50);
ERROR:  there is no unique or exclusion constraint matching the ON CONFLICT specification
CONTEXT:  SQL statement "INSERT INTO good_sum_mart (good_name, sum_sale)
    VALUES (
        (SELECT good_name FROM goods WHERE goods_id = NEW.good_id),
        (SELECT good_price FROM goods WHERE goods_id = NEW.good_id) * NEW.sales_qty
    )
    ON CONFLICT (good_name) DO UPDATE SET
        sum_sale = good_sum_mart.sum_sale + EXCLUDED.sum_sale"
PL/pgSQL function update_good_sum_mart_on_insert() line 3 at SQL statement
testshopdb=# UPDATE sales SET sales_qty = 60 WHERE sales_id = 1;

```

Вижу ошибку, связанную с отсутствием уникального или исключающего ограничения на таблицу good_sum_mart, что необходимо для использования конструкции ON CONFLICT. Исправлю это, добавив уникальный индекс на good_name:
```
testshopdb=# CREATE UNIQUE INDEX idx_good_sum_mart_good_name ON good_sum_mart (good_name);
CREATE INDEX

```
И делаю пересоздание функций:
```
testshopdb=# CREATE OR REPLACE FUNCTION update_good_sum_mart_on_insert()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO good_sum_mart (good_name, sum_sale)
    VALUES (
        (SELECT good_name FROM goods WHERE goods_id = NEW.good_id),
        (SELECT good_price FROM goods WHERE goods_id = NEW.good_id) * NEW.sales_qty
    )
    ON CONFLICT (good_name) DO UPDATE SET
        sum_sale = good_sum_mart.sum_sale + EXCLUDED.sum_sale;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
CREATE FUNCTION
testshopdb=# CREATE OR REPLACE FUNCTION update_good_sum_mart_on_update()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE good_sum_mart
    SET sum_sale = sum_sale - (SELECT good_price FROM goods WHERE goods_id = OLD.good_id) * OLD.sales_qty
    WHERE good_name = (SELECT good_name FROM goods WHERE goods_id = OLD.good_id);

    INSERT INTO good_sum_mart (good_name, sum_sale)
    VALUES (
        (SELECT good_name FROM goods WHERE goods_id = NEW.good_id),
        (SELECT good_price FROM goods WHERE goods_id = NEW.good_id) * NEW.sales_qty
    )
    ON CONFLICT (good_name) DO UPDATE SET
        sum_sale = good_sum_mart.sum_sale + EXCLUDED.sum_sale;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
CREATE FUNCTION
testshopdb=# CREATE OR REPLACE FUNCTION update_good_sum_mart_on_delete()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE good_sum_mart
    SET sum_sale = sum_sale - (SELECT good_price FROM goods WHERE goods_id = OLD.good_id) * OLD.sales_qty
    WHERE good_name = (SELECT good_name FROM goods WHERE goods_id = OLD.good_id);

    DELETE FROM good_sum_mart WHERE sum_sale <= 0;
    RETURN OLD;
END;
$$ LANGUAGE plpgsql;
CREATE FUNCTION

```
Проверяю снова:
```
testshopdb=# INSERT INTO sales (good_id, sales_qty) VALUES (1, 50);
INSERT 0 1
```
Успешно, продолжу обновлением существующей продажи:
```
testshopdb=# UPDATE sales SET sales_qty = 60 WHERE sales_id = 1;
UPDATE 1

```
Теперь Удаление продажи:
```
testshopdb=# DELETE FROM sales WHERE sales_id = 2;
DELETE 1

```

Проверка содержимого витрины:
```
testshopdb=# SELECT * FROM good_sum_mart;
        good_name         |   sum_sale
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |       115.00
(2 rows)

```

## 6. Преимущества использования витрины с триггерами
Чем такая схема (витрина + триггер) предпочтительнее отчета, создаваемого "по требованию", кроме производительности?

Преимущества:
1. Актуальность данных:
- Витрина: Данные в витрине обновляются автоматически и мгновенно при каждой операции (INSERT, UPDATE, DELETE), что обеспечивает актуальность данных.
- Отчет по требованию: Данные могут быть неактуальными между запусками отчетов, что может привести к неточным результатам.
2. Уменьшение нагрузки на базу данных:
- Витрина: Обновление витрины происходит постепенно, с каждой транзакцией, что распределяет нагрузку на базу данных.
- Отчет по требованию: Запуск отчетов "по требованию" может вызвать значительную нагрузку на базу данных, особенно если объем данных велик.
3. Поддержка изменений цен:
- Витрина: При изменении цены товара витрина может быть обновлена автоматически с помощью триггеров.
- Отчет по требованию: Для учета изменений цен необходимо запускать отчеты заново, что может быть ресурсоемким процессом.
