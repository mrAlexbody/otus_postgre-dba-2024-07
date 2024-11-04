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
C:\>yc compute instance create --name db-02 --hostname db-02 --cores 2 --memory 4 --create-boot-disk size=15G,type=network-hdd,image-folder-id=standard-images,image-family=ubuntu-2204-lts --network-interface subnet-name=otus-subnet,nat-ip-version=ipv4 --ssh-key C:\Users\Alexander/.ssh/yc_key.pub
done (41s)
C:\>yc compute instance create --name db-03 --hostname db-03 --cores 2 --memory 4 --create-boot-disk size=15G,type=network-hdd,image-folder-id=standard-images,image-family=ubuntu-2204-lts --network-interface subnet-name=otus-subnet,nat-ip-version=ipv4 --ssh-key C:\Users\Alexander/.ssh/yc_key.pub
done (41s)
C:\>yc compute instance create --name db-04 --hostname db-04 --cores 2 --memory 4 --create-boot-disk size=15G,type=network-hdd,image-folder-id=standard-images,image-family=ubuntu-2204-lts --network-interface subnet-name=otus-subnet,nat-ip-version=ipv4 --ssh-key C:\Users\Alexander/.ssh/yc_key.pub
done (36s)

C:\>yc compute instances list
+----------------------+-------+---------------+---------+----------------+----------------+
|          ID          | NAME  |    ZONE ID    | STATUS  |  EXTERNAL IP   |  INTERNAL IP   |
+----------------------+-------+---------------+---------+----------------+----------------+
| fhm0gu479bf2ffjtv1iv | db-03 | ru-central1-a | RUNNING | 89.169.134.110 | 192.168.100.30 |
| fhm8r5hpohfba63l91ht | db-02 | ru-central1-a | RUNNING | 89.169.158.97  | 192.168.100.8  |
| fhmqu1t2chr1v4n12v39 | db-04 | ru-central1-a | RUNNING | 89.169.138.16  | 192.168.100.26 |
| fhmvdijccvfbhlvedmij | db-01 | ru-central1-a | RUNNING | 89.169.140.41  | 192.168.100.17 |
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

otus_db=# CREATE TABLE test AS SELECT generate_series(1, 10) AS id, md5(random()::text)::char(100) AS fio;
CREATE TABLE

otus_db=# CREATE TABLE test2  (id SERIAL PRIMARY KEY, fio char(100));
CREATE TABLE

otus_db=# SELECT * FROM test;
  id |                                                 fio
----+------------------------------------------------------------------------------------------------------
  1 | d20548f8cc92acd196f8e4af49352124
  2 | dfef47531caa952a2fc87b7964b3655f
  3 | 15417bfdab1be36583b1f1eb7e96211e
  4 | 626ab7e1cf261e27d0a3ceb5e74f0368
  5 | 60e6a0c021f2604bf9d08c8c079b596f
  6 | 5a0a4cf12b0e90ac9e1ebbea19700788
  7 | 6acd6b46c7892ed85f09a1c6d4e81582
  8 | 7e55ac7c30ede484237d858fa13fb95e
  9 | 4dd9f245eb2da094affaf5c008a2e5ff
 10 | 5419f552511f67afe71458a7e9607c9e
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

otus_db=# CREATE TABLE test2 AS SELECT generate_series(1, 10) AS id, md5(random()::text)::char(100) AS fio;
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

otus_db=# SELECT * FROM test;
  id |                                                 fio
----+------------------------------------------------------------------------------------------------------
  1 | d20548f8cc92acd196f8e4af49352124
  2 | dfef47531caa952a2fc87b7964b3655f
  3 | 15417bfdab1be36583b1f1eb7e96211e
  4 | 626ab7e1cf261e27d0a3ceb5e74f0368
  5 | 60e6a0c021f2604bf9d08c8c079b596f
  6 | 5a0a4cf12b0e90ac9e1ebbea19700788
  7 | 6acd6b46c7892ed85f09a1c6d4e81582
  8 | 7e55ac7c30ede484237d858fa13fb95e
  9 | 4dd9f245eb2da094affaf5c008a2e5ff
 10 | 5419f552511f67afe71458a7e9607c9e
