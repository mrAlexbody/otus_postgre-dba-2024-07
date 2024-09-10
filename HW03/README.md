Домашнее задание 03
====================
## Установка и настройка PostgreSQL

### Цель:
* создавать дополнительный диск для уже существующей виртуальной машины, размечать его и делать на нем файловую систему
* переносить содержимое базы данных PostgreSQL на дополнительный диск
* переносить содержимое БД PostgreSQL между виртуальными машинами

### Описание/Пошаговая инструкция выполнения домашнего задания:
* создайте виртуальную машину c Ubuntu 20.04/22.04 LTS в ЯО/Virtual Box/докере
* поставьте на нее PostgreSQL 15 через sudo apt
* проверьте что кластер запущен через sudo -u postgres pg_lsclusters
* зайдите из-под пользователя postgres в psql и сделайте произвольную таблицу с произвольным содержимым

````postgresql
postgres=# create table test(c1 text);
postgres=# insert into test values('1');
\q
````
* остановите postgres, например, через **sudo -u postgres pg_ctlcluster 15 main stop** 
* создайте новый диск к ВМ размером 10GB
* добавьте свежесозданный диск к виртуальной машине - надо зайти в режим ее редактирования и дальше выбрать пункт attach existing disk
* проинициализируйте диск согласно инструкции и подмонтировать файловую систему, только не забывайте менять имя диска на актуальное, в вашем случае это скорее всего будет **/dev/sdb** 
<https://www.digitalocean.com/community/tutorials/how-to-partition-and-format-storage-devices-in-linux>
* перезагрузите инстанс и убедитесь, что диск остается примонтированным (если не так смотрим в сторону fstab)
* сделайте пользователя postgres владельцем **/mnt/data** - chown -R postgres:postgres /mnt/data/
* перенесите содержимое **/var/lib/postgres/15** в **/mnt/data** - **mv /var/lib/postgresql/15 /mnt/data**
* попытайтесь запустить кластер - **sudo -u postgres pg_ctlcluster 15 main start**
* напишите, получилось или нет и почему
* задание: найти конфигурационный параметр в файлах, расположенных в **/etc/postgresql/15/main**, который надо поменять и поменяйте его
* напишите, что и почему поменяли
* попытайтесь запустить кластер - **sudo -u postgres pg_ctlcluster 15 main start**
* напишите, получилось или нет и почему
* зайдите через psql и проверьте содержимое ранее созданной таблицы
* задание со звездочкой *: не удаляя существующий инстанс ВМ сделайте новый, поставьте на его PostgreSQL, удалите файлы с данными из **/var/lib/postgres**, перемонтируйте внешний диск который сделали ранее от первой виртуальной машины ко второй и запустите PostgreSQL на второй машине так чтобы он работал с данными на внешнем диске, расскажите как вы это сделали и что в итоге получилось.
----------------------------
copyright: https://github.com/mrAlexbody/otus_postgre-dba-2024-07/blob/main/HW03/README.md
### Создание виртуальной машины с UBUNTU 22.04 LTS в YandexCloud
```` shell
PS C:\Users\Alexander> yc compute instance create --name otus-db --hostname otus-db --cores 2 --memory 4 --create-boot-disk size=15G,type=
network-hdd,image-folder-id=standard-images,image-family=ubuntu-2204-lts --network-interface subnet-name=otus-subnet,nat-ip-version=ipv4 -
-ssh-key C:\Users\Alexander/.ssh/yc_key.pub
````
```shell
done (35s)
id: fhmdbi854jp4opdmjaqn
folder_id: b1gdmm6es3knkn350jdo
created_at: "2024-09-08T18:33:39Z"
name: otus-db
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
  device_name: fhmmt557edn7tfocqd7a
  auto_delete: true
  disk_id: fhmmt557edn7tfocqd7a
network_interfaces:
  - index: "0"
    mac_address: d0:0d:d5:c9:05:24
    subnet_id: e9bbp2c51jd655n334rk
    primary_v4_address:
      address: 192.168.100.35
      one_to_one_nat:
        address: 51.250.70.201
        ip_version: IPV4
serial_port_settings:
  ssh_authorization: OS_LOGIN
gpu_settings: {}
fqdn: otus-db.ru-central1.internal
scheduling_policy: {}
network_settings:
  type: STANDARD
placement_policy: {}
```
### Установка БД Postgresql 15 на виртуальную машину
Поскольку Postgresql 15 нет в репозитарии пакетов по умолчанию, то нужно сделать следующее: 
```shell
 sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
```
Далее импортируем ключи подписи GPG для репозитория:
```shell
wget -qO- https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo tee /etc/apt/trusted.gpg.d/pgdg.asc &>/dev/null
```
Обновим системные пакеты:
```shell
sudo apt update -y
```
Установка Postgresql 15: 
```shell
sudo apt install -y postgresql-15
```
Проверяем:
```shell
yc-user@otus-db:~$ ps aux |grep postgres
postgres    5348  0.0  0.7 219256 30628 ?        Ss   19:05   0:00 /usr/lib/postgresql/15/bin/postgres -D /var/lib/postgresql/15/main -c config_file=/etc/postgresql/15/main/postgresql.conf
postgres    5349  0.0  0.2 219388  9008 ?        Ss   19:05   0:00 postgres: 15/main: checkpointer
postgres    5350  0.0  0.1 219404  6788 ?        Ss   19:05   0:00 postgres: 15/main: background writer
postgres    5352  0.0  0.2 219256 11340 ?        Ss   19:05   0:00 postgres: 15/main: walwriter
postgres    5353  0.0  0.2 220852  9436 ?        Ss   19:05   0:00 postgres: 15/main: autovacuum launcher
postgres    5354  0.0  0.1 220828  7496 ?        Ss   19:05   0:00 postgres: 15/main: logical replication launcher
yc-user     5561  0.0  0.0   6612  2408 pts/0    S+   19:15   0:00 grep --color=auto postgres
yc-user@otus-db:~$ sudo ss -tupln |grep 5432
tcp   LISTEN 0      244              127.0.0.1:5432      0.0.0.0:*    users:(("postgres",pid=5348,fd=6))
tcp   LISTEN 0      244                  [::1]:5432         [::]:*    users:(("postgres",pid=5348,fd=5))
```
```shell
yc-user@otus-db:~$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```
### Сделаем произвольную таблицу с произвольным содержимым
```shell
postgres=# create table test(c1 text);
CREATE TABLE
postgres=# insert into test values('1');
INSERT 0 1
postgres=# \dt
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | test | table | postgres
(1 row)
\q
```
### Остановим postgres и сразу проверим: 
```shell
yc-user@otus-db:~$ sudo -u postgres pg_ctlcluster 15 main stop
Warning: stopping the cluster using pg_ctlcluster will mark the systemd unit as failed. Consider using systemctl:
  sudo systemctl stop postgresql@15-main
yc-user@otus-db:~$ sudo ps aux|grep postgres
yc-user     5621  0.0  0.0   6612  2236 pts/0    S+   19:24   0:00 grep --color=auto postgres
````
````shell
yc-user@otus-db:~$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 down   postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
````
### Создадим новый диск на 10 ГБ и подключим его к ВМ:

````shell
PS C:\Users\Alexander> yc compute disk create --name new-disk --type network-hdd --size 10 --description "second disk for otus-db"
done (5s)
id: fhm8ttbolbqbbkp3g2lj
folder_id: b1gdmm6es3knkn350jdo
created_at: "2024-09-08T19:42:26Z"
name: new-disk
description: second disk for otus-db
type_id: network-hdd
zone_id: ru-central1-a
size: "10737418240"
block_size: "4096"
status: READY
disk_placement_policy: {}

````
Проверим:
````shell
PS C:\Users\Alexander> yc compute disk list
+----------------------+-------------+-------------+---------------+--------+----------------------+-----------------+-------------------------+
|          ID          |    NAME     |    SIZE     |     ZONE      | STATUS |     INSTANCE IDS     | PLACEMENT GROUP |       DESCRIPTION       |
+----------------------+-------------+-------------+---------------+--------+----------------------+-----------------+-------------------------+
| fhm8ttbolbqbbkp3g2lj | new-disk    | 10737418240 | ru-central1-a | READY  | fhmdbi854jp4opdmjaqn |                 | second disk for otus-db |
| fhmaj7ncv1obfnjntpvg |             | 16106127360 | ru-central1-a | READY  | fhmpjlt2fqaga6vqhsi0 |                 |                         |
| fhmmt557edn7tfocqd7a |             | 16106127360 | ru-central1-a | READY  | fhmdbi854jp4opdmjaqn |                 |                         |
+----------------------+-------------+-------------+---------------+--------+----------------------+-----------------+-------------------------+

PS C:\Users\Alexander> yc compute instance list
+----------------------+---------+---------------+---------+---------------+----------------+
|          ID          |  NAME   |    ZONE ID    | STATUS  |  EXTERNAL IP  |  INTERNAL IP   |
+----------------------+---------+---------------+---------+---------------+----------------+
| fhmdbi854jp4opdmjaqn | otus-db | ru-central1-a | RUNNING | 51.250.70.201 | 192.168.100.35 |
+----------------------+---------+---------------+---------+---------------+----------------+
````
Вот появился 2-ой диск:
```shell
yc-user@otus-db:~$ sudo lsblk -o NAME,FSTYPE,SIZE,MOUNTPOINT,LABEL
NAME   FSTYPE     SIZE MOUNTPOINT        LABEL
loop0  squashfs  63.3M /snap/core20/1822
loop1  squashfs  63.9M /snap/core20/2318
loop2  squashfs 111.9M /snap/lxd/24322
loop3  squashfs    87M /snap/lxd/29351
loop4  squashfs  49.8M /snap/snapd/18357
loop5  squashfs  38.8M /snap/snapd/21759
vda                15G
├─vda1              1M
└─vda2 ext4        15G /
vdb                10G
```
### Создадим разделы с помощью fdisk:

```shell
yc-user@otus-db:~$ sudo fdisk /dev/vdb
Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1):
First sector (2048-20971519, default 2048):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-20971519, default 20971519):

