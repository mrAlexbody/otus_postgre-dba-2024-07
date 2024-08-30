
Домашнее задание
============================
#### Установка и настройка PostgteSQL в контейнере Docker

#### Цель:
установить PostgreSQL в Docker контейнере
настроить контейнер для внешнего подключения

#### Описание/Пошаговая инструкция выполнения домашнего задания:
1. создать ВМ с Ubuntu 20.04/22.04 или развернуть докер любым удобным способом
2. поставить на нем Docker Engine
3. сделать каталог /var/lib/postgres
4. развернуть контейнер с PostgreSQL 15 смонтировав в него /var/lib/postgresql
5. развернуть контейнер с клиентом postgres
6. подключится из контейнера с клиентом к контейнеру с сервером и сделать таблицу с парой строк
7. подключится к контейнеру с сервером с ноутбука/компьютера извне инстансов ЯО/места установки докера
8. удалить контейнер с сервером
9. создать его заново
10. подключится снова из контейнера с клиентом к контейнеру с сервером
11. проверить, что данные остались на месте
12. оставляйте в ЛК ДЗ комментарии что и как вы делали и как боролись с проблемами

------------------------------------
### Создание ВМ в Yandex CLOUD 
#### Создаем сетевую инфраструктуру для VM:
````shell
PS C:\Users\Alexander> yc vpc network create --name otus-net --description "otus-net"
id: enpn7jqomlc6d91uboad
folder_id: b1gdmm6es3knkn350jdo
created_at: "2024-08-30T06:09:21Z"
name: otus-net
description: otus-net
default_security_group_id: enp97slpiqcoek1qjqsk

PS C:\Users\Alexander> yc vpc network list
+----------------------+----------+
|          ID          |   NAME   |
+----------------------+----------+
| enp6jcd823d6nq1l4gkd | default  |
| enpn7jqomlc6d91uboad | otus-net |
+----------------------+----------+

PS C:\Users\Alexander> yc vpc subnet create --name otus-subnet --range 192.168.100.0/24 --network-name otus-net --description "otus-subnet"
id: e9bbp2c51jd655n334rk
folder_id: b1gdmm6es3knkn350jdo
created_at: "2024-08-30T06:10:48Z"
name: otus-subnet
description: otus-subnet
network_id: enpn7jqomlc6d91uboad
zone_id: ru-central1-a
v4_cidr_blocks:
  - 192.168.100.0/24
  
 PS C:\Users\Alexander> yc vpc subnet list
+----------------------+-----------------------+----------------------+----------------+---------------+--------------------+
|          ID          |         NAME          |      NETWORK ID      | ROUTE TABLE ID |     ZONE      |       RANGE        |
+----------------------+-----------------------+----------------------+----------------+---------------+--------------------+
| e2ln63lso5ja4obthji6 | default-ru-central1-b | enp6jcd823d6nq1l4gkd |                | ru-central1-b | [10.129.0.0/24]    |
| e9b52rp0pb9v95o4hm56 | default-ru-central1-a | enp6jcd823d6nq1l4gkd |                | ru-central1-a | [10.128.0.0/24]    |
| e9bbp2c51jd655n334rk | otus-subnet           | enpn7jqomlc6d91uboad |                | ru-central1-a | [192.168.100.0/24] |
| fl8u851uoo4f0h1ed99i | default-ru-central1-d | enp6jcd823d6nq1l4gkd |                | ru-central1-d | [10.131.0.0/24]    |
+----------------------+-----------------------+----------------------+----------------+---------------+--------------------+

````
### Устанавливаем ВМ:
````shell
PS C:\Users\Alexander> yc compute instance create --name otus-vm --hostname otus-vm --cores 2 --memory 4 --create-boot-disk size=15G,type=network-hdd,image-folder-id=standard-images,image-family=ubuntu-2204-lts --network-interface subnet-name=otus-subnet,nat-ip-version=ipv4 --ssh-key C:\Users\Alexander/.ssh/id_rsa.pub
done (52s)
id: fhmpjlt2fqaga6vqhsi0
folder_id: b1gdmm6es3knkn350jdo
created_at: "2024-08-30T06:47:17Z"
name: otus-vm
zone_id: ru-central1-a
platform_id: standard-v2
resources:
  memory: "4294967296"
  cores: "2"
  core_fraction: "100"
