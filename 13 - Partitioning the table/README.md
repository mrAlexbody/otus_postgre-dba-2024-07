# Домашнее задание
## Секционирование таблицы

### Цель:
* научиться выполнять секционирование таблиц в PostgreSQL;
* повысить производительность запросов и упростив управление данными;


### Описание/Пошаговая инструкция выполнения домашнего задания:
**Описание задания:**
* На основе готовой базы данных примените один из методов секционирования в зависимости от структуры данных.
* https://postgrespro.ru/education/demodb

### Шаги выполнения домашнего задания:
**Анализ структуры данных:**
* Ознакомьтесь с таблицами базы данных, особенно с таблицами bookings, tickets, ticket_flights, flights, boarding_passes, seats, airports, aircrafts.
* Определите, какие данные в таблице bookings или других таблицах имеют логическую привязку к диапазонам, по которым можно провести секционирование (например, дата бронирования, рейсы).

**Выбор таблицы для секционирования:**
* Основной акцент делается на секционировании таблицы bookings. Но вы можете выбрать и другие таблицы, если видите в этом смысл для оптимизации производительности (например, flights, boarding_passes).
* Обоснуйте свой выбор: почему именно эта таблица требует секционирования? Какой тип данных является ключевым для секционирования?
* 
**Определение типа секционирования:**
* Определитесь с типом секционирования, которое наилучшим образом подходит для ваших данных:
  * По диапазону (например, по дате бронирования или дате рейса).
  * По списку (например, по пунктам отправления или по номерам рейсов).
  * По хэшированию (для равномерного распределения данных).

**Создание секционированной таблицы:**
* Преобразуйте таблицу в секционированную с выбранным типом секционирования.
* Например, если вы выбрали секционирование по диапазону дат бронирования, создайте секции по месяцам или годам.

**Миграция данных:**
* Перенесите существующие данные из исходной таблицы в секционированную структуру.
* Убедитесь, что все данные правильно распределены по секциям.

**Оптимизация запросов:**
* Проверьте, как секционирование влияет на производительность запросов. Выполните несколько выборок данных до и после секционирования для оценки времени выполнения.
* Оптимизируйте запросы при необходимости (например, добавьте индексы на ключевые столбцы).

**Тестирование решения:**
* Протестируйте секционирование, выполняя несколько запросов к секционированной таблице.
* Проверьте, что операции вставки, обновления и удаления работают корректно.

**Документирование:**
* Добавьте комментарии к коду, поясняющие выбранный тип секционирования и шаги его реализации.
* Опишите, как секционирование улучшает производительность запросов и как оно может быть полезно в реальных условиях.

**Критерии оценивания:**
* Корректность секционирования – таблица должна быть разделена логично и эффективно.
* Выбор типа секционирования – обоснование выбранного типа (например, секционирование по диапазону дат рейсов или по месту отправления/прибытия).
* Работоспособность решения – код должен успешно выполнять секционирование без ошибок.
* Оптимизация запросов – после секционирования, запросы к таблице должны быть оптимизированы (например, быстрее выполняться для конкретных диапазонов).
* Комментирование – код должен содержать поясняющие комментарии, объясняющие выбор секционирования и основные шаги.

**Формат сдачи:**
* SQL-скрипты с реализованным секционированием.
* Краткий отчет с описанием процесса и результатами тестирования.
* Пример запросов и результаты до и после секционирования.

--- 
Скачаем базу данных:
```shell
ubuntu@db-join:~$ curl https://edu.postgrespro.ru/demo-small.zip -o demo-medium.zip
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 21.1M  100 21.1M    0     0  23.9M      0 --:--:-- --:--:-- --:--:-- 23.9M

ubuntu@db-join:~$ ls -la ./demo-medium.zip
-rw-rw-r-- 1 ubuntu ubuntu 22187733 ноя 30 11:04 ./demo-medium.zip
```
Установим базу данных из архива: 
```postgresql
ubuntu@db-join:~$ sudo -i -u postgres psql
[sudo] password for ubuntu:
psql (17.0 (Ubuntu 17.0-1.pgdg24.04+1), сервер 15.8 (Ubuntu 15.8-1.pgdg24.04+1))
Введите "help", чтобы получить справку.

postgres=# \l
                                                       Список баз данных
    Имя    | Владелец | Кодировка | Провайдер локали | LC_COLLATE  |  LC_CTYPE   | Локаль | Правила ICU |     Права доступа
-----------+----------+-----------+------------------+-------------+-------------+--------+-------------+-----------------------
 joinstat  | postgres | UTF8      | libc             | ru_RU.UTF-8 | ru_RU.UTF-8 |        |             |
 otus      | postgres | UTF8      | libc             | ru_RU.UTF-8 | ru_RU.UTF-8 |        |             |
 postgres  | postgres | UTF8      | libc             | ru_RU.UTF-8 | ru_RU.UTF-8 |        |             |
 template0 | postgres | UTF8      | libc             | ru_RU.UTF-8 | ru_RU.UTF-8 |        |             | =c/postgres          +
           |          |           |                  |             |             |        |             | postgres=CTc/postgres
 template1 | postgres | UTF8      | libc             | ru_RU.UTF-8 | ru_RU.UTF-8 |        |             | =c/postgres          +
           |          |           |                  |             |             |        |             | postgres=CTc/postgres
(5 строк)

postgres=# CREATE DATABASE demo;
CREATE DATABASE
postgres=# \q

```
**Диаграмма:**