Created a new partition 1 of type 'Linux' and of size 10 GiB.

Command (m for help): i

Selected partition 1
         Device: /dev/vdb1
          Start: 2048
            End: 20971519
        Sectors: 20969472
      Cylinders: 323
           Size: 10G
             Id: 83
           Type: Linux
    Start-C/H/S: 0/8/9
      End-C/H/S: 322/131/1

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.

```
```shell
yc-user@otus-db:~$ sudo fdisk -l
.......
Disk /dev/vdb: 10 GiB, 10737418240 bytes, 20971520 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: dos
Disk identifier: 0x5dbfaf97

Device     Boot Start      End  Sectors Size Id Type
/dev/vdb1        2048 20971519 20969472  10G 83 Linux
```
Отформатируем диск в нужную файловую систему с помощью утилиты mkfs (файловую систему возьмем EXT4):
```shell
yc-user@otus-db:~$ sudo mkfs.ext4 /dev/vdb1
mke2fs 1.46.5 (30-Dec-2021)
Creating filesystem with 2621184 4k blocks and 655360 inodes
Filesystem UUID: 37dfcffe-80fe-42d5-a7fc-e57e7370acf8
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done
```
Смонтируем раздел диска vdb1 в папку /mnt/vdb1 с помощью утилиты mount и сделаем разрешение на запись всем пользователям:
```shell
yc-user@otus-db:~$ sudo mkdir /mnt/vdb1
yc-user@otus-db:~$ sudo mount /dev/vdb1 /mnt/vdb1
yc-user@otus-db:~$ sudo chmod a+w /mnt/vdb1
yc-user@otus-db:~$ df -h
Filesystem      Size  Used Avail Use% Mounted on
tmpfs           392M  1.1M  391M   1% /run
/dev/vda2        15G  4.3G  9.8G  31% /
tmpfs           2.0G     0  2.0G   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           392M  4.0K  392M   1% /run/user/1000
/dev/vdb1       9.8G   24K  9.3G   1% /mnt/vdb1
```
Перенесём содержимое папки /var/lib/postgres/15 на новый диск:
```shell
yc-user@otus-db:~$ sudo mv -vi /var/lib/postgresql/15/ /mnt/vdb1/
```
Попробуем запустить Postgresql 15:
```shell
yc-user@otus-db:~$ sudo -u postgres pg_ctlcluster 15 main start
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
  sudo systemctl start postgresql@15-main
