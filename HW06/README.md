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
#### Измерим, какой объем журнальных файлов был сгенерирован за это время. Оцените, какой объем приходится в среднем на одну контрольную точку.
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

#### Проверьте данные статистики: все ли контрольные точки выполнялись точно по расписанию. Почему так произошло?
````postgresql
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

postgres=# show max_wal_size;
 max_wal_size
--------------
 1GB
(1 row)
````
>> Из проверки следует, что _**checkpoints_req**_ остался такой же и уложился в checkpoints_timed. Все точки выполнялись точно по расписанию, так как мало данных что-бы сработал  _**max_wal_size**_.
#### Сравните tps в синхронном/асинхронном режиме утилитой pgbench. Объясните полученный результат.
Проверим режим работы:
```postgresql
postgres@otus-wal:~$ psql
psql (14.13 (Ubuntu 14.13-0ubuntu0.22.04.1))
Type "help" for help.

postgres=# show synchronous_commit;
 synchronous_commit
--------------------
 on
(1 row)

postgres=#

```
Запустим тест в синхронном режиме:
```shell
postgres@otus-wal:~$ pgbench -P 60 -T 600 testdb
pgbench (14.13 (Ubuntu 14.13-0ubuntu0.22.04.1))
starting vacuum...end.
progress: 60.0 s, 343.2 tps, lat 2.913 ms stddev 4.494
progress: 120.0 s, 364.4 tps, lat 2.744 ms stddev 4.245
progress: 180.0 s, 371.5 tps, lat 2.691 ms stddev 4.234
progress: 240.0 s, 356.6 tps, lat 2.804 ms stddev 4.548
progress: 300.0 s, 368.8 tps, lat 2.711 ms stddev 4.550
progress: 360.0 s, 360.4 tps, lat 2.774 ms stddev 4.350
progress: 420.0 s, 343.2 tps, lat 2.913 ms stddev 4.449
progress: 480.0 s, 368.6 tps, lat 2.713 ms stddev 4.216
progress: 540.0 s, 344.7 tps, lat 2.901 ms stddev 4.388
progress: 600.0 s, 377.6 tps, lat 2.648 ms stddev 4.291
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
duration: 600 s
number of transactions actually processed: 215939
latency average = 2.778 ms
latency stddev = 4.377 ms
initial connection time = 4.089 ms
tps = 359.899131 (without initial connection time)

```
Выключаем синхронный режим:
````postgresql
postgres@otus-wal:~$ psql
psql (14.13 (Ubuntu 14.13-0ubuntu0.22.04.1))
Type "help" for help.

postgres=# alter system set synchronous_commit=off;
ALTER SYSTEM
postgres=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

postgres=# show synchronous_commit;
 synchronous_commit
--------------------
 off
(1 row)

postgres=#

````
Теперь можно запустить тест в асинхронном режиме:
```shell
postgres@otus-wal:~$ pgbench -P 60 -T 600 testdb
pgbench (14.13 (Ubuntu 14.13-0ubuntu0.22.04.1))
starting vacuum...end.
progress: 60.0 s, 1799.4 tps, lat 0.555 ms stddev 0.070
progress: 120.0 s, 1789.7 tps, lat 0.558 ms stddev 0.119
progress: 180.0 s, 1766.1 tps, lat 0.566 ms stddev 0.100
progress: 240.0 s, 1773.4 tps, lat 0.564 ms stddev 0.137
progress: 300.0 s, 1769.5 tps, lat 0.565 ms stddev 0.213
progress: 360.0 s, 1774.4 tps, lat 0.563 ms stddev 0.109
progress: 420.0 s, 1800.5 tps, lat 0.555 ms stddev 0.092
progress: 480.0 s, 1785.3 tps, lat 0.560 ms stddev 0.088
progress: 540.0 s, 1767.5 tps, lat 0.565 ms stddev 0.071
progress: 600.0 s, 1731.8 tps, lat 0.577 ms stddev 0.215
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
duration: 600 s
number of transactions actually processed: 1065467
latency average = 0.563 ms
latency stddev = 0.131 ms
initial connection time = 4.192 ms
tps = 1775.788248 (without initial connection time)

```
>> Из теста следует, что _**tps**_  существенно вырос в асинхронном режиме. Асинхронный режим транзакция завершается быстрее, но в случае краха БД последние транзакции будут потеряны.
#### Создайте новый кластер с включенной контрольной суммой страниц. Создайте таблицу. Вставьте несколько значений. Выключите кластер. Измените пару байт в таблице. Включите кластер и сделайте выборку из таблицы. 
> Что и почему произошло? как проигнорировать ошибку и продолжить работу?
````shell
postgres@otus-wal:~$ pg_dropcluster 14 main --stop
Warning: systemd was not informed about the removed cluster yet. Operations like "service postgresql start" might fail. To fix, run:
  sudo systemctl daemon-reload