status: RUNNING
metadata_options:
  gce_http_endpoint: ENABLED
  aws_v1_http_endpoint: ENABLED
  gce_http_token: ENABLED
  aws_v1_http_token: DISABLED
boot_disk:
  mode: READ_WRITE
  device_name: fhmaj7ncv1obfnjntpvg
  auto_delete: true
  disk_id: fhmaj7ncv1obfnjntpvg
network_interfaces:
  - index: "0"
    mac_address: d0:0d:19:9d:7a:27
    subnet_id: e9bbp2c51jd655n334rk
    primary_v4_address:
      address: 192.168.100.12
      one_to_one_nat:
        address: 89.169.153.126
        ip_version: IPV4
serial_port_settings:
  ssh_authorization: OS_LOGIN
gpu_settings: {}
fqdn: otus-vm.ru-central1.internal
scheduling_policy: {}
network_settings:
  type: STANDARD
placement_policy: {}
````
````shell
PS C:\Users\Alexander> yc compute instances list
+----------------------+---------+---------------+---------+----------------+----------------+
|          ID          |  NAME   |    ZONE ID    | STATUS  |  EXTERNAL IP   |  INTERNAL IP   |
+----------------------+---------+---------------+---------+----------------+----------------+
| fhmpjlt2fqaga6vqhsi0 | otus-vm | ru-central1-a | RUNNING | 89.169.153.126 | 192.168.100.12 |
+----------------------+---------+---------------+---------+----------------+----------------+
````
### Установка Docker Engine:
Подключение к ВМ:
````shell
PS C:\Users\Alexander> ssh -i ~/.ssh/id_rsa yc-user@89.169.153.126
````
Установка Docker:
````shell
curl -fsSL https://get.docker.com -o get-docker.sh && sudo sh get-docker.sh && rm get-docker.sh && sudo usermod -aG docker $USER && newgrp docker
````
Проверим:
````shell
yc-user@otus-vm:~$ sudo docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
c1ec31eb5944: Pull complete
Digest: sha256:53cc4d415d839c98be39331c948609b659ed725170ad2ca8eb36951288f81b75
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
 
````
````shell
yc-user@otus-vm:~$ docker ps -a
CONTAINER ID   IMAGE         COMMAND    CREATED              STATUS                          PORTS     NAMES
079a3f07f6ea   hello-world   "/hello"   About a minute ago   Exited (0) About a minute ago             nifty_raman
````

