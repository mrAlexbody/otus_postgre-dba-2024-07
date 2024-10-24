
# Домашнее задание 04
## Работа с базами данных, пользователями и правами

### Цель:
* Создание новой базы данных, схемы и таблицы
* Создание роли для чтения данных из созданной схемы созданной базы данных
* Создание роли для чтения и записи из созданной схемы созданной базы данных

### Описание/Пошаговая инструкция выполнения домашнего задания:
1. Cоздайте новый кластер **PostgresSQL 14**
2. Зайдите в созданный кластер под пользователем **_postgres_**
3. Создайте новую базу данных _**testdb**_
4. Зайдите в созданную базу данных под пользователем _**postgres**_
5. Создайте новую схему _**testnm**_
6. Создайте новую таблицу _**t1**_ с одной колонкой _**c1**_ типа integer
7. Вставьте строку со значением _**c1=1**_
8. Создайте новую роль **_readonly_**
9. Дайте новой роли право на подключение к базе данных _**testdb**_
10. Дайте новой роли право на использование схемы _**testnm**_
11. Дайте новой роли право на select для всех таблиц схемы _**testnm**_
12. Создайте пользователя _**testread**_ с паролем _**test123**_
13. Дайте роль readonly пользователю _**testread**_
14. Зайдите под пользователем _**testread**_ в базу данных _**testdb**_
15. Сделайте: 
````postgresql
select * from t1;
````
16. Получилось? (могло если вы делали сами не по шпаргалке и не упустили один существенный момент про который позже)
17. Напишите что именно произошло в тексте домашнего задания
18. У вас есть идеи почему? ведь права то дали?
19. Посмотрите на список таблиц
20. Подсказка в шпаргалке под пунктом _**20**_
21. А, почему так получилось с таблицей (если делали сами и без шпаргалки, то может у вас все нормально)
22. Вернитесь в базу данных _**testdb**_ под пользователем postgres
23. Удалите таблицу _**t1**_
24. Создайте ее заново но уже с явным указанием имени схемы _**testnm**_
25. Вставьте строку со значением _**c1=1**_
26. Зайдите под пользователем _**testread**_ в базу данных _**testdb**_
27. Сделайте: 
````postgresql
select * from t1;
````
28. Получилось?
29. Есть идеи почему? если нет - _смотрите шпаргалку_
30. Как сделать так чтобы такое больше не повторялось? если нет идей - _смотрите шпаргалку_
31. Сделайте:
```postgresql
select * from testnm.t1;
```
32. Получилось?
33. Есть идеи почему? если нет - _смотрите шпаргалку_
34. Сделайте:
```postgresql
select * from testnm.t1;
```
35. Получилось?
36. **УРА!**
37. Теперь попробуйте выполнить команду:
```postgresql
create table t2(c1 integer); insert into t2 values (2);
```
38. А, как так? Нам же никто прав на создание **таблиц** и **insert** в них под ролью _**readonly**_?
39. Есть идеи как убрать эти права? если нет - _смотрите шпаргалку_
40. Если вы справились сами то расскажите что сделали и почему, если смотрели шпаргалку - объясните что сделали и почему выполнив указанные в ней команды
41. Теперь попробуйте выполнить команду:
```postgresql
create table t3(c1 integer); insert into t2 values (2);
```
42. Расскажите что получилось и почему
-----------
copyright: https://github.com/mrAlexbody/otus_postgre-dba-2024-07/blob/main/HW04/README.md
#### Создадим новый кластер **Postresql 14**:
Создую виртуальную машины с UBUNTU 22.04 LTS в YandexCloud и подключаемся к ней по SSH:
```shell
PS C:\Users\Alexander> yc compute instance create --name otus-hw07 --hostname otus-hw07 --cores 2 --memory 4 --create-boot-disk size=15G,type=network-hdd,image-folder-id=standard-images,image-family=ubuntu-2204-lts --network-interface subnet-name=otus-subnet,nat-ip-version=ip
v4 --ssh-key C:\Users\Alexander/.ssh/id_rsa.pub
PS C:\Users\Alexander> yc compute instances list
+----------------------+-----------+---------------+---------+---------------+----------------+
|          ID          |   NAME    |    ZONE ID    | STATUS  |  EXTERNAL IP  |  INTERNAL IP   |
+----------------------+-----------+---------------+---------+---------------+----------------+
| fhm2d7ckivh8gh3vbr0q | otus-hw07 | ru-central1-a | RUNNING | 62.84.118.106 | 192.168.100.10 |
+----------------------+-----------+---------------+---------+---------------+----------------+

PS C:\Users\Alexander> ssh yc-user@62.84.118.106 -i C:\Users\Alexander\.ssh\id_ed25519
Warning: Permanently added '62.84.118.106' (ED25519) to the list of known hosts.
Welcome to Ubuntu 22.04.4 LTS (GNU/Linux 5.15.0-119-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Sat Sep 14 07:54:51 PM UTC 2024

  System load:  0.0                Processes:             128
  Usage of /:   27.4% of 14.68GB   Users logged in:       0
  Memory usage: 5%                 IPv4 address for eth0: 192.168.100.10
  Swap usage:   0%


Expanded Security Maintenance for Applications is not enabled.

0 updates can be applied immediately.

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status



The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

yc-user@otus-hw07:~$
```
Установка Postgresql14:
```shell
yc-user@otus-hw07:~$ sudo apt update -y
yc-user@otus-hw07:~$ sudo apt install -y postgresql-14
```
Проверяем:
````shell
yc-user@otus-hw07:~$ sudo ss -tulp |grep post
tcp   LISTEN 0      244              127.0.0.1:postgresql      0.0.0.0:*    users:(("postgres",pid=4540,fd=6))
tcp   LISTEN 0      244                  [::1]:postgresql         [::]:*    users:(("postgres",pid=4540,fd=5))

