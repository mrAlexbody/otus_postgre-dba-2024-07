
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