Error: /usr/lib/postgresql/15/bin/pg_ctl /usr/lib/postgresql/15/bin/pg_ctl start -D /var/lib/postgresql/15/main -l /var/log/postgresql/postgresql-15-main.log -s -o  -c config_file="/etc/postgresql/15/main/postgresql.conf"  exited with status 1:
pg_ctl: directory "/var/lib/postgresql/15/main" is not a database cluster directory
```
> ### Напишите, получилось или нет и почему, запустить кластер БД ? 
>> Запустить не удалось, так как нет данных кластера для БД Postgresql 15, а они были перенесены в другое место. 

Найдём конфиг, где поменять:
```shell
yc-user@otus-db:~$ sudo -u postgres pg_conftool 15 main show all
cluster_name = '15/main'
data_directory = '/var/lib/postgresql/15/main'
datestyle = 'iso, mdy'
default_text_search_config = 'pg_catalog.english'
dynamic_shared_memory_type = posix
external_pid_file = '/var/run/postgresql/15-main.pid'
hba_file = '/etc/postgresql/15/main/pg_hba.conf'
ident_file = '/etc/postgresql/15/main/pg_ident.conf'
lc_messages = 'en_US.UTF-8'
lc_monetary = 'en_US.UTF-8'
lc_numeric = 'en_US.UTF-8'
lc_time = 'en_US.UTF-8'
log_line_prefix = '%m [%p] %q%u@%d '
log_timezone = 'Etc/UTC'
max_connections = 100
max_wal_size = 1GB
min_wal_size = 80MB
port = 5432
shared_buffers = 128MB
ssl = on
ssl_cert_file = '/etc/ssl/certs/ssl-cert-snakeoil.pem'
ssl_key_file = '/etc/ssl/private/ssl-cert-snakeoil.key'
timezone = 'Etc/UTC'
unix_socket_directories = '/var/run/postgresql'
```
> ### Что нужно поменять и где, чтобы кластер БД заработал ? 
> > Нужно поменять параметр "data_directory = '/var/lib/postgresql/15/main'", т.к. данные были перенесены и запись нужно поменять на "data_directory = '/mnt/vdb1/15/main'

Поменяем данный параметр:

```shell
yc-user@otus-db:~$ sudo -u postgres pg_conftool 15 main set data_directory /mnt/vdb1/15/main
yc-user@otus-db:~$ sudo -u postgres pg_conftool 15 main show all
cluster_name = '15/main'
data_directory = '/mnt/vdb1/15/main'
datestyle = 'iso, mdy'
default_text_search_config = 'pg_catalog.english'
dynamic_shared_memory_type = posix
external_pid_file = '/var/run/postgresql/15-main.pid'
hba_file = '/etc/postgresql/15/main/pg_hba.conf'
ident_file = '/etc/postgresql/15/main/pg_ident.conf'
lc_messages = 'en_US.UTF-8'
lc_monetary = 'en_US.UTF-8'
lc_numeric = 'en_US.UTF-8'
lc_time = 'en_US.UTF-8'
log_line_prefix = '%m [%p] %q%u@%d '
log_timezone = 'Etc/UTC'
max_connections = 100
max_wal_size = 1GB
min_wal_size = 80MB
port = 5432
shared_buffers = 128MB
ssl = on
ssl_cert_file = '/etc/ssl/certs/ssl-cert-snakeoil.pem'
ssl_key_file = '/etc/ssl/private/ssl-cert-snakeoil.key'
timezone = 'Etc/UTC'
unix_socket_directories = '/var/run/postgresql'
```
Пробуем запустить Postgresql 15 ещё раз:
````shell
yc-user@otus-db:/mnt/vdb1$ sudo -u postgres pg_ctlcluster 15 main start
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
  sudo systemctl restart postgresql@15-main
yc-user@otus-db:/mnt/vdb1$ ps aux | grep postgres
postgres    6256  0.1  0.7 219256 30604 ?        Ss   20:43   0:00 /usr/lib/postgresql/15/bin/postgres -D /mnt/vdb1/15/main -c config_file=/etc/postgresql/15/main/postgresql.conf
postgres    6257  0.0  0.1 219388  6872 ?        Ss   20:43   0:00 postgres: 15/main: checkpointer
postgres    6258  0.0  0.1 219256  6872 ?        Ss   20:43   0:00 postgres: 15/main: background writer
postgres    6260  0.0  0.1 219256  6872 ?        Ss   20:43   0:00 postgres: 15/main: walwriter
postgres    6261  0.0  0.2 220852  9584 ?        Ss   20:43   0:00 postgres: 15/main: autovacuum launcher
postgres    6262  0.0  0.1 220828  7776 ?        Ss   20:43   0:00 postgres: 15/main: logical replication launcher
yc-user     6274  0.0  0.0   6612  2320 pts/0    S+   20:44   0:00 grep --color=auto postgres
yc-user@otus-db:/mnt/vdb1$ sudo ss -tlpn |grep 5432
LISTEN 0      244        127.0.0.1:5432      0.0.0.0:*    users:(("postgres",pid=6256,fd=6))
LISTEN 0      244            [::1]:5432         [::]:*    users:(("postgres",pid=6256,fd=5))
````
> ### Напишите, получилось или нет и почему!
>> Запустить получилось кластер Postgresql 15, так как был изменён параметр "data_directory" на правильное место расположение данных. 

Проверим содержимое ранее созданной таблицы:
```shell
yc-user@otus-db:/mnt/vdb1$ sudo -u postgresql psql
sudo: unknown user postgresql
sudo: error initializing audit plugin sudoers_audit
yc-user@otus-db:/mnt/vdb1$ sudo -u postgres psql
psql (15.8 (Ubuntu 15.8-1.pgdg22.04+1))
Type "help" for help.