(10 rows)
       
otus_db=# create publication test2_pub for table test2;
```
На 3 ВМ просто сделаем подписки: 
```postgresql
otus_db=# CREATE TABLE test  (id SERIAL PRIMARY KEY, fio char(100));
otus_db=# CREATE TABLE test2  (id SERIAL PRIMARY KEY, fio char(100));

otus_db=# CREATE  SUBSCRIPTION test13_sub CONNECTION 'host=192.168.100.17 port=5432 user=postgres password=postgres dbname=otus_db' PUBLICATION test1_pub;
otus_db=# CREATE SUBSCRIPTION test23_sub CONNECTION 'host=192.168.100.8 port=5432 user=postgres password=postgres dbname=otus_db' PUBLICATION test2_pub;

otus_db=# \dRs
             List of subscriptions
    Name    |  Owner   | Enabled | Publication
------------+----------+---------+-------------
 test13_sub | postgres | t       | {test1_pub}
 test23_sub | postgres | t       | {test2_pub}
(2 rows)

otus_db=# SELECT * FROM test;
 id |                                                 fio
----+------------------------------------------------------------------------------------------------------
  1 | d20548f8cc92acd196f8e4af49352124
  2 | dfef47531caa952a2fc87b7964b3655f
  3 | 15417bfdab1be36583b1f1eb7e96211e
  4 | 626ab7e1cf261e27d0a3ceb5e74f0368
  5 | 60e6a0c021f2604bf9d08c8c079b596f
  6 | 5a0a4cf12b0e90ac9e1ebbea19700788
  7 | 6acd6b46c7892ed85f09a1c6d4e81582
  8 | 7e55ac7c30ede484237d858fa13fb95e
  9 | 4dd9f245eb2da094affaf5c008a2e5ff
 10 | 5419f552511f67afe71458a7e9607c9e
(10 rows)

otus_db=# SELECT * FROM test2;
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
```
#### * реализовать горячее реплицирование для высокой доступности на 4ВМ. Источником должна выступать ВМ №3. Написать с какими проблемами столкнулись.
```shell
yc-user@db-04:~$ sudo apt install postgresql
yc-user@db-04:~$ sudo -i -u postgres bash
postgres@db-04:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log

root@db-04:~# sudo systemctl stop postgresql@14-main
root@db-04:~#  pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 down   postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log

root@db-04:~# rm -rf /var/lib/postgresql/14/main/*
root@db-04:/var/lib/postgresql/14/main# ls -la
total 8
drwx------ 2 postgres postgres 4096 Nov  4 12:50 .
drwxr-xr-x 3 postgres postgres 4096 Nov  4 12:42 ..
root@db-04:/var/lib/postgresql/14/main# exit
logout
yc-user@db-04:~$ sudo -i -u postgres bash
postgres@db-04:~$ ls -la /var/lib/postgresql/14/main
total 8
drwx------ 2 postgres postgres 4096 Nov  4 12:50 .
drwxr-xr-x 3 postgres postgres 4096 Nov  4 12:42 ..
postgres@db-04:~$ pg_basebackup -R -D  /var/lib/postgresql/14/main -h 192.168.100.30 -W
Password:
postgres@db-04:~$ pg_lsclusters
Ver Cluster Port Status        Owner    Data directory              Log file
14  main    5432 down,recovery postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
postgres@db-04:~$ exit
exit
yc-user@db-04:~$ sudo -i
root@db-04:~#  sudo systemctl start postgresql@14-main
root@db-04:~#  sudo systemctl status postgresql@14-main
● postgresql@14-main.service - PostgreSQL Cluster 14-main
     Loaded: loaded (/lib/systemd/system/postgresql@.service; enabled-runtime; vendor preset: enabled)
     Active: active (running) since Mon 2024-11-04 12:53:04 UTC; 10s ago
    Process: 3873 ExecStart=/usr/bin/pg_ctlcluster --skip-systemctl-redirect 14-main start (code=exited, status=0/SUCCESS)
   Main PID: 3878 (postgres)
      Tasks: 6 (limit: 4564)
     Memory: 33.8M
        CPU: 198ms
     CGroup: /system.slice/system-postgresql.slice/postgresql@14-main.service
             ├─3878 /usr/lib/postgresql/14/bin/postgres -D /var/lib/postgresql/14/main -c config_file=/etc/postgresql/14/main/postgresql.conf
             ├─3879 "postgres: 14/main: startup recovering 000000010000000000000003" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" ""
             ├─3880 "postgres: 14/main: checkpointer " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "">
             ├─3881 "postgres: 14/main: background writer " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" >
             ├─3882 "postgres: 14/main: stats collector " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "">
             └─3883 "postgres: 14/main: walreceiver streaming 0/3000060" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" ""

