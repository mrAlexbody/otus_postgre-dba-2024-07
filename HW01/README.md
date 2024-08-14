Домашнее задание 01
=========================
"Работа с уровнями изоляции транзакции в PostgreSQL"
--------------------------

## Цель:
+ научиться работать в Яндекс Облаке
+ научиться управлять уровнем изоляции транзакции в PostgreSQL и понимать особенность работы уровней read commited и repeatable read
> P.S: Для подключения используется PowerShell в Windows11
### Создание сети в Yandex Cloud
````
PS > yc vpc network create --name otus-net --description "otus-net"
````

### Создание подсети в Yandex Cloud

````
PS > yc vpc subnet create --name otus-subnet --range 192.168.0.0/24 --network-name otus-net --description "otus-subnet"
````

#### Проверка 

````
PS > yc vpc network list

|       ID              |   NAME   | 
| ----------------------|----------| 
| enp6g9s9dtgu5p49drvv  | otus-net | 

PS > yc vpc subnet list

|----------------------|-----------------------|----------------------|----------------|---------------+------------------+
|          ID          |         NAME          |      NETWORK ID      | ROUTE TABLE ID |     ZONE      |      RANGE       |
+----------------------+-----------------------+----------------------+----------------+---------------+------------------+
| e9bkd7fbtg29offoajl0 | otus-subnet           | enp6g9s9dtgu5p49drvv |                | ru-central1-a | [192.168.0.0/24] |
+----------------------+-----------------------+----------------------+----------------+---------------+------------------+
````
#### Создание инстанс виртуальной машины в Yandex CLoud c добавление своего ключа
````
PS > ssh-keygen.exe -t rsa -b 2048
PS > yc compute instance create --name otus-vm --hostname otus-vm --cores 2 --memory 4 --create-boot-disk size=15G,type=network-hdd,image-folder-id=standard-images,image-family=ubuntu-2004-lts --network-interface subnet-name=otus-subnet,nat-ip-version=ipv4 --ssh-key C:\Users\Alexander\.ssh\yc_key.pub

done (40s)
````
#### Включаем SSH-агент в PowerShell и добавляем закрытый ключ в агент SSH:
````
PS > Get-Service ssh-agent | Set-Service -StartupType Manual
PS > Get-Service ssh-agent

Status   Name               DisplayName
------   ----               -----------
Stopped  ssh-agent          OpenSSH Authentication Agent

PS > Start-Service ssh-agent
PS > Get-Service ssh-agent

Status   Name               DisplayName
------   ----               -----------
Running  ssh-agent          OpenSSH Authentication Agent

PS > ssh-add C:\Users\Alexander\.ssh\yc_key
Identity added: C:\Users\Alexander\.ssh\yc_key (alexander@LAPTOP)
 ````
#### Подключаемся в ВМ машине по SSH:
````
PS > ssh yc-user@89.169.150.222

Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 5.4.0-190-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro
New release '22.04.3 LTS' available.
Run 'do-release-upgrade' to upgrade to it.

Last login: Wed Aug 14 17:48:06 2024 from 85.30.248.78
yc-user@otus-vm:~$

````
#### Установка PostgreSQL (ОС Ubunut Linux):
````
yc-user@otus-vm:~$ sudo apt update
yc-user@otus-vm:~$ sudo apt install postgresql
yc-user@otus-vm:~$ sudo dpkg -l |grep postgresql
ii  postgresql                            12+214ubuntu0.1                   all          object-relational SQL database (supported version)
ii  postgresql-12                         12.19-0ubuntu0.20.04.1            amd64        object-relational SQL database, version 12 server
ii  postgresql-client-12                  12.19-0ubuntu0.20.04.1            amd64        front-end programs for PostgreSQL 12
ii  postgresql-client-common              214ubuntu0.1                      all          manager for multiple PostgreSQL client versions
ii  postgresql-common                     214ubuntu0.1                      all          PostgreSQL database-cluster manager

````
#### Подключение второй сессии SSH и в каждой сессии запускаем psql из-под пользователя postges

````
yc-user@otus-vm:~$ sudo su - postgres
postgres@otus-vm:~$ psql
psql (12.19 (Ubuntu 12.19-0ubuntu0.20.04.1))
Type "help" for help.

postgres=#\l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges
-----------+----------+----------+-------------+-------------+-----------------------
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(3 rows)

````
В первой сессии создать новую таблицу и наполнить ее данными: 
````
create table persons(
    id serial, 
    first_name text, 
    second_name text
); 
insert into persons(first_name, second_name) values('ivan', 'ivanov'); 
insert into persons(first_name, second_name) values('petr', 'petrov'); 
commit;
````
Посмотрим текущий уровень изоляции: 
````
postgres=# show transaction isolation level
postgres-# ;
 transaction_isolation
-----------------------
 read committed
(1 row)
````