postgres=# \dt
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | test | table | postgres
(1 row)
postgres=# select * from test;
 c1
----
 1
(1 row)

```
>## Задание со звездочкой *: 
> _Не удаляя существующий инстанс ВМ, сделайте новый, поставьте на его PostgreSQL, удалите файлы с данными из /var/lib/postgres, перемонтируйте внешний диск который сделали ранее от первой виртуальной машины ко второй и запустите PostgreSQL на второй машине так чтобы он работал с данными на внешнем диске, расскажите как вы это сделали и что в итоге получилось.
>_ 
> > 1. Отлючим ВМ и отсоеденим диск с данными 
> > 2. Далеее описание действий:
> 
> 
-----
>>  2.1. Установим 2-ую ВМ
```shell
PS C:\Users\Alexander> yc compute instance create --name otus-vm --hostname otus-vm --cores 2 --memory 4 --create-boot-disk size=15G,type=network-hdd,image-folder-id=standard-images,image-family=ubuntu-2204-lts --network-interface subnet-name=otus-subnet,nat-ip-version=ipv4 --ssh-key C:\Users\Alexander/.ssh/id_rsa.pub
done (52s)
id: fhmpjlt2fqaga6vqhsi0
folder_id: b1gdmm6es3knkn350jdo
created_at: "2024-10-08T12:21:39Z"
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
        address: 89.169.150.111
        ip_version: IPV4
