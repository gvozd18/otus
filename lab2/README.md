# Underlay. OSPF

#### Цель:
Настроить OSPF для Underlay сети


#### Описание/Пошаговая инструкция выполнения домашнего задания:

В этой самостоятельной работе мы ожидаем, что вы самостоятельно:
1. Настроите OSPF в Underlay сети, для IP связанности между всеми сетевыми устройствами.
2. Зафиксируете в документации - план работы, адресное пространство, схему сети, конфигурацию устройств
3. Убедитесь в наличии IP связанности между устройствами в OSFP домене


### IP адресация 

### Адресное пространство 

Сеть DC 10.0.0.0/16

| Назначение            | CIDR              | Использование             |
| --------------------- | ----------------- | ------------------------- |
| Loopback устройств    | `10.0.1.0/24`     | Router-ID, BGP            |
| Underlay P2P          | `10.0.0.0/24`     | Spine ↔ Leaf              |
| VTEP/Overlay Loopback | `10.0.2.0/24`     | VXLAN VTEP                |
| Service               | `10.0.16.0/20`    | Клиентские/серверные сети |


## Таблицы IP адресов

| Device | Interface | IP             | Подключение |
| ------ | --------- | -------------- | ----------- |
| SPINE1 | Lo0       | `10.0.1.1/32`  | Router-ID   |
| SPINE1 | Eth1      | `10.0.0.0/31`  | LEAF1       |
| SPINE1 | Eth2      | `10.0.0.4/31`  | LEAF2       |
| SPINE1 | Eth3      | `10.0.0.8/31`  | LEAF3       |
| SPINE2 | Lo0       | `10.0.1.2/32`  | Router-ID   |
| SPINE2 | Eth1      | `10.0.0.6/31`  | LEAF2       |
| SPINE2 | Eth2      | `10.0.0.2/31`  | LEAF1       |
| SPINE2 | Eth3      | `10.0.0.10/31` | LEAF3       |
| LEAF1  | Lo0       | `10.0.1.11/32` | Router-ID   |
| LEAF1  | Eth1      | `10.0.0.1/31`  | SPINE1      |
| LEAF1  | Eth2      | `10.0.0.3/31`  | SPINE2      |
| LEAF2  | Lo0       | `10.0.1.12/32` | Router-ID   |
| LEAF2  | Eth1      | `10.0.0.5/31`  | SPINE1      |
| LEAF2  | Eth2      | `10.0.0.7/31`  | SPINE2      |
| LEAF3  | Lo0       | `10.0.1.13/32` | Router-ID   |
| LEAF3  | Eth1      | `10.0.0.9/31`  | SPINE1      |
| LEAF3  | Eth2      | `10.0.0.11/31` | SPINE2      |


## Итоговая схема

![Схема.jpg](Схема.jpg)


## План работы

1. Настровам ip адресацию на интерфейсах
2. Настраиваем протокол OSPF, все маршрутзаторы будут находится в area 0
3. Проверяем связность

<details> 

<summary> Пример конфигурации SPINE1 </summary>

```
interface Loopback0
   description Router-ID
   ip address 10.0.1.1/32
   ip ospf area 0.0.0.0

interface Ethernet1
   description P2P-to-LEAF1-Eth1
   no switchport
   ip address 10.0.0.0/31
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
   ip ospf authentication message-digest
   ip ospf message-digest-key 1 md5 OTUS
   bfd interval 300 min-rx 300 multiplier 3

interface Ethernet2
   description P2P-to-LEAF2-Eth1
   no switchport
   ip address 10.0.0.4/31
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
   ip ospf authentication message-digest
   ip ospf message-digest-key 1 md5 OTUS
   bfd interval 300 min-rx 300 multiplier 3

interface Ethernet3
   description P2P-to-LEAF3-Eth1
   no switchport
   ip address 10.0.0.8/31
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
   ip ospf authentication message-digest
   ip ospf message-digest-key 1 md5 OTUS
   bfd interval 300 min-rx 300 multiplier 3


router ospf 1
   router-id 10.0.1.1
   passive-interface Loopback0
   bfd default
!
```
</details>


## Конфигураций

[Конфигурации](https://github.com/gvozd18/otus/blob/main/lab2/configslab2.zip).

## Проверка связности
![Проверка 1](Ospf1.jpg) ![Проверка 2](Ospf2.jpg)
![Проверка 3](Ospf3.jpg)
![Проверка 4](Ospf4.jpg)
![Проверка 5](Ospf5.jpg)