##### Развернём контейнер с PostgreSQL 15 смонтировав в него /var/lib/postgresql:
Создадим каталог "/var/lib/postgres":
````shell
yc-user@otus-vm:~$ sudo mkdir  /var/lib/postgres
````
Создадим подсеть и запустим контейнер с PostgreSQL:
````shell
yc-user@otus-vm:~$ docker network create postgresnet
yc-user@otus-vm:~$ sudo docker run --name server-db --network postgresnet -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgr
esql/data postgres:15
Unable to find image 'postgres:15' locally
15: Pulling from library/postgres
e4fff0779e6d: Pull complete
ac2c82f5abbf: Pull complete
6423808c0870: Pull complete
9fc1a9646316: Pull complete
f486755b3b27: Pull complete
e3f131c9ffee: Pull complete
bb37c85245b8: Pull complete
af85de36d2ba: Pull complete
3fb6cac099e8: Pull complete
31efa1bde9c4: Pull complete
15bf20fe9460: Pull complete
3216df30c32c: Pull complete
cecbd5a08338: Pull complete
a733d89613cc: Pull complete
Digest: sha256:0836104ba0de8d09e8d54e2d6a28389fbce9c0f4fe08f4aa065940452ec61c30
Status: Downloaded newer image for postgres:15
402d5c0f3d522b63129e43b663152a99c4c158615070b2fe2870e35ab68e16a8
yc-user@otus-vm:~$ docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED         STATUS         PORTS                                       NAMES
402d5c0f3d52   postgres:15   "docker-entrypoint.s…"   9 seconds ago   Up 4 seconds   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   server-db
````
Проверим:
````shell
yc-user@otus-vm:~$ sudo ls -la /var/lib/postgres
total 136
drwx------ 19 lxd  root    4096 Aug 30 08:09 .
drwxr-xr-x 43 root root    4096 Aug 30 08:09 ..
drwx------  5 lxd  docker  4096 Aug 30 08:09 base
drwx------  2 lxd  docker  4096 Aug 30 08:11 global
drwx------  2 lxd  docker  4096 Aug 30 08:09 pg_commit_ts
drwx------  2 lxd  docker  4096 Aug 30 08:09 pg_dynshmem
-rw-------  1 lxd  docker  4821 Aug 30 08:09 pg_hba.conf
-rw-------  1 lxd  docker  1636 Aug 30 08:09 pg_ident.conf
drwx------  4 lxd  docker  4096 Aug 30 08:09 pg_logical
drwx------  4 lxd  docker  4096 Aug 30 08:09 pg_multixact
drwx------  2 lxd  docker  4096 Aug 30 08:09 pg_notify
drwx------  2 lxd  docker  4096 Aug 30 08:09 pg_replslot
drwx------  2 lxd  docker  4096 Aug 30 08:09 pg_serial
drwx------  2 lxd  docker  4096 Aug 30 08:09 pg_snapshots
drwx------  2 lxd  docker  4096 Aug 30 08:09 pg_stat
drwx------  2 lxd  docker  4096 Aug 30 08:09 pg_stat_tmp
drwx------  2 lxd  docker  4096 Aug 30 08:09 pg_subtrans
drwx------  2 lxd  docker  4096 Aug 30 08:09 pg_tblspc
drwx------  2 lxd  docker  4096 Aug 30 08:09 pg_twophase
-rw-------  1 lxd  docker     3 Aug 30 08:09 PG_VERSION
drwx------  3 lxd  docker  4096 Aug 30 08:09 pg_wal
drwx------  2 lxd  docker  4096 Aug 30 08:09 pg_xact
-rw-------  1 lxd  docker    88 Aug 30 08:09 postgresql.auto.conf
-rw-------  1 lxd  docker 29517 Aug 30 08:09 postgresql.conf
-rw-------  1 lxd  docker    36 Aug 30 08:09 postmaster.opts
-rw-------  1 lxd  docker    94 Aug 30 08:09 postmaster.pid