yc-user@otus-hw07:~$ ps -axu |grep postgresql
postgres    4540  0.0  0.7 218348 30612 ?        Ss   19:59   0:00 /usr/lib/postgresql/14/bin/postgres -D /var/lib/postgresql/14/main -c config_file=/etc/postgresql/14/main/postgresql.conf
yc-user     4729  0.0  0.0   6612  2476 pts/1    S+   20:05   0:00 grep --color=auto postgresql
````
#### Зайдём в созданный кластер под пользователем _postgres_:
```shell
yc-user@otus-hw07:~$ sudo -u postgres psql
could not change directory to "/home/yc-user": Permission denied
psql (14.13 (Ubuntu 14.13-1.pgdg22.04+1))
Type "help" for help.

postgres=# \conninfo
You are connected to database "postgres" as user "postgres" via socket in "/var/run/postgresql" at port "5432".
postgres=#
```
#### Cоздадим новую базу данных _testdb_:
```postgresql
postgres=# create database testdb;
CREATE DATABASE
postgres=# \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges
-----------+----------+----------+-------------+-------------+-----------------------
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 testdb    | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
(4 rows)
```
#### Зайдите в созданную базу данных под пользователем _postgres_ создадим новую схему _testnm_:
```postgresql
postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
testdb=# create schema testnm;
CREATE SCHEMA
testdb=# \dn
  List of schemas
  Name  |  Owner
--------+----------
 public | postgres
 testnm | postgres
(2 rows)

testdb=#
```
#### Создадим новую таблицу _t1_ с одной колонкой _c1_ типа _integer_  и вставим строку со значением _c1=1_:
```postgresql
testdb=# create table t1 (c1 integer);
CREATE TABLE
testdb=# \dt
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | t1   | table | postgres
(1 row)

testdb=# \d t1
                 Table "public.t1"
 Column |  Type   | Collation | Nullable | Default
--------+---------+-----------+----------+---------
 c1     | integer |           |          |
testdb=# insert into t1 values (1);
INSERT 0 1
testdb=# select * from t1;
 c1
----
  1
(1 row)
```
#### Создадим новую роль _readonly_ и дадим новой роли право на подключение к базе данных _testdb_,а так же дадим новой роли право на использование схемы _testnm_:
```postgresql
testdb=# create role readonly;
CREATE ROLE
testdb=# \du
                                   List of roles
 Role name |                         Attributes                         | Member of
-----------+------------------------------------------------------------+-----------
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
 readonly  | Cannot login                                               | {}

testdb=# GRANT connect on DATABASE testdb TO readonly;
GRANT
testdb=# GRANT usage on SCHEMA testnm to readonly;
GRANT
```
#### Дадим новой роли право на _select_ для всех таблиц схемы _testnm_:
```postgresql
testdb=# GRANT SELECT on all TABLES in SCHEMA testnm TO readonly;
GRANT
```
#### Создадим пользователя _testread_ с паролем _test123_:
```postgresql
testdb=# CREATE USER testread with password 'test123';
CREATE ROLE
```
#### Дадим роль _readonly_ пользователю _testread_:
```postgresql
testdb=# GRANT readonly TO testread;
GRANT ROLE

testdb=# \du testread
            List of roles
 Role name | Attributes | Member of
-----------+------------+------------
 testread  |            | {readonly}

```

