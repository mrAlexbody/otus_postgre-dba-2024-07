Домашнее задание 01
=========================
"Работа с уровнями изоляции транзакции в PostgreSQL"
--------------------------

##### Цель:
+ научиться работать в Яндекс Облаке
+ научиться управлять уровнем изолции транзации в PostgreSQL и понимать особенность работы уровней read commited и repeatable read

#### Создание сети в Yandex Cloud
> yc vpc network create --name otus-net --description "otus-net"
>> id: enp6g9s9dtgu5p49drvv </br>
folder_id: b1gdmm6es3knkn350jdo </br>
created_at: "2024-08-11T18:48:59Z" </br>
name: otus-net </br>
description: otus-net </br>
default_security_group_id: enp15ppdclbcjidfo1ml </br>

#### Создание подсети в Yandex Cloud

> yc vpc subnet create --name otus-subnet --range 192.168.0.0/24 --network-name otus-net --description "otus-subnet"
>> id: e9bkd7fbtg29offoajl0 </br>
folder_id: b1gdmm6es3knkn350jdo </br>
created_at: "2024-08-11T18:49:03Z" </br>
name: otus-subnet </br>
description: otus-subnet </br>
network_id: enp6g9s9dtgu5p49drvv </br>
zone_id: ru-central1-a </br>
v4_cidr_blocks: </br> - 192.168.0.0/24 </br>

### Проверка 

 > yc vpc network list
````
|       ID              |   NAME   | 
| ----------------------|----------| 
| enp6g9s9dtgu5p49drvv  | otus-net | 
````

> yc vpc subnet list
````
|----------------------|-----------------------|----------------------|----------------|---------------+------------------+
|          ID          |         NAME          |      NETWORK ID      | ROUTE TABLE ID |     ZONE      |      RANGE       |
+----------------------+-----------------------+----------------------+----------------+---------------+------------------+
| e9bkd7fbtg29offoajl0 | otus-subnet           | enp6g9s9dtgu5p49drvv |                | ru-central1-a | [192.168.0.0/24] |
+----------------------+-----------------------+----------------------+----------------+---------------+------------------+
````
### Создание инстанс виртуальной машины в Yandex CLoud c добавление своего ключа
> PS C:\Users\Alexander> ssh-keygen.exe -t rsa -b 2048
> 
> yc compute instance create --name otus-vm --hostname otus-vm --cores 2 --memory 4 --create-boot-disk size=15G,type=network-hdd,image-folder-id=standard-images,image-family=ubuntu-2004-lts --network-interface subnet-name=otus-subnet,nat-ip-version=ipv4 --ssh-key C:\Users\alexbody\.ssh\yc_key.pub
````
done (40s)
id: fhm6sm8jr1r1quk8d836
folder_id: b1gdmm6es3knkn350jdo
created_at: "2024-08-11T18:51:31Z"
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
  device_name: fhmcq4ku1er8l5c4h50a
  auto_delete: true
  disk_id: fhmcq4ku1er8l5c4h50a
network_interfaces:
  - index: "0"
    mac_address: d0:0d:6e:59:13:d8
    subnet_id: e9bkd7fbtg29offoajl0
    primary_v4_address:
      address: 192.168.0.17
      one_to_one_nat:
        address: 89.169.147.149
        ip_version: IPV4
serial_port_settings:
  ssh_authorization: INSTANCE_METADATA
gpu_settings: {}
fqdn: otus-vm.ru-central1.internal
scheduling_policy: {}
network_settings:
  type: STANDARD
placement_policy: {}
````