serial_port_settings:
  ssh_authorization: OS_LOGIN
gpu_settings: {}
fqdn: otus-vm.ru-central1.internal
scheduling_policy: {}
network_settings:
  type: STANDARD
placement_policy: {}

PPS C:\Users\Alexander> yc compute instance list
+----------------------+---------+---------------+---------+----------------+----------------+
|          ID          |  NAME   |    ZONE ID    | STATUS  |  EXTERNAL IP   |  INTERNAL IP   |
+----------------------+---------+---------------+---------+----------------+----------------+
| fhmdbi854jp4opdmjaqn | otus-db | ru-central1-a | STOPPED |                | 192.168.100.35 |
| fhmpjlt2fqaga6vqhsi0 | otus-vm | ru-central1-a | RUNNING | 89.169.150.111 | 192.168.100.12 |
+----------------------+---------+---------------+---------+----------------+----------------+

PS C:\Users\Alexander> yc compute disk list
+----------------------+-------------+-------------+---------------+--------+----------------------+-----------------+-------------------------+
|          ID          |    NAME     |    SIZE     |     ZONE      | STATUS |     INSTANCE IDS     | PLACEMENT GROUP |       DESCRIPTION       |
+----------------------+-------------+-------------+---------------+--------+----------------------+-----------------+-------------------------+
| fhm8ttbolbqbbkp3g2lj | new-disk    | 10737418240 | ru-central1-a | READY  |                      |                 | second disk for otus-db |
| fhmaj7ncv1obfnjntpvg |             | 16106127360 | ru-central1-a | READY  | fhmpjlt2fqaga6vqhsi0 |                 |                         |
| fhmmt557edn7tfocqd7a |             | 16106127360 | ru-central1-a | READY  | fhmdbi854jp4opdmjaqn |                 |                         |
| fhmposveic73o1rimkqg | otus-hdd-db | 10737418240 | ru-central1-a | READY  |                      |                 |                         |
+----------------------+-------------+-------------+---------------+--------+----------------------+-----------------+-------------------------+
```
>> 2.2. Подключим диск:
```shell
PS C:\Users\Alexander> yc compute instance attach-disk --name otus-vm --disk-name new-disk
done (15s)
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
secondary_disks:
  - mode: READ_WRITE
    device_name: fhm8ttbolbqbbkp3g2lj
    disk_id: fhm8ttbolbqbbkp3g2lj