#### Зайдём под пользователем _testread_ в базу данных _testdb_ и выполним запрос  _*select * from t1;*_ :
```postgresql
yc-user@otus-hw07:~$ psql -h localhost -U testread -b testdb
Password for user testread:
psql (14.13 (Ubuntu 14.13-1.pgdg22.04+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.
testdb=> \dt
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | t1   | table | postgres
(1 row)
                       
testdb=> select * from t1;
ERROR:  permission denied for table t1
STATEMENT:  select * from t1;
                       
```
> Получилось? Есть идеи почему? 
>> Нет доступа в схему _*public*_, разрешения давали для схемы _*testnm*_.
>> Нужно явно указать схему _*tetsnm*_, т.к. по умолчанию  используется схема _*public*_
```postgresql
testdb=> \dt testnm.t1
Did not find any relation named "testnm.t1".

testdb=> \dt public.t1
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | t1   | table | postgres
(1 row)

testdb=> \dn+
List of schemas
  Name  |  Owner   |  Access privileges   |      Description
--------+----------+----------------------+------------------------
 public | postgres | postgres=UC/postgres+| standard public schema
        |          | =UC/postgres         |
 testnm | postgres | postgres=UC/postgres+|
        |          | readonly=U/postgres  |
(2 rows)
```
#### Вернитесь в базу данных _*testdb*_ под пользователем _*postgres*_ удалите таблицу t1
```postgresql
testdb=# DROP TABLE public.t1 ;
DROP TABLE
```
#### Создайте ее заново но уже с явным указанием имени схемы _**testnm**_
```postgresql

testdb=# CREATE TABLE testnm.t1 ( c1 integer );
CREATE TABLE
testdb=# \dt+ testnm.t1
                                   List of relations
 Schema | Name | Type  |  Owner   | Persistence | Access method |  Size   | Description
--------+------+-------+----------+-------------+---------------+---------+-------------
 testnm | t1   | table | postgres | permanent   | heap          | 0 bytes |
(1 row)
```
#### Вставьте строку со значением _**c1=1**_
```postgresql
testdb=# INSERT INTO testnm.t1 values(1);
INSERT 0 1
```
#### Зайдите под пользователем _**testread**_ в базу данных _**testdb**_
```postgresql
yc-user@otus-hw07:~$ psql -h localhost -U testread -b testdb
Password for user testread:
psql (14.13 (Ubuntu 14.13-1.pgdg22.04+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

testdb=>
```
#### Сделайте запрос: _**SELECT  * FROM testnm.t1;**_
```postgresql
testdb=> SELECT  * FROM testnm.t1;
ERROR:  permission denied for table t1
STATEMENT:  SELECT  * FROM testnm.t1;
```
> Получилось?
>> Нет.  Потому что _*grant SELECT on all TABLEs in SCHEMA testnm TO readonlyж;*_ дал доступ только для существующих на тот момент времени таблиц а t1 пересоздавалась.
#### Как сделать так чтобы такое больше не повторялось?
>> Нужна изменить привилегии для схемы _*testnm*_:
```postgresql
testdb=# ALTER default privileges in SCHEMA testnm grant SELECT on TABLES to readonly;
ALTER DEFAULT PRIVILEGES
testdb=# \c testdb testread;
Password for user testread:
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
You are now connected to database "testdb" as user "testread".
testdb=>
```
#### Cделайте _*SELECT * FROM testnm.t1;*_
```postgresql
testdb=# \c testdb testread;
Password for user testread:
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
You are now connected to database "testdb" as user "testread".
testdb=> select * from testnm.t1;
ERROR:  permission denied for table t1
```
> Получилось?
>> Нет. Потому что *ALTER* default будет действовать для новых таблиц, 
>>  а _*GRANT SELECT on all TABLES in SCHEMA testnm TO readonly;*_ 
>>  отработал только для существующих на тот момент времени. 
>>  Надо сделать снова *GRANT SELECT* или пересоздать таблицу.
```postgresql
postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
testdb=# GRANT SELECT on all TABLES in SCHEMA testnm TO readonly;
GRANT
```
>> Проверим:
```postgresql
yc-user@otus-hw07:~$ psql -h localhost -U testread -b testdb
Password for user testread:
psql (14.13 (Ubuntu 14.13-1.pgdg22.04+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

testdb=> select * from testnm.t1;
 c1
----
  1
(1 row)

testdb=> \dp testnm.t1
                                Access privileges
 Schema | Name | Type  |     Access privileges     | Column privileges | Policies
--------+------+-------+---------------------------+-------------------+----------
 testnm | t1   | table | postgres=arwdDxt/postgres+|                   |
        |      |       | readonly=r/postgres       |                   |
(1 row)
```
>> Теперь всё получилось, после выполнения *GRANT SELECT*.
#### Теперь попробуйте выполнить команду _*CREATE TABLE t2(c1 integer);*_ _*INSERT INTO t2 values (2);*_
````postgresql
postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
testdb=# CREATE TABLE t2(c1 integer);
CREATE TABLE
testdb=# INSERT INTO t2 values (2);
INSERT 0 1
testdb=> \dt
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | t2   | table | postgres
(1 row)

