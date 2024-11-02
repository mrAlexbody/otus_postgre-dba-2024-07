# Домашнее задание 10
## Репликация

### Цель: реализовать свой миникластер на 3 ВМ.

**Описание/Пошаговая инструкция выполнения домашнего задания:**
1. На 1 ВМ создаем таблицы test для записи, test2 для запросов на чтение.
2. Создаем публикацию таблицы test и подписываемся на публикацию таблицы test2 с ВМ №2.
3. На 2 ВМ создаем таблицы test2 для записи, test для запросов на чтение.
4. Создаем публикацию таблицы test2 и подписываемся на публикацию таблицы test1 с ВМ №1.
5. 3 ВМ использовать как реплику для чтения и бэкапов (подписаться на таблицы из ВМ №1 и №2 ).
* реализовать горячее реплицирование для высокой доступности на 4ВМ. Источником должна выступать ВМ №3. Написать с какими проблемами столкнулись.

_**ДЗ сдается в виде миниотчета на гитхабе с описанием шагов и с какими проблемами столкнулись.**_

---
Создадим подсеть в YC:
```shell
C:\>yc vpc network create --name otus-net --description "otus-net" && \
C:\>yc vpc subnet create --name otus-subnet --range 192.168.100.0/24 --network-name otus-net --description "otus-subnet" 
```
Установка 3 ВМ:
```shell
C:\>yc compute instance create --name db-01 --hostname db-01 --cores 2 --memory 4 --create-boot-disk size=15G,type=network-hdd,image-folder-id=standard-images,image-family=ubuntu-2204-lts --network-interface subnet-name=otus-subnet,nat-ip-version=ipv4 --ssh-key C:\Users\Alexander/.ssh/yc_key.pub
done (41s)
C:\>yc compute instance create --name db-02 --hostname db-01 --cores 2 --memory 4 --create-boot-disk size=15G,type=network-hdd,image-folder-id=standard-images,image-family=ubuntu-2204-lts --network-interface subnet-name=otus-subnet,nat-ip-version=ipv4 --ssh-key C:\Users\Alexander/.ssh/yc_key.pub
done (41s)
C:\>yc compute instance create --name db-03 --hostname db-01 --cores 2 --memory 4 --create-boot-disk size=15G,type=network-hdd,image-folder-id=standard-images,image-family=ubuntu-2204-lts --network-interface subnet-name=otus-subnet,nat-ip-version=ipv4 --ssh-key C:\Users\Alexander/.ssh/yc_key.pub
done (41s)
C:\>yc compute instances list
+----------------------+-------+---------------+---------+----------------+----------------+
|          ID          | NAME  |    ZONE ID    | STATUS  |  EXTERNAL IP   |  INTERNAL IP   |
+----------------------+-------+---------------+---------+----------------+----------------+
| fhm0gu479bf2ffjtv1iv | db-03 | ru-central1-a | RUNNING | 89.169.143.51  | 192.168.100.30 |
| fhm8r5hpohfba63l91ht | db-02 | ru-central1-a | RUNNING | 51.250.74.163  | 192.168.100.8  |
| fhmvdijccvfbhlvedmij | db-01 | ru-central1-a | RUNNING | 89.169.155.100 | 192.168.100.17 |
+----------------------+-------+---------------+---------+----------------+----------------+

```
Установим на всех ВМ PostgreSQL и добавим :
```shell
root@db-*:~# apt install postgresql
````
```shell

root@db-01:~# pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log

root@db-02:~# pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log

root@db-03:~# pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
```
```postgresql

postgres=# select * from pg_hba_file_rules;
 line_number | type  |   database    | user_name  |    address    |                 netmask                 |  auth_method  | options | error
-------------+-------+---------------+------------+---------------+-----------------------------------------+---------------+---------+-------
          90 | local | {all}         | {postgres} |               |                                         | peer          |         |
          95 | local | {all}         | {all}      |               |                                         | peer          |         |
          97 | host  | {all}         | {all}      | 127.0.0.1     | 255.255.255.255                         | scram-sha-256 |         |
          98 | host  | {all}         | {all}      | 192.168.100.0 | 255.255.255.0                           | scram-sha-256 |         |
         100 | host  | {all}         | {all}      | ::1           | ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff | scram-sha-256 |         |
         103 | local | {replication} | {all}      |               |                                         | peer          |         |
         104 | host  | {replication} | {all}      | 127.0.0.1     | 255.255.255.255                         | scram-sha-256 |         |
         105 | host  | {replication} | {all}      | 192.168.100.0 | 255.255.255.0                           | scram-sha-256 |         |
         106 | host  | {replication} | {all}      | ::1           | ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff | scram-sha-256 |         |
(9 rows)

 otus_db=# SHOW wal_level;
 wal_level
-----------
 logical
(1 row)

```

Создадим на 1ВМ таблицу **_test**_ и _**test2**_:
```postgresql
yc-user@db-01:~$ sudo -i -u postgres psql
psql (14.13 (Ubuntu 14.13-0ubuntu0.22.04.1))
Type "help" for help.

