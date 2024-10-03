# Домашнее задание 06
## Работа с журналами

### Цель:
* Уметь работать с журналами и контрольными точками
* Уметь настраивать параметры журналов

### Описание/Пошаговая инструкция выполнения домашнего задания:
1. Настройте выполнение контрольной точки раз в 30 секунд.
2. 10 минут c помощью утилиты pgbench подавайте нагрузку.
3. Измерьте, какой объем журнальных файлов был сгенерирован за это время. Оцените, какой объем приходится в среднем на одну контрольную точку.
4. Проверьте данные статистики: все ли контрольные точки выполнялись точно по расписанию. Почему так произошло?
5. Сравните tps в синхронном/асинхронном режиме утилитой pgbench. Объясните полученный результат.
6. Создайте новый кластер с включенной контрольной суммой страниц. Создайте таблицу. Вставьте несколько значений. Выключите кластер. Измените пару байт в таблице. Включите кластер и сделайте выборку из таблицы. Что и почему произошло? как проигнорировать ошибку и продолжить работу?
---
#### Установим instance YC и настроим выполнение контрольной точки раз в 30 секунд.
Установка instance YandexCloud:
````shell
PS C:\Users\Alexander\.ssh> yc compute instance create --name otus-wal --hostname otus-wal --cores 2 --memory 4 --create-boot-disk size=10
G,type=network-hdd,image-folder-id=standard-images,image-family=ubuntu-2204-lts --network-interface subnet-name=otus-subnet,nat-ip-version
=ipv4 --ssh-key C:\Users\Alexander/.ssh/id_rsa.pub
done (47s)
````
Проверим:
```postgresql
PS C:\Users\Alexander> yc compute instances list
+----------------------+----------+---------------+---------+----------------+----------------+
|          ID          |   NAME   |    ZONE ID    | STATUS  |  EXTERNAL IP   |  INTERNAL IP   |
+----------------------+----------+---------------+---------+----------------+----------------+
| fhmud82u5blg9pu293be | otus-wal | ru-central1-a | RUNNING | 84.252.131.151 | 192.168.100.32 |
+----------------------+----------+---------------+---------+----------------+----------------+
```
Установим БД Postgresql:
```shell
yc-user@otus-wal:~$ sudo apt install postgresql
yc-user@otus-wal:~$ sudo pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
```
Создадим тестовую БД:
```postgresql
postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
testdb=# CREATE TABLE Products
(
    Id SERIAL PRIMARY KEY,
    ProductName VARCHAR(30) NOT NULL,
    Manufacturer VARCHAR(20) NOT NULL,
    ProductCount INTEGER DEFAULT 0,
    Price NUMERIC
);
CREATE TABLE
testdb=# \d Products
                                      Table "public.products"
    Column    |         Type          | Collation | Nullable |               Default
--------------+-----------------------+-----------+----------+--------------------------------------
 id           | integer               |           | not null | nextval('products_id_seq'::regclass)
 productname  | character varying(30) |           | not null |
 manufacturer | character varying(20) |           | not null |
 productcount | integer               |           |          | 0
 price        | numeric               |           |          |
Indexes:
    "products_pkey" PRIMARY KEY, btree (id)

testdb=# INSERT INTO Products  (ProductName, Manufacturer, ProductCount, Price)
VALUES
('iPhone 6', 'Apple', 3, 36000),
('Galaxy S8', 'Samsung', 2, 46000),
('Galaxy S8 Plus', 'Samsung', 1, 56000),
('Desire 12', 'HTC', 8, 21000),
('iPhone X', 'Apple', 9, 93000);
INSERT 0 5
testdb=# SELECT * FROM Products;
 id |  productname   | manufacturer | productcount | price
----+----------------+--------------+--------------+-------
  1 | iPhone 6       | Apple        |            3 | 36000
  2 | Galaxy S8      | Samsung      |            2 | 46000
  3 | Galaxy S8 Plus | Samsung      |            1 | 56000
  4 | Desire 12      | HTC          |            8 | 21000
  5 | iPhone X       | Apple        |            9 | 93000
(5 rows)
```

