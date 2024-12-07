# Домашнее задание
## Триггеры, поддержка заполнения витрин

### Цель:
Создать триггер для поддержки витрины в актуальном состоянии.

### Описание/Пошаговая инструкция выполнения домашнего задания:
* Скрипт и развернутое описание задачи – в ЛК (файл hw_triggers.sql) или по ссылке: https://disk.yandex.ru/d/l70AvknAepIJXQ
* В **БД** создана структура, описывающая товары (таблица **goods**) и продажи (таблица **sales**).
* Есть запрос для генерации отчета – сумма продаж по каждому товару.
* **БД** была денормализована, создана таблица (витрина), структура которой повторяет структуру отчета.
* Создать триггер на таблице продаж, для поддержки данных в витрине в актуальном состоянии (вычисляющий при каждой продаже сумму и записывающий её в витрину)
* **Подсказка:** не забыть, что кроме **INSERT** есть еще **UPDATE** и **DELETE**
---
**Задание со звездочкой***

_Чем такая схема (витрина+триггер) предпочтительнее отчета, создаваемого "по требованию" (кроме производительности)?
Подсказка: В реальной жизни возможны изменения цен._

**Скачаем скрипт** ["hw_triggers.sql"](https://disk.yandex.ru/d/l70AvknAepIJXQ)

** Создадим БД:
```postgresql
postgres=# CREATE DATABASE hw_triggers;
CREATE DATABASE
postgres=# \l
                                                    List of databases
    Name     |  Owner   | Encoding | Locale Provider | Collate |  Ctype  | ICU Locale | ICU Rules |   Access privileges
-------------+----------+----------+-----------------+---------+---------+------------+-----------+-----------------------
 hw_triggers | postgres | UTF8     | libc            | C.UTF-8 | C.UTF-8 |            |           |
 postgres    | postgres | UTF8     | libc            | C.UTF-8 | C.UTF-8 |            |           |
 template0   | postgres | UTF8     | libc            | C.UTF-8 | C.UTF-8 |            |           | =c/postgres          +
             |          |          |                 |         |         |            |           | postgres=CTc/postgres
 template1   | postgres | UTF8     | libc            | C.UTF-8 | C.UTF-8 |            |           | =c/postgres          +
             |          |          |                 |         |         |            |           | postgres=CTc/postgres
(4 rows)
```
**Создадим в БД структуру описывающая товары (таблица **goods**) и продажи (таблица **sales**)**:
>> Создание схемы
```postgresql
postgres=# \c hw_triggers
hw_triggers=# CREATE SCHEMA pract_functions;
CREATE SCHEMA
You are now connected to database "hw_triggers" as user "postgres".
hw_triggers=# \dn
           List of schemas
      Name       |       Owner
-----------------+-------------------
 pract_functions | postgres
 public          | pg_database_owner
(2 rows)

hw_triggers=# SET search_path = pract_functions, publ;
SET
```
>> Заполнение витрины товарами:
```postgresql
hw_triggers=# CREATE TABLE goods
(
    goods_id    integer PRIMARY KEY,
    good_name   varchar(63) NOT NULL,
    good_price  numeric(12, 2) NOT NULL CHECK (good_price > 0.0)
);
INSERT INTO goods (goods_id, good_name, good_price)
VALUES  (1, 'Спички хозайственные', .50),
                (2, 'Автомобиль Ferrari FXX K', 185000000.01);
CREATE TABLE
INSERT 0 2
hw_triggers=# \dt
             List of relations
     Schema      | Name  | Type  |  Owner
-----------------+-------+-------+----------
 pract_functions | goods | table | postgres
(1 row)

hw_triggers=# SELEct * FROM goods;
 goods_id |        good_name         |  good_price
----------+--------------------------+--------------
        1 | Спички хозайственные     |         0.50
        2 | Автомобиль Ferrari FXX K | 185000000.01
(2 rows)

```
>> Заполняем таблицу продаж
```postgresql
hw_triggers=# CREATE TABLE sales
(
    sales_id    integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    good_id     integer REFERENCES goods (goods_id),
    sales_time  timestamp with time zone DEFAULT now(),
    sales_qty   integer CHECK (sales_qty > 0)
);
CREATE TABLE
hw_triggers=# \dt+
                                          List of relations
     Schema      | Name  | Type  |  Owner   | Persistence | Access method |    Size    | Description
-----------------+-------+-------+----------+-------------+---------------+------------+-------------
 pract_functions | goods | table | postgres | permanent   | heap          | 8192 bytes |
 pract_functions | sales | table | postgres | permanent   | heap          | 0 bytes    |
(2 rows)

hw_triggers=# SELECT * FROM sales;
 sales_id | good_id | sales_time | sales_qty
----------+---------+------------+-----------
(0 rows)

 hw_triggers=# INSERT INTO sales (good_id, sales_qty) VALUES (1, 10), (1, 1), (1, 120), (2, 1);
INSERT 0 4
hw_triggers=# SELECT * FROM sales;
 sales_id | good_id |          sales_time           | sales_qty
----------+---------+-------------------------------+-----------
        1 |       1 | 2024-12-07 17:23:17.122385+00 |        10
        2 |       1 | 2024-12-07 17:23:17.122385+00 |         1
        3 |       1 | 2024-12-07 17:23:17.122385+00 |       120
        4 |       2 | 2024-12-07 17:23:17.122385+00 |         1
(4 rows)
```
**Создадим запрос для генерации отчета, где будет сумма продаж по каждому товару.**:
```postgresql
hw_triggers=# SELECT * FROM sales;
 sales_id | good_id |          sales_time           | sales_qty
----------+---------+-------------------------------+-----------
        1 |       1 | 2024-12-07 17:23:17.122385+00 |        10
        2 |       1 | 2024-12-07 17:23:17.122385+00 |         1
        3 |       1 | 2024-12-07 17:23:17.122385+00 |       120
        4 |       2 | 2024-12-07 17:23:17.122385+00 |         1
(4 rows)

hw_triggers=# SELECT G.good_name, sum(G.good_price * S.sales_qty)
FROM goods G
INNER JOIN sales S ON S.good_id = G.goods_id
GROUP BY G.good_name;
        good_name         |     sum
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        65.50
(2 rows)

```
>> C увеличением объёма данных отчет стал создаваться медленно и было принято решение денормализовать БД, создать таблицу:
```postgresql
hw_triggers=# CREATE TABLE good_sum_mart
(
        good_name   varchar(63) NOT NULL,
        sum_sale        numeric(16, 2)NOT NULL
);
CREATE TABLE
hw_triggers=# \dt
                 List of relations
     Schema      |     Name      | Type  |  Owner
-----------------+---------------+-------+----------
 pract_functions | good_sum_mart | table | postgres
 pract_functions | goods         | table | postgres
 pract_functions | sales         | table | postgres
(3 rows)

hw_triggers=# SELECT * FROM good_sum_mart;
 good_name | sum_sale
-----------+----------
(0 rows)

```
****Создадим триггер на таблице продаж, для поддержки данных в витрине в актуальном состоянии:
```postgresql
hw_triggers=# INSERT INTO good_sum_mart (good_name, sum_sale) SELECT G.good_name, sum(G.good_price * S.sales_qty) FROM goods G INNER JOIN sales S ON S.good_id = G.goods_id GROUP BY G.good_name;
INSERT 0 2
hw_triggers=# \dt
                 List of relations
     Schema      |     Name      | Type  |  Owner
-----------------+---------------+-------+----------
 pract_functions | good_sum_mart | table | postgres
 pract_functions | goods         | table | postgres
 pract_functions | sales         | table | postgres
(3 rows)

hw_triggers=# SELECT * FROM good_sum_mart;
        good_name         |   sum_sale
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        65.50
(2 rows)

hw_triggers=# CREATE OR REPLACE FUNCTION trg_dz() 
RETURNS trigger AS $TRIG_FUNC$
BEGIN
DELETE FROM good_sum_mart * ;
INSERT INTO good_sum_mart (good_name, sum_sale) SELECT G.good_name, sum(G.good_price * S.sales_qty)  FROM goods G INNER JOIN sales S ON S.good_id = G.goods_id GROUP BY G.good_n                                                                                                ame;
RETURN NULL;
END;
$TRIG_FUNC$
LANGUAGE plpgsql;
CREATE FUNCTION

hw_triggers=# \df+
                                                                                  List of functions
     Schema      |  Name  | Result data type | Argument data types | Type | Volatility | Parallel |  Owner   | Security | Access privileges | Language | Internal name | Description
-----------------+--------+------------------+---------------------+------+------------+----------+----------+----------+-------------------+----------+---------------+-------------
 pract_functions | trg_dz | trigger          |                     | func | volatile   | unsafe   | postgres | invoker  |                   | plpgsql  |               |
(1 row)

hw_triggers=# CREATE TRIGGER tr_dz AFTER INSERT OR UPDATE OR DELETE ON sales FOR EACH STATEMENT EXECUTE FUNCTION trg_dz();
CREATE TRIGGER
hw_triggers=# \df
                            List of functions
     Schema      |  Name  | Result data type | Argument data types | Type
-----------------+--------+------------------+---------------------+------
 pract_functions | trg_dz | trigger          |                     | func
(1 row)

```
**Проверим c INSERT**
```postgresql
hw_triggers=# SELECT * FROM good_sum_mart;
        good_name         |   sum_sale
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        65.50
(2 rows)

hw_triggers=# INSERT INTO sales (good_id, sales_qty) VALUES (1,25);
INSERT 0 1
hw_triggers=# SELECT * FROM good_sum_mart;
        good_name         |   sum_sale
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        78.00
(2 rows)

hw_triggers=# SELECT * FROM sales;
 sales_id | good_id |          sales_time           | sales_qty
----------+---------+-------------------------------+-----------
        1 |       1 | 2024-12-07 17:23:17.122385+00 |        10
        2 |       1 | 2024-12-07 17:23:17.122385+00 |         1
        3 |       1 | 2024-12-07 17:23:17.122385+00 |       120
        4 |       2 | 2024-12-07 17:23:17.122385+00 |         1
        5 |       1 | 2024-12-07 17:59:14.683758+00 |        25
(5 rows)

 hw_triggers=# SELECT * FROM sales;
 sales_id | good_id |          sales_time           | sales_qty
----------+---------+-------------------------------+-----------
        1 |       1 | 2024-12-07 17:23:17.122385+00 |        10
        2 |       1 | 2024-12-07 17:23:17.122385+00 |         1
        3 |       1 | 2024-12-07 17:23:17.122385+00 |       120
        4 |       2 | 2024-12-07 17:23:17.122385+00 |         1
        5 |       1 | 2024-12-07 17:59:14.683758+00 |        25
(5 rows)

hw_triggers=# INSERT INTO sales (good_id, sales_qty) VALUES (1,250);
INSERT 0 1
hw_triggers=# INSERT INTO sales (good_id, sales_qty) VALUES (1,43);
INSERT 0 1
hw_triggers=# SELECT * FROM sales;
 sales_id | good_id |          sales_time           | sales_qty
----------+---------+-------------------------------+-----------
        1 |       1 | 2024-12-07 17:23:17.122385+00 |        10
        2 |       1 | 2024-12-07 17:23:17.122385+00 |         1
        3 |       1 | 2024-12-07 17:23:17.122385+00 |       120
        4 |       2 | 2024-12-07 17:23:17.122385+00 |         1
        5 |       1 | 2024-12-07 17:59:14.683758+00 |        25
        6 |       1 | 2024-12-07 18:00:45.546942+00 |       250
        7 |       1 | 2024-12-07 18:00:51.591305+00 |        43
(7 rows)

hw_triggers=# SELECT * FROM good_sum_mart;
        good_name         |   sum_sale
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |       224.50
(2 rows)

```
>> ok
**Проверим с UPDATE**
```postgresql
hw_triggers=# UPDATE sales SET sales_qty=120 WHERE sales_id=3;
UPDATE 1
hw_triggers=# SELECT * FROM sales;
 sales_id | good_id |          sales_time           | sales_qty
----------+---------+-------------------------------+-----------
        1 |       1 | 2024-12-07 17:23:17.122385+00 |        10
        2 |       1 | 2024-12-07 17:23:17.122385+00 |         1
        4 |       2 | 2024-12-07 17:23:17.122385+00 |         1
        5 |       1 | 2024-12-07 17:59:14.683758+00 |        25
        6 |       1 | 2024-12-07 18:00:45.546942+00 |       250
        7 |       1 | 2024-12-07 18:00:51.591305+00 |        43
        3 |       1 | 2024-12-07 17:23:17.122385+00 |       120
(7 rows)

```
>> ok
Проверим с DELETE
```postgresql
hw_triggers=# SELECT * FROM sales;
 sales_id | good_id |          sales_time           | sales_qty
----------+---------+-------------------------------+-----------
        1 |       1 | 2024-12-07 17:23:17.122385+00 |        10
        2 |       1 | 2024-12-07 17:23:17.122385+00 |         1
        4 |       2 | 2024-12-07 17:23:17.122385+00 |         1
        5 |       1 | 2024-12-07 17:59:14.683758+00 |        25
        6 |       1 | 2024-12-07 18:00:45.546942+00 |       250
        7 |       1 | 2024-12-07 18:00:51.591305+00 |        43
        3 |       1 | 2024-12-07 17:23:17.122385+00 |       120
(7 rows)

hw_triggers=# DELETE FROM sales WHERE sales_id=3;
DELETE 1
hw_triggers=# SELECT * FROM good_sum_mart;
        good_name         |   sum_sale
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |       164.50
(2 rows)

hw_triggers=# SELECT * FROM sales;
 sales_id | good_id |          sales_time           | sales_qty
----------+---------+-------------------------------+-----------
        1 |       1 | 2024-12-07 17:23:17.122385+00 |        10
        2 |       1 | 2024-12-07 17:23:17.122385+00 |         1
        4 |       2 | 2024-12-07 17:23:17.122385+00 |         1
        5 |       1 | 2024-12-07 17:59:14.683758+00 |        25
        6 |       1 | 2024-12-07 18:00:45.546942+00 |       250
        7 |       1 | 2024-12-07 18:00:51.591305+00 |        43
(6 rows)

```
>> Всё работает, актуализация данных происходит. 