network_interfaces:
  - index: "0"
    mac_address: d0:0d:19:9d:7a:27
    subnet_id: e9bbp2c51jd655n334rk
    primary_v4_address:
      address: 192.168.100.12
      one_to_one_nat:
        address: 89.169.150.111
        ip_version: IPV4
serial_port_settings:
  ssh_authorization: OS_LOGIN
gpu_settings: {}
fqdn: otus-vm.ru-central1.internal
scheduling_policy: {}
network_settings:
  type: STANDARD
placement_policy: {}
```
>> 2.3. Подключаемся к ВМ и монтируем диск в системе в **/mnt/data**:
```shell
yc-user@otus-vm:~$ sudo fdisk -l
.....
Device     Boot Start      End  Sectors Size Id Type
/dev/vdb1        2048 20971519 20969472  10G 83 Linux
yc-user@otus-vm:~$ sudo lsblk -l
NAME  MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0   7:0    0  63.3M  1 loop /snap/core20/1822
loop1   7:1    0  63.9M  1 loop /snap/core20/2318
loop2   7:2    0 111.9M  1 loop /snap/lxd/24322
loop3   7:3    0    87M  1 loop /snap/lxd/29351
loop4   7:4    0  49.8M  1 loop /snap/snapd/18357
loop5   7:5    0  38.8M  1 loop /snap/snapd/21759
vda   252:0    0    15G  0 disk
vda1  252:1    0     1M  0 part
vda2  252:2    0    15G  0 part /
vdb   252:16   0    10G  0 disk
vdb1  252:17   0    10G  0 part

yc-user@otus-vm:~$ sudo mkdir /mnt/data
yc-user@otus-vm:~$ sudo chmod a+w /mnt/data
yc-user@otus-vm:~$ sudo mount /dev/vdb1 /mnt/data
yc-user@otus-vm:~$ ls -la /mnt/data
total 28
drwxrwxrwx 4 root root  4096 Sep  8 20:06 .
drwxr-xr-x 3 root root  4096 Sep 10 09:41 ..
drwxr-xr-x 3  114  120  4096 Sep  8 19:05 15
drwx------ 2 root root 16384 Sep  8 20:01 lost+found

yc-user@otus-vm:~$ ls -la /dev/disk/by-uuid/
total 0
drwxr-xr-x 2 root root  80 Sep 10 09:36 .
drwxr-xr-x 6 root root 120 Sep 10 09:22 ..
lrwxrwxrwx 1 root root  10 Sep 10 09:36 37dfcffe-80fe-42d5-a7fc-e57e7370acf8 -> ../../vdb1
lrwxrwxrwx 1 root root  10 Sep 10 09:23 ed465c6e-049a-41c6-8e0b-c8da348a3577 -> ../../vda2
yc-user@otus-vm:~$ sudo nano /etc/fstab
yc-user@otus-vm:~$ cat /etc/fstab
# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
# / was on /dev/vda2 during curtin installation
/dev/disk/by-uuid/ed465c6e-049a-41c6-8e0b-c8da348a3577 / ext4 defaults 0 1
/dev/disk/by-uuid/37dfcffe-80fe-42d5-a7fc-e57e7370acf8 /mnt/data ext4 defaults 0 1

