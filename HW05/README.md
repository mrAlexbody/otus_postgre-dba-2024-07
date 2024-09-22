# Домашнее задание 05
## Настройка autovacuum с учетом особеностей производительности

### Цель:
* Запустить нагрузочный тест pgbench
* Настроить параметры autovacuum
* Проверить работу autovacuum

#### Описание/Пошаговая инструкция выполнения домашнего задания:
1. Создать **instance ВМ** с **2 ядрами** и **4 Гб ОЗУ** и **SSD 10GB**
2. Установить на него **PostgreSQL 15** с дефолтными настройками
3. Создать БД для тестов: выполнить _**pgbench -i postgres**_
4. Запустить _**pgbench -c8 -P 6 -T 60 -U postgres postgres**_
5. Применить параметры настройки **PostgreSQL** из прикрепленного к материалам занятия файла
6. Протестировать заново
7. Что изменилось и почему?
8. Создать таблицу с текстовым полем и заполнить случайными или сгенерированными данным в размере **1млн строк**
9. Посмотреть размер файла с таблицей
10. **5 раз** обновить все строчки и добавить к каждой строчке любой символ
11. Посмотреть количество мертвых строчек в таблице и когда последний раз приходил автовакуум
12. Подождать некоторое время, проверяя, пришел ли автовакуум
13. **5 раз** обновить все строчки и добавить к каждой строчке любой символ
14. Посмотреть размер файла с таблицей
15. Отключить Автовакуум на конкретной таблице
16. **10 раз** обновить все строчки и добавить к каждой строчке любой символ
17. Посмотреть размер файла с таблицей
18. Объясните полученный результат (
P.S. Не забудьте включить **автовакуум** =)
### Задание со *:
* Написать анонимную процедуру, в которой в цикле 10 раз обновятся все строчки в искомой таблице. 
* Не забыть вывести номер шага цикла.