```postgresql
postgres=# show checkpoint_timeout;
 checkpoint_timeout
--------------------
 5min
(1 row)
```
Установим выполнение контрольных точек в 30 сек.:
```postgresql
testdb=# show checkpoint_timeout;
 checkpoint_timeout
--------------------
 5min
(1 row)

testdb=# alter system set checkpoint_timeout TO '30s';
ALTER SYSTEM
testdb=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

testdb=# show checkpoint_timeout;
 checkpoint_timeout
--------------------
 30s
(1 row)

testdb=#
 
````
#### 10 минут c помощью утилиты **pgbench** дадим нагрузку:
До теста проверим текущий LSN:
````postgresql
postgres=# select pg_current_wal_lsn(), pg_current_wal_insert_lsn();
 pg_current_wal_lsn | pg_current_wal_insert_lsn
--------------------+---------------------------
 0/16EF6F48         | 0/16EF6F48
(1 row)

postgres=# select *from pg_stat_bgwriter \gx
-[ RECORD 1 ]---------+------------------------------
checkpoints_timed     | 267
checkpoints_req       | 2
checkpoint_write_time | 569541
checkpoint_sync_time  | 1029
buffers_checkpoint    | 37149
buffers_clean         | 0
maxwritten_clean      | 0
buffers_backend       | 2834
buffers_backend_fsync | 0
buffers_alloc         | 3595
stats_reset           | 2024-10-03 09:35:46.469022+00

````
Теперь проведём тест утилитой **pgbench**:
````shell
postgres@otus-wal:~$ pgbench -i testdb
postgres@otus-wal:~$ pgbench -P 60 -T 600 testdb
pgbench (14.13 (Ubuntu 14.13-0ubuntu0.22.04.1))
starting vacuum...end.
progress: 60.0 s, 273.1 tps, lat 3.661 ms stddev 5.140
progress: 120.0 s, 248.8 tps, lat 4.018 ms stddev 5.660
progress: 180.0 s, 298.1 tps, lat 3.354 ms stddev 4.873
progress: 240.0 s, 301.4 tps, lat 3.317 ms stddev 4.988
progress: 300.0 s, 302.1 tps, lat 3.310 ms stddev 5.106
progress: 360.0 s, 282.6 tps, lat 3.539 ms stddev 5.149
progress: 420.0 s, 322.3 tps, lat 3.102 ms stddev 4.859
progress: 480.0 s, 277.0 tps, lat 3.611 ms stddev 5.197
progress: 540.0 s, 318.2 tps, lat 3.142 ms stddev 4.743
progress: 600.0 s, 272.6 tps, lat 3.667 ms stddev 5.324
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
duration: 600 s
number of transactions actually processed: 173773
latency average = 3.452 ms
latency stddev = 5.099 ms
initial connection time = 4.365 ms
tps = 289.622830 (without initial connection time)
````
#### Измерим, какой объем журнальных файлов был сгенерирован за это время.
```postgresql
postgres=# select pg_current_wal_lsn(), pg_current_wal_insert_lsn();
 pg_current_wal_lsn | pg_current_wal_insert_lsn
--------------------+---------------------------
 0/2BD6FD28         | 0/2BD6FD28
(1 row)

postgres=# select *from pg_stat_bgwriter \gx
-[ RECORD 1 ]---------+------------------------------
checkpoints_timed     | 297
checkpoints_req       | 2
checkpoint_write_time | 1134709
checkpoint_sync_time  | 1469
buffers_checkpoint    | 71482
buffers_clean         | 0
maxwritten_clean      | 0
buffers_backend       | 3955
buffers_backend_fsync | 0
buffers_alloc         | 4717
stats_reset           | 2024-10-03 09:35:46.469022+00

postgres=# select '0/16EF6F48'::pg_lsn - '0/2BD6FD28'::pg_lsn as bytes;
   bytes
------------
 -350719456 (334MB)
(1 row)
   
postgres=# select pg_size_pretty(('0/16EF6F48'::pg_lsn - '0/2BD6FD28'::pg_lsn) / 30);
 pg_size_pretty
----------------
 -11 MB
(1 row)   
```
>> В среднем получается 30 точек за 10 минут теста, размер одной точки получился 11 МB , а общий объем 334 MB

> Почему так произошло?
> >
### Сравните **tps** в синхронном/асинхронном режиме утилитой **pgbench**:
> - Объясните полученный результат.
> >

#### Создадим новый кластер с включенной контрольной суммой страниц: 
```shell

```
#### Создадим таблицу:
```postgresql

```
#### Вставим несколько значений: 
```postgresql

```
#### Выключим кластер:
```shell

```
#### Изменим пару байт в таблице: 
```shell

```
#### Включим кластер и сделайте выборку из таблицы:
```shell

```
```postgresql

```
> Что и почему произошло? Как проигнорировать ошибку и продолжить работу?
>> 
```shell

```