```
>> 2.4. Установим Postgresql:
```shell
yc-user@otus-vm:~$ sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
yc-user@otus-vm:~$ wget -qO- https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo tee /etc/apt/trusted.gpg.d/pgdg.asc &>/dev/null
yc-user@otus-vm:~$ sudo apt update -y
yc-user@otus-vm:~$ sudo apt install -y postgresql-15
```
>> 2.5. Удалим файлы с данными из **/var/lib/postgres**:
```shell
yc-user@otus-vm:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 down   postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
yc-user@otus-vm:~$ sudo rm -rf /var/lib/postgresql/*
yc-user@otus-vm:~$ ls -la  /var/lib/postgresql/
total 8
drwxr-xr-x  2 postgres postgres 4096 Sep 10 09:59 .
drwxr-xr-x 43 root     root     4096 Aug 30 08:09 ..
```
>> 2.6. Изменим конфиг и запустим Postgresql так, что-бы заработал: 
```shell
yc-user@otus-vm:~$ sudo -u postgres pg_conftool 15 main show all
cluster_name = '15/main'
data_directory = '/var/lib/postgresql/15/main'
datestyle = 'iso, mdy'
default_text_search_config = 'pg_catalog.english'
dynamic_shared_memory_type = posix
external_pid_file = '/var/run/postgresql/15-main.pid'
hba_file = '/etc/postgresql/15/main/pg_hba.conf'
ident_file = '/etc/postgresql/15/main/pg_ident.conf'
lc_messages = 'en_US.UTF-8'
lc_monetary = 'en_US.UTF-8'
lc_numeric = 'en_US.UTF-8'
lc_time = 'en_US.UTF-8'
log_line_prefix = '%m [%p] %q%u@%d '
log_timezone = 'Etc/UTC'
max_connections = 100
max_wal_size = 1GB
min_wal_size = 80MB
port = 5432
shared_buffers = 128MB
ssl = on
ssl_cert_file = '/etc/ssl/certs/ssl-cert-snakeoil.pem'
ssl_key_file = '/etc/ssl/private/ssl-cert-snakeoil.key'
timezone = 'Etc/UTC'
unix_socket_directories = '/var/run/postgresql'

yc-user@otus-vm:~$ sudo -u postgres pg_conftool 15 main set data_directory /mnt/data/15/main

yc-user@otus-vm:~$ sudo -u postgres pg_conftool 15 main show all
cluster_name = '15/main'
data_directory = '/mnt/data/15/main'
datestyle = 'iso, mdy'
default_text_search_config = 'pg_catalog.english'
dynamic_shared_memory_type = posix
external_pid_file = '/var/run/postgresql/15-main.pid'
hba_file = '/etc/postgresql/15/main/pg_hba.conf'
ident_file = '/etc/postgresql/15/main/pg_ident.conf'
lc_messages = 'en_US.UTF-8'
lc_monetary = 'en_US.UTF-8'
lc_numeric = 'en_US.UTF-8'
lc_time = 'en_US.UTF-8'
log_line_prefix = '%m [%p] %q%u@%d '
log_timezone = 'Etc/UTC'
max_connections = 100
max_wal_size = 1GB
min_wal_size = 80MB
port = 5432
shared_buffers = 128MB
ssl = on
ssl_cert_file = '/etc/ssl/certs/ssl-cert-snakeoil.pem'
ssl_key_file = '/etc/ssl/private/ssl-cert-snakeoil.key'
timezone = 'Etc/UTC'
unix_socket_directories = '/var/run/postgresql'
```
>> 2.7. Стартуем и проверяем: 
```shell
yc-user@otus-vm:~$ sudo -u postgres pg_ctlcluster 15 main start
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
  sudo systemctl start postgresql@15-main
yc-user@otus-vm:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory    Log file
15  main    5432 online postgres /mnt/data/15/main /var/log/postgresql/postgresql-15-main.log
yc-user@otus-vm:~$ sudo -u postgres psql
````
```postgresql
psql (16.4 (Ubuntu 16.4-1.pgdg22.04+1), server 15.8 (Ubuntu 15.8-1.pgdg22.04+1))
Type "help" for help.
postgres=# \l
                                                       List of databases
   Name    |  Owner   | Encoding | Locale Provider |   Collate   |    Ctype    | ICU Locale | ICU Rules |   Access privileges
-----------+----------+----------+-----------------+-------------+-------------+------------+-----------+-----------------------
 postgres  | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |            |           |
 template0 | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |            |           | =c/postgres          +
           |          |          |                 |             |             |            |           | postgres=CTc/postgres
 template1 | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |            |           | =c/postgres          +
           |          |          |                 |             |             |            |           | postgres=CTc/postgres
(3 rows)

postgres=# select * from test;
 c1
----
 1
(1 row)

postgres=# \dt
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | test | table | postgres
(1 row)
```
>> **Всё ОК !!!**  
