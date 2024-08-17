Домашнее задание 01
=========================
"Работа с уровнями изоляции транзакции в PostgreSQL"
--------------------------

## Цель:
+ научиться работать в Яндекс Облаке
+ научиться управлять уровнем изоляции транзакции в PostgreSQL и понимать особенность работы уровней read commited и repeatable read
> P.S: Для подключения используется PowerShell в Windows11
### Создание сети в Yandex Cloud
````shell
PS > yc vpc network create --name otus-net --description "otus-net"
````

### Создание подсети в Yandex Cloud

````shell
PS > yc vpc subnet create --name otus-subnet --range 192.168.0.0/24 --network-name otus-net --description "otus-subnet"
````

#### Проверка 

````shell
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
````shell
PS > ssh-keygen.exe -t rsa -b 2048
PS > yc compute instance create --name otus-vm --hostname otus-vm --cores 2 --memory 4 --create-boot-disk size=15G,type=network-hdd,image-folder-id=standard-images,image-family=ubuntu-2004-lts --network-interface subnet-name=otus-subnet,nat-ip-version=ipv4 --ssh-key C:\Users\Alexander\.ssh\yc_key.pub

done (40s)
````
#### Включаем SSH-агент в PowerShell и добавляем закрытый ключ в агент SSH:
````shell
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
````shell
PS > ssh yc-user@89.169.150.222

Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 5.4.0-190-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro
New release '22.04.3 LTS' available.
Run 'do-release-upgrade' to upgrade to it.
````
#### Установка PostgreSQL (ОС Ubunut Linux):
````shell
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

```shell
yc-user@otus-vm:~$ sudo su - postgres
postgres@otus-vm:~$ psql

````
````postgresql
psql (12.19 (Ubuntu 12.19-0ubuntu0.20.04.1))
Type "help" for help.
ostgres=#\l
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
#### В первой сессии создать новую таблицу и наполнить ее данными: 
```postgresql
create table persons(
    id serial, 
    first_name text, 
    second_name text
); 
insert into persons(first_name, second_name) values('ivan', 'ivanov'); 
insert into persons(first_name, second_name) values('petr', 'petrov'); 
commit;
```
#### Текущий уровень изоляции в БД: 
````postgresql
postgres=# show transaction isolation level
postgres-# ;
 transaction_isolation
-----------------------
 read committed
(1 row)
````
#### Ноовая транзакция в обоих сессиях с дефолтным уровнем изоляции и в первую сессии добавлена запись:
````postgresql
postgres=# postgres=# insert into persons(first_name, second_name) values('sergey', 'sergeev');
INSERT 0 1
````
#### Во второй  сессии сделаем _select * from persons_ :
````sql
postgres=# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
````
> #### Видите ли вы новую запись и если да то почему?
>>  Нет записи, потому-что отлючен автокоммит и уровень изоляции "read committed". 
> 
#### Завершим первую транзакцию - commit: 
````sql
postgres=# commit;
COMMIT
````
#### и повторяем _select * from persons_ во второй сессии:
````sql
postgres=# commit;
COMMIT
postgres=# select * from persons 
postgres-# ;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
````
> #### Видите ли вы новую запись и если да то почему?
>> Запись появилась, так как транзакция со вставкой завершилась.
#### Завершим вторую транзакцию - commit:
````sql
postgres=# commit;
COMMIT
````
#### Начнём новую транзакцию, но уже - _repeatable read_:
````postgresql
postgres=# set transaction isolation level repeatable read;
SET
````
#### В первой сессии добавить новую запись:
````postgresql
postgres=# insert into persons(first_name, second_name) values('sveta', 'svetova');
INSERT 0 1
````
#### Сделаем _select_ во второй сессии:
```postgresql
postgres=# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```
> Видите ли вы новую запись и если да то почему?
>> Данных нет, так как транзакция в первой сессии ещё не завершена. 
#### Завершим первую транзакцию:
```postgresql
postgres=# commit;
COMMIT
```
#### Снова сделаем _select_ во второй сессии:
```postgresql
postgres=# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```
> Видите ли вы новую запись и если да то почему?
>> Данных нет, так как включен уровень изоляции _repeatable read_ в первой сессии, поэтому данные не будут видны во второй сессии.
#### Завершим вторую транзакцию, сделаем _select_ во второй сессии:
```postgresql
postgres=# commit;
COMMIT
postgres=# select * from persons;
id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
(4 rows)
```
>Видите ли вы новую запись и если да то почему?
>> Данные видны , так как начало новой транзакция во второй сессии позволяет увидеть все завершённые транзакции.

##### The end