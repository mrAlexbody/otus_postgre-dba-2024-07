Домашнее задание 01
=========================
"Работа с уровнями изоляции транзакции в PostgreSQL"
--------------------------

### Цель:
+ научиться работать в Яндекс Облаке
+ научиться управлять уровнем изолции транзации в PostgreSQL и понимать особенность работы уровней read commited и repeatable read

### Создание сети в Yandex Cloud
````
> yc vpc network create --name otus-net --description "otus-net"
````

### Создание подсети в Yandex Cloud

````
> yc vpc subnet create --name otus-subnet --range 192.168.0.0/24 --network-name otus-net --description "otus-subnet"
````

#### Проверка 

````
> yc vpc network list

|       ID              |   NAME   | 
| ----------------------|----------| 
| enp6g9s9dtgu5p49drvv  | otus-net | 

> yc vpc subnet list

|----------------------|-----------------------|----------------------|----------------|---------------+------------------+
|          ID          |         NAME          |      NETWORK ID      | ROUTE TABLE ID |     ZONE      |      RANGE       |
+----------------------+-----------------------+----------------------+----------------+---------------+------------------+
| e9bkd7fbtg29offoajl0 | otus-subnet           | enp6g9s9dtgu5p49drvv |                | ru-central1-a | [192.168.0.0/24] |
+----------------------+-----------------------+----------------------+----------------+---------------+------------------+
````
### Создание инстанс виртуальной машины в Yandex CLoud c добавление своего ключа
````
> ssh-keygen.exe -t rsa -b 2048
> yc compute instance create --name otus-vm --hostname otus-vm --cores 2 --memory 4 --create-boot-disk size=15G,type=network-hdd,image-folder-id=standard-images,image-family=ubuntu-2004-lts --network-interface subnet-name=otus-subnet,nat-ip-version=ipv4 --ssh-key C:\Users\Alexander\.ssh\yc_key.pub

done (40s)
````