Nov 04 12:53:02 db-04 systemd[1]: Starting PostgreSQL Cluster 14-main...
Nov 04 12:53:04 db-04 systemd[1]: Started PostgreSQL Cluster 14-main.
root@db-04:~# pg_lsclusters
Ver Cluster Port Status          Owner    Data directory              Log file
14  main    5432 online,recovery postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
```
Проверим на 4ВМ:
```postgresql
yc-user@db-04:~$ sudo -i -u postgres psql
psql (14.13 (Ubuntu 14.13-0ubuntu0.22.04.1))
Type "help" for help.

postgres-# \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges
-----------+----------+----------+-------------+-------------+-----------------------
 otus_db   | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(4 rows)

postgres-# \c otus_db
You are now connected to database "otus_db" as user "postgres".
otus_db-# \dt
         List of relations
 Schema | Name  | Type  |  Owner
--------+-------+-------+----------
 public | test  | table | postgres
 public | test2 | table | postgres
(2 rows)

otus_db=# SELECT * FROM test;
 id |                                                 fio
----+------------------------------------------------------------------------------------------------------
  1 | d20548f8cc92acd196f8e4af49352124
  2 | dfef47531caa952a2fc87b7964b3655f
  3 | 15417bfdab1be36583b1f1eb7e96211e
  4 | 626ab7e1cf261e27d0a3ceb5e74f0368
  5 | 60e6a0c021f2604bf9d08c8c079b596f
  6 | 5a0a4cf12b0e90ac9e1ebbea19700788
  7 | 6acd6b46c7892ed85f09a1c6d4e81582
  8 | 7e55ac7c30ede484237d858fa13fb95e
  9 | 4dd9f245eb2da094affaf5c008a2e5ff
 10 | 5419f552511f67afe71458a7e9607c9e
(10 rows)

otus_db=# SELECT * FROM test2;
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
                        
```
Проверим на 3ВМ:
```postgresql
yc-user@db-03:~$ sudo -i -u postgres psql
psql (14.13 (Ubuntu 14.13-0ubuntu0.22.04.1))
Type "help" for help.

postgres=# SELECT * FROM pg_stat_replication \gx
-[ RECORD 1 ]----+------------------------------
pid              | 1321
usesysid         | 10
usename          | postgres
application_name | 14/main
client_addr      | 192.168.100.26
client_hostname  |
client_port      | 38998
backend_start    | 2024-11-04 12:53:02.68543+00
backend_xmin     |
state            | streaming
sent_lsn         | 0/3000148
write_lsn        | 0/3000148
flush_lsn        | 0/3000148
replay_lsn       | 0/3000148
write_lag        |
flush_lag        |
replay_lag       |
sync_priority    | 0
sync_state       | async
reply_time       | 2024-11-04 12:56:25.169043+00

postgres=#
```