postgres@otus-wal:~$ pg_lsclusters
Ver Cluster Port Status Owner Data directory Log file
postgres@otus-wal:~$ pg_createcluster --start -- -k  14 main
Error: invalid version '-k'
postgres@otus-wal:~$ pg_createcluster 14 main --start -- --data-checksums
Creating new PostgreSQL cluster 14/main ...
/usr/lib/postgresql/14/bin/initdb -D /var/lib/postgresql/14/main --auth-local peer --auth-host scram-sha-256 --no-instructions --data-checksums
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locale "en_US.UTF-8".
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are enabled.

fixing permissions on existing directory /var/lib/postgresql/14/main ... ok
creating subdirectories ... ok
selecting dynamic shared memory implementation ... posix
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting default time zone ... Etc/UTC
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok
Warning: systemd does not know about the new cluster yet. Operations like "service postgresql start" will not handle it. To fix, run:
  sudo systemctl daemon-reload
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
  sudo systemctl start postgresql@14-main
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
````
```postgresql
postgres@otus-wal:~$ psql
psql (14.13 (Ubuntu 14.13-0ubuntu0.22.04.1))
Type "help" for help.

postgres=# show fsync;
 fsync
-------
 on
(1 row)

postgres=# show wal_sync_method;
 wal_sync_method
-----------------
 fdatasync
(1 row)

postgres=# show data_checksums;
 data_checksums
----------------
 on
(1 row)

postgres=#
```
Создадим таблицу и вставим несколько значений:
```postgresql
postgres=# create table test1(id serial,name text);
CREATE TABLE
postgres=# \dt
         List of relations
 Schema | Name  | Type  |  Owner
--------+-------+-------+----------
 public | test1 | table | postgres
(1 row)

postgres=# insert into test1 (name) select md5(random()::text) from generate_series(1,1000);
INSERT 0 1000
postgres=# select * from test1 limit 10;
 id |               name
----+----------------------------------
  1 | 7c07e818e6dbe9da47fb040c962162c9
  2 | 5e7ce2bcd927c322aa4bb1ab96519310
  3 | f47e94bb5e25102348eef444501c7ae0
  4 | 8bc7cd94d22bf53758cd123df49d9bd1
  5 | e366faed83016e9b86a31822a0734976
  6 | 9539136622729063b79308732e65e220
  7 | 1c4c139f2416ca4ddd96bbea513da898
  8 | 8b46d5c46da6ddfc81135f3c4b76cce7
  9 | 0183b7f1fccc7b9149bc8f2828c641fb
 10 | b412e101600be69bf240f54ea76ecac9
(10 rows)

 postgres=# SELECT pg_relation_filepath('test1');
 pg_relation_filepath
----------------------
 base/13761/16385
(1 row)
```
Остановим кластер БД и изменим пару байт в таблице:
```shell
postgres@otus-wal:~$ pg_ctlcluster 14 main stop
postgres@otus-wal:~$ pg_ctlcluster 14 main status
pg_ctl: no server running
postgres@otus-wal:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 down   postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
postgres@otus-wal:~$ dd if=/dev/random of=/var/lib/postgresql/14/main/base/13761/16385 oflag=dsync conv=notrunc bs=1 count=8
8+0 records in
8+0 records out
8 bytes copied, 0.00681014 s, 1.2 kB/s
postgres@otus-wal:~$ pg_ctlcluster 14 main start
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
  sudo systemctl start postgresql@14-main
```
```postgresql
postgres@otus-wal:~$ psql
psql (14.13 (Ubuntu 14.13-0ubuntu0.22.04.1))
Type "help" for help.

postgres=# show data_checksums ;
 data_checksums
----------------
 on
(1 row)

postgres=# select * from test1 limit 10;
WARNING:  page verification failed, calculated checksum 27878 but expected 3906
ERROR:  invalid page in block 0 of relation base/13761/16385
```
>> При выборке данных из таблицы, выдается ошибка что данные повреждены. Это происходит из-за того, что чексума не сходится. 
>> Попробуем проигнорировать ошибку и запуститься:
```postgresql
postgres=# alter system set ignore_checksum_failure to true;
ALTER SYSTEM
postgres=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

postgres=# show ignore_checksum_failure;
 ignore_checksum_failure
-------------------------
 on
(1 row)

postgres=# select * from test1 limit 10;
WARNING:  page verification failed, calculated checksum 27878 but expected 3906
 id |               name
----+----------------------------------
  1 | 7c07e818e6dbe9da47fb040c962162c9
  2 | 5e7ce2bcd927c322aa4bb1ab96519310
  3 | f47e94bb5e25102348eef444501c7ae0
  4 | 8bc7cd94d22bf53758cd123df49d9bd1
  5 | e366faed83016e9b86a31822a0734976
  6 | 9539136622729063b79308732e65e220
  7 | 1c4c139f2416ca4ddd96bbea513da898
  8 | 8b46d5c46da6ddfc81135f3c4b76cce7
  9 | 0183b7f1fccc7b9149bc8f2828c641fb
 10 | b412e101600be69bf240f54ea76ecac9
(10 rows)
```