---
#### Создадим инстанс в _Yandex cloud_ с **2 ядрами** и **4 Гб ОЗУ** и **SSD 10GB**:
```shell
PS C:\Users\Alexander> yc compute instance create --name otus-vacum --hostname otus-vacum --cores 2 --memory 4 --create-boot-disk size=10G,type=network-hdd,image-folder-id=standard-images,image-family=ubuntu-2204-lts --network-interface subnet-name=otus-subnet,nat-ip-version=ipv4 --ssh-key C:\Users\Alexander/.ssh/id_rsa.pub
done (43s)
PS C:\Users\Alexander> yc compute instances list
+----------------------+------------+---------------+---------+---------------+----------------+
|          ID          |    NAME    |    ZONE ID    | STATUS  |  EXTERNAL IP  |  INTERNAL IP   |
+----------------------+------------+---------------+---------+---------------+----------------+
| fhmvo1einkk6da5acc0h | otus-vacum | ru-central1-a | RUNNING | 84.201.172.47 | 192.168.100.19 |
+----------------------+------------+---------------+---------+---------------+----------------+
```
### Установим на него PostgreSQL 15 с дефолтными настройками:
```shell
yc-user@otus-vacum:~$ sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
yc-user@otus-vacum:~$ wget -qO- https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo tee /etc/apt/trusted.gpg.d/pgdg.asc &>/dev/null
yc-user@otus-vacum:~$ sudo apt update -y
yc-user@otus-vacum:~$ sudo apt install -y postgresql-15

yc-user@otus-vacum:~$ sudo pg_createcluster 15 vacum
Creating new PostgreSQL cluster 15/vacum ...
/usr/lib/postgresql/15/bin/initdb -D /var/lib/postgresql/15/vacum --auth-local peer --auth-host scram-sha-256 --no-instructions
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locale "en_US.UTF-8".
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are disabled.

fixing permissions on existing directory /var/lib/postgresql/15/vacum ... ok
creating subdirectories ... ok
selecting dynamic shared memory implementation ... posix
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting default time zone ... Etc/UTC
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok
Ver Cluster Port Status Owner    Data directory               Log file
15  vacum   5432 down   postgres /var/lib/postgresql/15/vacum /var/log/postgresql/postgresql-15-vacum.log
yc-user@otus-vacum:~$ sudo pg_ctlcluster 15 vacum start
yc-user@otus-vacum:~$ sudo pg_ctlcluster 15 vacum status
pg_ctl: server is running (PID: 14472)
/usr/lib/postgresql/15/bin/postgres "-D" "/var/lib/postgresql/15/vacum" "-c" "config_file=/etc/postgresql/15/vacum/postgresql.conf"
```
#### Создадим БД для тестов и  выполним _**pgbench -i postgres**_
Заходим
```shell
yc-user@otus-vacum:~$ sudo -u postgres psql
could not change directory to "/home/yc-user": Permission denied
psql (15.8 (Ubuntu 15.8-1.pgdg22.04+1))
Type "help" for help.
```
Создаём
```postgresql
postgres=# CREATE DATABASE testdb;
CREATE DATABASE
postgres=# \l
                                                 List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    | ICU Locale | Locale Provider |   Access privileges
-----------+----------+----------+-------------+-------------+------------+-----------------+-----------------------
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            |
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            | =c/postgres          +
           |          |          |             |             |            |                 | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            | =c/postgres          +
           |          |          |             |             |            |                 | postgres=CTc/postgres
 testdb    | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            |
(4 rows)
```
Проверим занимаемо место в системе wal-журнала
```shell
postgres@otus-vacum:~$ du -h /var/lib/postgresql/15/vacum/pg_wal
4.0K    /var/lib/postgresql/15/vacum/pg_wal/archive_status
17M     /var/lib/postgresql/15/vacum/pg_wal
```
Тестируем и посмотрим 
```shell
postgres@otus-vacum:~$ pgbench -i testdb
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.09 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.67 s (drop tables 0.00 s, create tables 0.02 s, client-side generate 0.46 s, vacuum 0.06 s, primary keys 0.14 s).
postgres@otus-vacum:~$ du -h /var/lib/postgresql/15/vacum/pg_wal
4.0K    /var/lib/postgresql/15/vacum/pg_wal/archive_status
33M     /var/lib/postgresql/15/vacum/pg_wal
```
#### Запустим _**pgbench -c8 -P 6 -T 60 -U postgres postgres**_
````shell
postgres@otus-vacum:~$ pgbench -c8 -P 6 -T 60 -U postgres testdb
pgbench (15.8 (Ubuntu 15.8-1.pgdg22.04+1))
starting vacuum...end.
progress: 6.0 s, 445.2 tps, lat 17.885 ms stddev 13.103, 0 failed
progress: 12.0 s, 385.0 tps, lat 20.767 ms stddev 13.207, 0 failed
progress: 18.0 s, 439.7 tps, lat 18.181 ms stddev 13.899, 0 failed
progress: 24.0 s, 245.7 tps, lat 31.744 ms stddev 19.974, 0 failed
progress: 30.0 s, 426.0 tps, lat 19.243 ms stddev 17.288, 0 failed
progress: 36.0 s, 401.8 tps, lat 19.895 ms stddev 15.912, 0 failed
progress: 42.0 s, 452.2 tps, lat 17.715 ms stddev 11.064, 0 failed
progress: 48.0 s, 436.5 tps, lat 18.323 ms stddev 14.996, 0 failed
progress: 54.0 s, 238.3 tps, lat 33.530 ms stddev 23.919, 0 failed
progress: 60.0 s, 393.2 tps, lat 20.382 ms stddev 15.288, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 23189
number of failed transactions: 0 (0.000%)
latency average = 20.694 ms
latency stddev = 16.251 ms
initial connection time = 22.967 ms
tps = 386.503489 (without initial connection time)
````
#### Применим параметры настройки PostgreSQL из прикрепленного к материалам занятия файла и протестируем заново:
```markdown
*  DB Version: 11
*  OS Type: linux
*  DB Type: dw
*  Total Memory (RAM): 4 GB
*  CPUs num: 1
*  Data Storage: hdd

max_connections = 40 
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 512MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 500
random_page_cost = 4
effective_io_concurrency = 2
work_mem = 6553kB
min_wal_size = 4GB
max_wal_size = 16GB
```
Параметры по умолчанию:
```postgresql
SELECT * FROM pg_file_settings ;
                sourcefile                | sourceline | seqno |            name            |                setting                 | applied | error
------------------------------------------+------------+-------+----------------------------+----------------------------------------+---------+-------
 /etc/postgresql/15/vacum/postgresql.conf |         42 |     1 | data_directory             | /var/lib/postgresql/15/vacum           | t       |
 /etc/postgresql/15/vacum/postgresql.conf |         44 |     2 | hba_file                   | /etc/postgresql/15/vacum/pg_hba.conf   | t       |
 /etc/postgresql/15/vacum/postgresql.conf |         46 |     3 | ident_file                 | /etc/postgresql/15/vacum/pg_ident.conf | t       |
 /etc/postgresql/15/vacum/postgresql.conf |         50 |     4 | external_pid_file          | /var/run/postgresql/15-vacum.pid       | t       |
 /etc/postgresql/15/vacum/postgresql.conf |         64 |     5 | port                       | 5432                                   | t       |
 /etc/postgresql/15/vacum/postgresql.conf |         65 |     6 | max_connections            | 100                                    | t       |
 /etc/postgresql/15/vacum/postgresql.conf |         67 |     7 | unix_socket_directories    | /var/run/postgresql                    | t       |
 /etc/postgresql/15/vacum/postgresql.conf |        105 |     8 | ssl                        | on                                     | t       |
 /etc/postgresql/15/vacum/postgresql.conf |        107 |     9 | ssl_cert_file              | /etc/ssl/certs/ssl-cert-snakeoil.pem   | t       |
 /etc/postgresql/15/vacum/postgresql.conf |        110 |    10 | ssl_key_file               | /etc/ssl/private/ssl-cert-snakeoil.key | t       |
 /etc/postgresql/15/vacum/postgresql.conf |        127 |    11 | shared_buffers             | 128MB                                  | t       |
 /etc/postgresql/15/vacum/postgresql.conf |        150 |    12 | dynamic_shared_memory_type | posix                                  | t       |
 /etc/postgresql/15/vacum/postgresql.conf |        241 |    13 | max_wal_size               | 1GB                                    | t       |
 /etc/postgresql/15/vacum/postgresql.conf |        242 |    14 | min_wal_size               | 80MB                                   | t       |
 /etc/postgresql/15/vacum/postgresql.conf |        559 |    15 | log_line_prefix            | %m [%p] %q%u@%d                        | t       |
 /etc/postgresql/15/vacum/postgresql.conf |        597 |    16 | log_timezone               | Etc/UTC                                | t       |
 /etc/postgresql/15/vacum/postgresql.conf |        604 |    17 | cluster_name               | 15/vacum                               | t       |
 /etc/postgresql/15/vacum/postgresql.conf |        711 |    18 | datestyle                  | iso, mdy                               | t       |
 /etc/postgresql/15/vacum/postgresql.conf |        713 |    19 | timezone                   | Etc/UTC                                | t       |
 /etc/postgresql/15/vacum/postgresql.conf |        727 |    20 | lc_messages                | en_US.UTF-8                            | t       |
 /etc/postgresql/15/vacum/postgresql.conf |        729 |    21 | lc_monetary                | en_US.UTF-8                            | t       |
 /etc/postgresql/15/vacum/postgresql.conf |        730 |    22 | lc_numeric                 | en_US.UTF-8                            | t       |
 /etc/postgresql/15/vacum/postgresql.conf |        731 |    23 | lc_time                    | en_US.UTF-8                            | t       |
 /etc/postgresql/15/vacum/postgresql.conf |        734 |    24 | default_text_search_config | pg_catalog.english                     | t       |
(24 rows)
```
Сменим параметры кластера БД: 
```postgresql
postgres=# ALTER SYSTEM SET max_connections = '40';
ALTER SYSTEM
postgres=# ALTER SYSTEM SET shared_buffers = '1GB';
ALTER SYSTEM
postgres=# ALTER SYSTEM SET effective_cache_size = '3GB';
ALTER SYSTEM
postgres=# ALTER SYSTEM SET maintenance_work_mem = '512MB';
ALTER SYSTEM
postgres=# ALTER SYSTEM SET checkpoint_completion_target = '0.9';
ALTER SYSTEM
postgres=# ALTER SYSTEM SET wal_buffers = '16MB';
ALTER SYSTEM
postgres=# ALTER SYSTEM SET default_statistics_target = 500;
ALTER SYSTEM
postgres=# ALTER SYSTEM SET random_page_cost = 4;
ALTER SYSTEM
postgres=# ALTER SYSTEM SET effective_io_concurrency = 2;
ALTER SYSTEM
postgres=# ALTER SYSTEM SET work_mem = '6553kB';
ALTER SYSTEM
postgres=# ALTER SYSTEM SET min_wal_size = '4GB';
ALTER SYSTEM
postgres=# ALTER SYSTEM SET max_wal_size = '16GB';
ALTER SYSTEM
```
> Что изменилось и почему?
> > Протестируем заново _*pgbench -c8 -P 6 -T 60 -U postgres testdb
> >````shell
> >yc-user@otus-vacum:~$ sudo pg_ctlcluster 15 vacum restart
> >yc-user@otus-vacum:~$ sudo -i -u postgres bash
> >postgres@otus-vacum:~$ pgbench -c8 -P 6 -T 60 -U postgres testdb
> >pgbench (15.8 (Ubuntu 15.8-1.pgdg22.04+1))
> >starting vacuum...end.
> >progress: 6.0 s, 379.5 tps, lat 20.956 ms stddev 14.833, 0 failed
> >progress: 12.0 s, 334.7 tps, lat 23.813 ms stddev 17.399, 0 failed
> >progress: 18.0 s, 266.5 tps, lat 30.085 ms stddev 19.852, 0 failed
> >progress: 24.0 s, 437.8 tps, lat 18.311 ms stddev 13.268, 0 failed
> >progress: 30.0 s, 400.0 tps, lat 19.992 ms stddev 15.924, 0 failed
> >progress: 36.0 s, 386.2 tps, lat 20.718 ms stddev 14.155, 0 failed
> >progress: 42.0 s, 499.2 tps, lat 15.963 ms stddev 10.152, 0 failed
> >progress: 48.0 s, 258.7 tps, lat 31.033 ms stddev 18.707, 0 failed
> >progress: 54.0 s, 417.2 tps, lat 19.172 ms stddev 13.220, 0 failed
> >progress: 60.0 s, 356.7 tps, lat 22.379 ms stddev 14.710, 0 failed
> >transaction type: <builtin: TPC-B (sort of)>
> >scaling factor: 1
> >query mode: simple
> >number of clients: 8
> >number of threads: 1
> >maximum number of tries: 1
> >duration: 60 s
> >number of transactions actually processed: 22426
> >number of failed transactions: 0 (0.000%)
> >latency average = 21.399 ms
> >latency stddev = 15.578 ms
> >initial connection time = 23.391 ms
> >tps = 373.767601 (without initial connection time)
> >````
> >Скорость работы кластера не изменилось, похоже упираемся в производительность дисковой подсистемы в YC. Если не облачный сервер, прирост был-бы заметен. 
#### Создадим таблицу с текстовым полем и заполним случайными данным в размере 1млн строк:
```postgresql
postgres@otus-vacum:~$ psql -b testdb
psql (15.8 (Ubuntu 15.8-1.pgdg22.04+1))
Type "help" for help.

testdb=# \dt
              List of relations
 Schema |       Name       | Type  |  Owner
--------+------------------+-------+----------
 public | pgbench_accounts | table | postgres
 public | pgbench_branches | table | postgres
 public | pgbench_history  | table | postgres
 public | pgbench_tellers  | table | postgres
(4 rows)

testdb=# CREATE TABLE t1 (i text);
CREATE TABLE
testdb=# \dt
              List of relations
 Schema |       Name       | Type  |  Owner
--------+------------------+-------+----------
 public | pgbench_accounts | table | postgres
 public | pgbench_branches | table | postgres
 public | pgbench_history  | table | postgres
 public | pgbench_tellers  | table | postgres
 public | t1               | table | postgres
(5 rows)
                                   
testdb=# INSERT INTO t1 SELECT random()::text FROM generate_series(1,1000000);
INSERT 0 1000000

```
#### Посмотрим размер файла с таблицей _**SELECT pg_size_pretty(pg_table_size('t1'));**_
```postgresql
testdb=# SELECT pg_size_pretty(pg_table_size('t1'));
 pg_size_pretty
----------------
 50 MB
(1 row)
```
#### 5 раз обновим все строчки и добавим к каждой строчке любой символ :
```postgresql
testdb=# UPDATE t1 SET i=random()::TEXT || 'v';
UPDATE 1000000
testdb=# UPDATE t1 SET i=random()::TEXT || 'a';
UPDATE 1000000
testdb=# UPDATE t1 SET i=random()::TEXT || 'c';
UPDATE 1000000
testdb=# UPDATE t1 SET i=random()::TEXT || 'u';
UPDATE 1000000
testdb=# UPDATE t1 SET i=random()::TEXT || 'm';
UPDATE 1000000
```
#### Посмотрим количество мертвых строчек в таблице и когда последний раз приходил **AUTOVACUUM**
```postgresql
testdb=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 't1';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
---------+------------+------------+--------+-------------------------------
 t1      |    1000000 |    1000000 |     99 | 2024-09-22 14:18:48.440698+00
(1 row)
```
> > Ждём!
#### Ждем некоторое время и проверяя, прошел ли **AUTOVACUUM**:
```postgresql
testdb=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 't1';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
---------+------------+------------+--------+-------------------------------
 t1      |    1000000 |          0 |      0 | 2024-09-22 14:19:47.793924+00
(1 row)
```
> > **AUTOVACUM** прошел! 
#### 5 раз обновим все строчки и добавим к каждой строчке любой символ:
```postgresql
testdb=# UPDATE t1 SET i=random()::TEXT || 'm';
UPDATE 1000000
testdb=# UPDATE t1 SET i=random()::TEXT || 'u';
UPDATE 1000000
testdb=# UPDATE t1 SET i=random()::TEXT || 'c';
UPDATE 1000000
testdb=# UPDATE t1 SET i=random()::TEXT || 'a';
UPDATE 1000000
testdb=# UPDATE t1 SET i=random()::TEXT || 'v';
UPDATE 1000000
```
#### Посмотреть размер файла с таблицей:
```postgresql
testdb=# SELECT pg_size_pretty(pg_table_size('t1'));
 pg_size_pretty
----------------
 299 MB
(1 row)
```
#### Отключим _AUTOVACUUM_ в таблице **t1**:
````postgresql
testdb=# ALTER TABLE t1 SET (autovacuum_enabled = off);
ALTER TABLE
`````
#### 10 раз обновим все строчки и добавим к каждой строчке любой символ и посмотрим размер файла с таблицей:
```postgresql
testdb=# UPDATE t1 SET i=random()::TEXT || 'a';
UPDATE t1 SET i=random()::TEXT || 'u';
UPDATE t1 SET i=random()::TEXT || 't';
UPDATE t1 SET i=random()::TEXT || 'o';
UPDATE t1 SET i=random()::TEXT || 'v';
UPDATE t1 SET i=random()::TEXT || 'a';
UPDATE t1 SET i=random()::TEXT || 'c';
UPDATE t1 SET i=random()::TEXT || 'u';
UPDATE t1 SET i=random()::TEXT || 'u';
UPDATE t1 SET i=random()::TEXT || 'm';
UPDATE 1000000
UPDATE 1000000
UPDATE 1000000
UPDATE 1000000
UPDATE 1000000
UPDATE 1000000
UPDATE 1000000
UPDATE 1000000
UPDATE 1000000
UPDATE 1000000