yc-user@otus-vm:~$ docker ps
CONTAINER ID   IMAGE         COMMAND                  CREATED         STATUS         PORTS                                       NAMES
402d5c0f3d52   postgres:15   "docker-entrypoint.s…"   5 minutes ago   Up 5 minutes   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   server-db
yc-user@otus-vm:~$ docker exec -it server-db /bin/bash
root@402d5c0f3d52:/# postgres -V
postgres (PostgreSQL) 15.8 (Debian 15.8-1.pgdg120+1)
````
#### Запускаем отдельный контейнер с клиентом в общей сети с БД:
```shell
yc-user@otus-vm:~$ sudo docker run -it --rm --network postgresnet --name pg-client postgres:15 psql -h server-db -U postgres
Password for user postgres:
psql (15.8 (Debian 15.8-1.pgdg120+1))
Type "help" for help.

postgres=#
````
````shell
yc-user@otus-vm:~$ docker container ps
CONTAINER ID   IMAGE         COMMAND                  CREATED              STATUS              PORTS                                       NAMES
6c79f1bc11cc   postgres:15   "docker-entrypoint.s…"   About a minute ago   Up About a minute   5432/tcp                                    pg-client
402d5c0f3d52   postgres:15   "docker-entrypoint.s…"   13 minutes ago       Up 13 minutes       0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   server-db
````
#### Cделаем БД и таблицы с парой строк:
````shell
postgres=# \l
                                                List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    | ICU Locale | Locale Provider |   Access privileges
-----------+----------+----------+------------+------------+------------+-----------------+-----------------------
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            |
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/postgres          +
           |          |          |            |            |            |                 | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/postgres          +
           |          |          |            |            |            |                 | postgres=CTc/postgres
(3 rows)

postgres=# CREATE DATABASE ddl_otus;
CREATE DATABASE
postgres=# \c ddl_otus;
ddl_otus=# CREATE TABLE table_one ( id_one integer PRIMARY KEY, some_text text);
CREATE TABLE
ddl_otus=# CREATE TABLE table_two ( id_one integer PRIMARY KEY, some_text text);
CREATE TABLE
ddl_otus=# \dt
           List of relations
 Schema |   Name    | Type  |  Owner
--------+-----------+-------+----------
 public | table_one | table | postgres
 public | table_two | table | postgres
(2 rows)
ddl_otus=# INSERT INTO table_one (id_one, some_text) VALUES (1, 'one'), (2, 'two');
INSERT 0 2
ddl_otus=# INSERT INTO table_two (id_one, some_text) VALUES (1, '1-2'), (2, '2-1');
INSERT 0 2
ddl_otus=# \q
````
#### Подключение из контейнера с клиентом к контейнеру с сервером БД:
````shell
yc-user@otus-vm:~$ docker run --rm -it --network postgresnet postgres:15 psql -h server-db -U postgres
Password for user postgres:
psql (15.8 (Debian 15.8-1.pgdg120+1))
Type "help" for help.

postgres=# \l
                                                List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    | ICU Locale | Locale Provider |   Access privileges
-----------+----------+----------+------------+------------+------------+-----------------+-----------------------
 ddl_otus  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            |
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            |
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/postgres          +
           |          |          |            |            |            |                 | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/postgres          +
           |          |          |            |            |            |                 | postgres=CTc/postgres
(4 rows)

postgres=#
````
#### Подключение к БД извне на внешний ip:
````shell
user@LAPTOP:~$ psql -p 5432 -h 89.169.153.126 -U postgres
Password for user postgres:
psql (14.13 (Ubuntu 14.13-0ubuntu0.22.04.1), server 15.8 (Debian 15.8-1.pgdg120+1))
WARNING: psql major version 14, server major version 15.
         Some psql features might not work.
Type "help" for help.

postgres=# \l
                                 List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges
-----------+----------+----------+------------+------------+-----------------------
 ddl_otus  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
(4 rows)

postgres=#
````
#### Подключение с сервера к БД:
```shell
yc-user@otus-vm:~$ docker ps
CONTAINER ID   IMAGE         COMMAND                  CREATED             STATUS             PORTS                                       NAMES
402d5c0f3d52   postgres:15   "docker-entrypoint.s…"   About an hour ago   Up About an hour   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   server-db
yc-user@otus-vm:~$ psql -h 127.0.0.1 -U postgres ddl_otus
Password for user postgres:
psql (14.13 (Ubuntu 14.13-0ubuntu0.22.04.1), server 15.8 (Debian 15.8-1.pgdg120+1))
WARNING: psql major version 14, server major version 15.
         Some psql features might not work.
Type "help" for help.

ddl_otus=# \dt
           List of relations
 Schema |   Name    | Type  |  Owner
--------+-----------+-------+----------
 public | table_one | table | postgres
 public | table_two | table | postgres
(2 rows)

ddl_otus=#

