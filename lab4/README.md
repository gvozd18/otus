# Underlay. BGP

#### Цель:
настроить BGP для Underlay сети.


#### Описание/Пошаговая инструкция выполнения домашнего задания:

В этой самостоятельной работе мы ожидаем, что вы самостоятельно:
1. Настроите BGP в Underlay сети, для IP связанности между всеми сетевыми устройствами. iBGP или eBGP - решать вам! 
2. Зафиксируете в документации - план работы, адресное пространство, схему сети, конфигурацию устройств
3. Убедитесь в наличии IP связанности между устройствами в BGP домене


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
2. Настраиваем протокол iBGP, все маршрутзаторы будут находится в AS 65000 и Multipath
3. Проверяем связность

<details> 

<summary> Пример конфигурации SPINE1 </summary>

```
interface Ethernet1
   description SPINE1
   no switchport
   ip address 10.0.0.1/31
!
interface Ethernet2
   description SPINE2
   no switchport
   ip address 10.0.0.3/31
!
interface Ethernet3
!
interface Ethernet4
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback0
   description Router-ID
   ip address 10.0.1.11/32
!
interface Management1
!
ip routing
!
route-map RM_connect permit 10
   match interface Loopback0
!
router bgp 65000
   router-id 10.0.1.11
   maximum-paths 10
   neighbor 10.0.0.0 remote-as 65000
   neighbor 10.0.0.2 remote-as 65000
   redistribute connected route-map RM_connect
!
end


```
</details>



## Конфигураций

[Конфигурации](https://github.com/gvozd18/otus/blob/main/lab4/lab4.zip).

## Проверка связности

![Проверка](IBGP_ECMP.jpg) 

## Соседство и маршруты

![Leaf 1](Leaf1.png)
![Leaf 2](Leaf2.png)
![Leaf 3](Leaf3.png)