testdb=# SELECT pg_size_pretty(pg_table_size('t1'));
 pg_size_pretty
----------------
 547 MB
(1 row)
```
> Объясните полученный результат.
>> Потому-что при UPDATE запись с начало удаляется (делается delete) в таблице, а потом добавляется новая запись (делается insert). 
> > Так как AUTOVACUUM отключён, чистить таблицы некому, а так же нет сбора статистики о распределении данных. 
> > Появляются мусорные данные и таблица растёт в размерах.

Включим AUTOVACUUM: 
```postgresql
testdb=# ALTER TABLE t1 SET (autovacuum_enabled = on);
ALTER TABLE
testdb=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 't1';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
---------+------------+------------+--------+-------------------------------
 t1      |    1000000 |          0 |      0 | 2024-09-22 15:44:54.477651+00
(1 row)
```
#### Задание со *:
> * Написать анонимную процедуру, в которой в цикле 10 раз обновятся все строчки в искомой таблице. 
> * Не забыть вывести номер шага цикла.
> > ```sql
> > testdb=# CREATE PROCEDURE procedure_update_data() AS $$
> >BEGIN
> >  FOR t IN 1..10 LOOP
> >    UPDATE t1 SET i = random()::TEXT || 'autovacuum';
> >  END LOOP;
> >END;
> >$$ LANGUAGE plpgsql;
> >CREATE PROCEDURE
> >```