```
#### Удаление контейнера с сервером:
```shell
yc-user@otus-vm:~$ docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED             STATUS             PORTS                                       NAMES
402d5c0f3d52   postgres:15   "docker-entrypoint.s…"   About an hour ago   Up About an hour   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   server-db
yc-user@otus-vm:~$ docker container stop 402d5c0f3d52
402d5c0f3d52
yc-user@otus-vm:~$ docker container rm 402d5c0f3d52
402d5c0f3d52
````
Проверим что контейнера нет:
````shell
yc-user@otus-vm:~$ docker container ps -a
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
````
#### Создание контейнера заново:
```shell
yc-user@otus-vm:~$ docker run --name server-db --network postgresnet -e POSTGRES_PASSWORD=Pa$$W0rd -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/
data postgres:15
587c2cd07bca286df51e5a4455e477c1fd08185378d1d390ce562306301d6870
yc-user@otus-vm:~$ docker ps
CONTAINER ID   IMAGE         COMMAND                  CREATED         STATUS         PORTS                                       NAMES
587c2cd07bca   postgres:15   "docker-entrypoint.s…"   4 seconds ago   Up 4 seconds   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   server-db
```
#### Подключение из контейнера с клиентом к контейнеру с сервером:
```shell
yc-user@otus-vm:~$ docker run --rm -it --network postgresnet postgres:15 psql -h server-db -U postgres
Password for user postgres:
psql (15.8 (Debian 15.8-1.pgdg120+1))
Type "help" for help.

postgres=# 
```
#### Проверка данных:
```shell
postgres=# \l
                                                List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    | ICU Locale | Locale Provider |   Access privileges
-----------+----------+----------+------------+------------+------------+-----------------+-----------------------
 ddl_otus  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            |
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            |
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/postgres          +
           |          |          |            |            |            |                 | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/postgres          +
           |          |          |            |            |            |                 | postgres=CTc/postgres
(4 rows)

postgres=# \ddl

postgres=# \c ddl_otus
You are now connected to database "ddl_otus" as user "postgres".
ddl_otus=# \dt
           List of relations
 Schema |   Name    | Type  |  Owner
--------+-----------+-------+----------
 public | table_one | table | postgres
 public | table_two | table | postgres
(2 rows)

ddl_otus=#
```
Файлы на ВМ в /var/lib/postgres :
````shell
yc-user@otus-vm:~$ sudo ls -la /var/lib/postgres
total 136
drwx------ 19 lxd  root    4096 Aug 30 09:25 .
drwxr-xr-x 43 root root    4096 Aug 30 08:09 ..
drwx------  6 lxd  docker  4096 Aug 30 08:34 base
drwx------  2 lxd  docker  4096 Aug 30 09:26 global
drwx------  2 lxd  docker  4096 Aug 30 08:09 pg_commit_ts
drwx------  2 lxd  docker  4096 Aug 30 08:09 pg_dynshmem
-rw-------  1 lxd  docker  4821 Aug 30 08:09 pg_hba.conf
-rw-------  1 lxd  docker  1636 Aug 30 08:09 pg_ident.conf
drwx------  4 lxd  docker  4096 Aug 30 09:22 pg_logical
drwx------  4 lxd  docker  4096 Aug 30 08:09 pg_multixact
drwx------  2 lxd  docker  4096 Aug 30 08:09 pg_notify
drwx------  2 lxd  docker  4096 Aug 30 08:09 pg_replslot
drwx------  2 lxd  docker  4096 Aug 30 08:09 pg_serial
drwx------  2 lxd  docker  4096 Aug 30 08:09 pg_snapshots
drwx------  2 lxd  docker  4096 Aug 30 09:25 pg_stat
drwx------  2 lxd  docker  4096 Aug 30 08:09 pg_stat_tmp
drwx------  2 lxd  docker  4096 Aug 30 08:09 pg_subtrans
drwx------  2 lxd  docker  4096 Aug 30 08:09 pg_tblspc
drwx------  2 lxd  docker  4096 Aug 30 08:09 pg_twophase
-rw-------  1 lxd  docker     3 Aug 30 08:09 PG_VERSION
drwx------  3 lxd  docker  4096 Aug 30 08:09 pg_wal
drwx------  2 lxd  docker  4096 Aug 30 08:09 pg_xact
-rw-------  1 lxd  docker    88 Aug 30 08:09 postgresql.auto.conf
-rw-------  1 lxd  docker 29517 Aug 30 08:09 postgresql.conf
-rw-------  1 lxd  docker    36 Aug 30 09:25 postmaster.opts
-rw-------  1 lxd  docker    94 Aug 30 09:25 postmaster.pid
````