postgres=# CREATE DATABASE otus_db;
CREATE DATABASE
postgres=# \l
postgres=# \c otus_db
You are now connected to database "otus_db" as user "postgres".

otus_db=# CREATE TABLE test AS SELECT generate_series(1, 10) AS id, md5(random()::text)::char(10) AS fio;
CREATE TABLE

    otus_db=# CREATE TABLE test2  (id SERIAL PRIMARY KEY, fio char(100));
CREATE TABLE

otus_db=# select * from test;
 id |    fio
----+------------
  1 | 671acc6628
  2 | b3713873d8
  3 | 4633efbe5d
  4 | 49478e2c9c
  5 | 6909c3763c
  6 | fffc446f0c
  7 | f205b8173f
  8 | 8676fc1522
  9 | 6cd1863ca1
 10 | 958e96d92a
(10 rows)

```
Создадим на 2ВМ таблицу **_test**_ и _**test2**_:
```postgresql
yc-user@db-01:~$ sudo -i -u postgres psql
psql (14.13 (Ubuntu 14.13-0ubuntu0.22.04.1))
Type "help" for help.

postgres=# CREATE DATABASE otus_db;
CREATE DATABASE
postgres=# \l
postgres=# \c otus_db
You are now connected to database "otus_db" as user "postgres".

otus_db=# CREATE TABLE test2 AS SELECT generate_series(1, 10) AS id, md5(random()::text)::char(10) AS fio;
CREATE TABLE

otus_db=# CREATE TABLE test  (id SERIAL PRIMARY KEY, fio char(100));
CREATE TABLE

otus_db=# select * from test2;
 id |    fio
----+------------
  1 | 2bcc8c62aa
  2 | d1ee349038
  3 | a1f77f2090
  4 | 7b2573341f
  5 | 06f169f15b
  6 | 9a1e54a8a5
  7 | 5d6c70ba50
  8 | e4fb503bfe
  9 | 3e9ed06a51
 10 | ce33f84584
(10 rows)
```

Сделаем публикацию и подписку на 1ВМ (192.168.100.17):
```postgresql
otus_db=# create subscription test2_sub connection 'host=192.168.100.8 port=5432 user=postgres password=postgres dbname=otus_db' publication test2_pub;
NOTICE:  created replication slot "test2_sub" on publisher
CREATE SUBSCRIPTION

otus_db=# \dRs+
                                                                        List of subscriptions
   Name    |  Owner   | Enabled | Publication | Binary | Streaming | Synchronous commit |                                  Conninfo
-----------+----------+---------+-------------+--------+-----------+--------------------+-----------------------------------------------------------------------------
 test2_sub | postgres | t       | {test2_pub} | f      | f         | off                | host=192.168.100.8 port=5432 user=postgres password=postgres dbname=otus_db
(1 row)
    
otus_db=# select * from test2;
 id |                                                 fio
----+------------------------------------------------------------------------------------------------------
  1 | 2bcc8c62aa
  2 | d1ee349038
  3 | a1f77f2090
  4 | 7b2573341f
  5 | 06f169f15b
  6 | 9a1e54a8a5
  7 | 5d6c70ba50
  8 | e4fb503bfe
  9 | 3e9ed06a51
 10 | ce33f84584
(10 rows)

otus_db=# create publication test1_pub for table test;

otus_db=# \dRp
                                  List of publications
   Name    |  Owner   | All tables | Inserts | Updates | Deletes | Truncates | Via root
-----------+----------+------------+---------+---------+---------+-----------+----------
 test1_pub | postgres | f          | t       | t       | t       | t         | f
(1 row)
```

Сделаем публикацию и подписку на 2ВМ (192.168.100.8):
```postgresql
otus_db=# create subscription test1_sub connection 'host=192.168.100.17 port=5432 user=postgres password=postgres dbname=otus_db' publication test1_pub;
NOTICE:  created replication slot "test1_sub" on publisher
CREATE SUBSCRIPTION
otus_db=# \dRp+
                           Publication test2_pub
  Owner   | All tables | Inserts | Updates | Deletes | Truncates | Via root
----------+------------+---------+---------+---------+-----------+----------
 postgres | f          | t       | t       | t       | t         | f
Tables:
    "public.test2"

otus_db=# \dRs+
                                                                         List of subscriptions
   Name    |  Owner   | Enabled | Publication | Binary | Streaming | Synchronous commit |                                   Conninfo
-----------+----------+---------+-------------+--------+-----------+--------------------+------------------------------------------------------------------------------
 test1_sub | postgres | t       | {test1_pub} | f      | f         | off                | host=192.168.100.17 port=5432 user=postgres password=postgres dbname=otus_db
(1 row)

otus_db=# select * from test;
 id |    fio
----+------------
  1 | 671acc6628
  2 | b3713873d8
  3 | 4633efbe5d
  4 | 49478e2c9c
  5 | 6909c3763c
  6 | fffc446f0c
  7 | f205b8173f
  8 | 8676fc1522
  9 | 6cd1863ca1
 10 | 958e96d92a
(10 rows)
       
otus_db=# create publication test2_pub for table test2;
```