testdb=> select * from pg_catalog.pg_namespace
testdb-> ;
  oid  |      nspname       | nspowner |                   nspacl
-------+--------------------+----------+--------------------------------------------
    99 | pg_toast           |       10 |
    11 | pg_catalog         |       10 | {postgres=UC/postgres,=U/postgres}
  2200 | public             |       10 | {postgres=UC/postgres,=UC/postgres}
 13395 | information_schema |       10 | {postgres=UC/postgres,=U/postgres}
 16385 | testnm             |       10 | {postgres=UC/postgres,readonly=U/postgres}
(5 rows)
````
#### А как так? Нам же никто прав на создание таблиц и insert в них под ролью readonly?
>> Это все потому что _*search_path*_ указывает в первую очередь на схему _*public._* 
А схема _*public*_ создается в каждой базе данных по умолчанию. 
И _*grant*_ на все действия в этой схеме дается роли _*public.*_ 
А роль _*public*_ добавляется всем новым пользователям. 
Соответсвенно  каждый пользователь может по умолчанию создавать объекты в схеме public любой базы данных, 
естественно если у него есть право на подключение к этой базе данных. 
#### Есть идеи как убрать эти права? 
>> Нужно отозвать права на схему _*public*_.
```postgresql
testdb=> \c testdb postgres;
Password for user postgres:
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
You are now connected to database "testdb" as user "postgres".
testdb=# REVOKE CREATE on SCHEMA public FROM PUBLIC;
REVOKE
testdb=# REVOKE ALL on DATABASE testdb FROM public;
REVOKE
```
#### Теперь попробуйте выполнить команду _*create table t3(c1 integer); insert into t2 values (2);*_
```shell
yc-user@otus-hw07:~$ psql -h localhost -U testread -b testdb
Password for user testread:
psql (14.13 (Ubuntu 14.13-1.pgdg22.04+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

testdb=>
```
```postgresql
testdb=> CREATE TABLE t3(c1 integer); INSERT INTO t2 VALUES (2);
ERROR:  permission denied for schema public
LINE 1: CREATE TABLE t3(c1 integer);
                     ^
STATEMENT:  CREATE TABLE t3(c1 integer);
ERROR:  permission denied for table t2
STATEMENT:  INSERT INTO t2 VALUES (2);
```
>> Создать не удалось, т.к. нет прав в сехеме public. Права было отозваны для всех пользователей. 