![demo - bookings.png](demo%20-%20bookings.png)

```shell
ubuntu@db-join:~$ mv ./demo-medium.zip /tmp
ubuntu@db-join:/tmp$ sudo -i -u postgres bash
postgres@db-join:/tmp$ unzip -p ./demo-medium.zip | psql -U postgres -d demo
SET
SET
SET
SET
SET
SET
SET
SET
ОШИБКА:  удалить базу данных, открытую в данный момент, нельзя
ОШИБКА:  база данных "demo" уже существует
Вы подключены к базе данных "demo" как пользователь "postgres".
SET
SET
SET
SET
SET
SET
SET
SET
CREATE SCHEMA
COMMENT
CREATE EXTENSION
COMMENT
SET
CREATE FUNCTION
CREATE FUNCTION
COMMENT
SET
SET
CREATE TABLE
COMMENT
COMMENT
COMMENT
COMMENT
CREATE VIEW
COMMENT
COMMENT
COMMENT
COMMENT
CREATE TABLE
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
CREATE VIEW
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
CREATE TABLE
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
CREATE TABLE
COMMENT
COMMENT
COMMENT
COMMENT
CREATE TABLE
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
CREATE SEQUENCE
ALTER SEQUENCE
CREATE VIEW
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
CREATE VIEW
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
CREATE TABLE
COMMENT
COMMENT
COMMENT
COMMENT
CREATE TABLE
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
CREATE TABLE
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
ALTER TABLE
COPY 9
COPY 104
COPY 579686
COPY 262788
COPY 33121
 setval
--------
  33121
(1 строка)

COPY 1339
COPY 1045726
COPY 366733
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER DATABASE
ALTER DATABASE

```
Проверим какой размер таблиц и есть ли секционированные таблицы:
```postgresql
demo=# SELECT
    table_name,
    pg_size_pretty(table_size) AS table_size,
    pg_size_pretty(indexes_size) AS indexes_size,
    pg_size_pretty(total_size) AS total_size
FROM (
    SELECT
        table_name,
        pg_table_size(table_name) AS table_size,
        pg_indexes_size(table_name) AS indexes_size,
        pg_total_relation_size(table_name) AS total_size
    FROM (
        SELECT ('"' || table_schema || '"."' || table_name || '"') AS table_name
        FROM information_schema.tables
    ) AS all_tables
    ORDER BY total_size DESC
) AS pretty_sizes limit 5;
          table_name          | table_size | indexes_size | total_size
------------------------------+------------+--------------+------------
 "bookings"."ticket_flights"  | 68 MB      | 41 MB        | 109 MB
 "bookings"."boarding_passes" | 33 MB      | 47 MB        | 81 MB
 "bookings"."tickets"         | 48 MB      | 11 MB        | 59 MB
 "bookings"."bookings"        | 13 MB      | 5784 kB      | 19 MB
 "bookings"."flights"         | 3160 kB    | 1776 kB      | 4936 kB
(5 строк)

demo=# SELECT count(1) FROM pg_catalog.pg_partitioned_table;
 count
-------
     0
(1 строка)
```
Проверим поиск:
```postgresql
demo=# EXPLAIN ANALYSE SELECT * FROM bookings WHERE book_ref='203B66';
                                                        QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------
 Index Scan using bookings_pkey on bookings  (cost=0.42..2.64 rows=1 width=52) (actual time=11.561..11.562 rows=0 loops=1)
   Index Cond: (book_ref = '203B66'::bpchar)
 Planning Time: 0.088 ms
 Execution Time: 11.582 ms
(4 строки)
```
Сделаем секционирование таблицы bookings: 
```postgresql
demo=# CREATE TABLE bookings_num (
       book_ref     character(6),
       book_date    timestamptz,
       total_amount numeric(10,2),
       CONSTRAINT bookings_num_pkey PRIMARY KEY (book_ref)
   ) PARTITION BY HASH(book_ref);
CREATE TABLE

demo=# CREATE TABLE bookings_0 PARTITION OF bookings_num FOR VALUES WITH (MODULUS 10, REMAINDER 0);
CREATE TABLE
demo=# CREATE TABLE bookings_1 PARTITION OF bookings_num FOR VALUES WITH (MODULUS 10, REMAINDER 1);
CREATE TABLE
demo=# CREATE TABLE bookings_2 PARTITION OF bookings_num FOR VALUES WITH (MODULUS 10, REMAINDER 2);
CREATE TABLE
demo=# CREATE TABLE bookings_3 PARTITION OF bookings_num FOR VALUES WITH (MODULUS 10, REMAINDER 3);
CREATE TABLE
demo=# CREATE TABLE bookings_4 PARTITION OF bookings_num FOR VALUES WITH (MODULUS 10, REMAINDER 4);
CREATE TABLE
demo=# CREATE TABLE bookings_5 PARTITION OF bookings_num FOR VALUES WITH (MODULUS 10, REMAINDER 5);
CREATE TABLE
demo=# CREATE TABLE bookings_6 PARTITION OF bookings_num FOR VALUES WITH (MODULUS 10, REMAINDER 6);
CREATE TABLE
demo=# CREATE TABLE bookings_7 PARTITION OF bookings_num FOR VALUES WITH (MODULUS 10, REMAINDER 7);
CREATE TABLE
demo=# CREATE TABLE bookings_8 PARTITION OF bookings_num FOR VALUES WITH (MODULUS 10, REMAINDER 8);
CREATE TABLE
demo=# CREATE TABLE bookings_9 PARTITION OF bookings_num FOR VALUES WITH (MODULUS 10, REMAINDER 9);
CREATE TABLE
```
В итоге, получили следующею таблицу:
```postgresql
demo=# \d+ bookings_p;
                                            Partitioned table "bookings.bookings_p"
    Column    |           Type           | Collation | Nullable | Default | Storage  | Compression | Stats target | Description
--------------+--------------------------+-----------+----------+---------+----------+-------------+--------------+-------------
 book_ref     | character(6)             |           | not null |         | extended |             |              |
 book_date    | timestamp with time zone |           |          |         | plain    |             |              |
 total_amount | numeric(10,2)            |           |          |         | main     |             |              |
Partition key: HASH (book_ref)
Indexes:
    "bookings_p_pkey" PRIMARY KEY, btree (book_ref)
Partitions: bookings_0 FOR VALUES WITH (modulus 8, remainder 0),
            bookings_1 FOR VALUES WITH (modulus 8, remainder 1),
            bookings_2 FOR VALUES WITH (modulus 8, remainder 2),
            bookings_3 FOR VALUES WITH (modulus 8, remainder 3),
            bookings_4 FOR VALUES WITH (modulus 8, remainder 4),
            bookings_5 FOR VALUES WITH (modulus 8, remainder 5),
            bookings_6 FOR VALUES WITH (modulus 8, remainder 6),
            bookings_7 FOR VALUES WITH (modulus 8, remainder 7)
```
Сделаем перенос данных из таблицы booking в booking_num:
```postgresql
demo=# INSERT INTO bookings_num SELECT * FROM bookings;
INSERT 0 262788
```
Проверим:
```postgresql
demo=# SELECT * FROM bookings_num LIMIT 10;
 book_ref |       book_date        | total_amount
----------+------------------------+--------------
 000012   | 2017-07-14 06:02:00+00 |     37900.00
 00053F   | 2017-08-06 00:15:00+00 |      6000.00
 0008F4   | 2017-07-31 21:33:00+00 |     28000.00
 00111B   | 2017-07-09 11:30:00+00 |     29600.00
 0012E3   | 2017-07-24 03:07:00+00 |    125700.00
 0014E3   | 2017-07-08 00:11:00+00 |     16000.00
 001628   | 2017-07-15 07:59:00+00 |    813300.00
 0016BD   | 2017-07-15 22:27:00+00 |     59300.00
 001A4A   | 2017-08-02 01:31:00+00 |     35900.00
 001FA1   | 2017-07-17 11:28:00+00 |     33000.00
(10 строк)
```
Поправим зависимые таблицы:
```postgresql
demo=# ALTER TABLE tickets DROP CONSTRAINT tickets_book_ref_fkey;
ALTER TABLE
```
```postgresql
demo=# ALTER TABLE tickets ADD CONSTRAINT tickets_book_ref_fkey FOREIGN KEY (book_ref) REFERENCES bookings_num (book_ref);
ALTER TABLE
```
```postgresql
demo=# ALTER TABLE bookings RENAME TO bookings_old;
ALTER TABLE
```
```postgresql
demo=# ALTER  TABLE bookings_num RENAME TO bookings;
ALTER TABLE
```
>> Получим в итоге более быстрый поиск в секционированной таблице. log n - поиск по бинарному дереву. 
>> Чем меньше записей в таблице тем быстрее поиск.
```postgresql
demo=# EXPLAIN ANALYSE SELECT * FROM bookings WHERE book_ref='203B66';
                                                              QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------------------
 Index Scan using bookings_5_pkey on bookings_5 bookings  (cost=0.29..2.51 rows=1 width=52) (actual time=0.010..0.010 rows=0 loops=1)
   Index Cond: (book_ref = '203B66'::bpchar)
 Planning Time: 0.268 ms
 Execution Time: 0.021 ms
(4 строки)
                                                        
```
