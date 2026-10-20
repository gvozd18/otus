# Отчет: VXLAN. L3VNI и EVPN Multihoming

## Цель

Настроить маршрутизацию в рамках Overlay между клиентами, а также обеспечить отказоустойчивость и агрегацию каналов на стороне клиентов с помощью технологии EVPN Multihoming (ESI-LAG).

В ходе работы было выполнено:
1. Настроен Underlay (IP-адресация, iBGP).
2. Настроен Overlay (EVPN/VXLAN) с L2VNI и L3VNI.
3. Настроена маршрутизация между клиентами (Inter-VRF Routing).
4. Настроен Multihoming (ESI-LAG) для клиентов H1 и H2.
5. Настроена агрегация каналов на стороне серверов (Ubuntu Server 16.04) с использованием `bonding`.

---

## Адресное пространство

Сеть DC `10.0.0.0/16`

| Назначение | CIDR | Использование |
|---|---|---|
| Loopback устройств | `10.0.1.0/24` | Router-ID, BGP |
| Underlay P2P | `10.0.0.0/24` | Spine ↔ Leaf |
| VTEP/Overlay Loopback | `10.0.2.0/24` | VXLAN VTEP |
| Service | `10.0.16.0/20` | Клиентские/серверные сети |

---

## Таблицы IP-адресов

### Loopback0 (Router-ID)

| Device | Interface | IP | Назначение |
|---|---|---|---|
| SPINE1 | Lo0 | `10.0.1.1/32` | Router-ID, BGP |
| SPINE2 | Lo0 | `10.0.1.2/32` | Router-ID, BGP |
| LEAF1 | Lo0 | `10.0.1.11/32` | Router-ID, BGP |
| LEAF2 | Lo0 | `10.0.1.12/32` | Router-ID, BGP |
| LEAF3 | Lo0 | `10.0.1.13/32` | Router-ID, BGP |

### Loopback1 (VTEP)

| Device | Interface | IP | Назначение |
|---|---|---|---|
| LEAF1 | Lo1 | `10.0.2.11/32` | VXLAN VTEP |
| LEAF2 | Lo1 | `10.0.2.12/32` | VXLAN VTEP |
| LEAF3 | Lo1 | `10.0.2.13/32` | VXLAN VTEP |

### Underlay P2P (Spine ↔ Leaf)

| Device | Interface | IP | Подключение |
|---|---|---|---|
| SPINE1 | Eth1 | `10.0.0.0/31` | LEAF1 |
| SPINE1 | Eth2 | `10.0.0.4/31` | LEAF2 |
| SPINE1 | Eth3 | `10.0.0.8/31` | LEAF3 |
| SPINE2 | Eth1 | `10.0.0.2/31` | LEAF1 |
| SPINE2 | Eth2 | `10.0.0.6/31` | LEAF2 |
| SPINE2 | Eth3 | `10.0.0.10/31` | LEAF3 |
| LEAF1 | Eth1 | `10.0.0.1/31` | SPINE1 |
| LEAF1 | Eth2 | `10.0.0.3/31` | SPINE2 |
| LEAF2 | Eth1 | `10.0.0.5/31` | SPINE1 |
| LEAF2 | Eth2 | `10.0.0.7/31` | SPINE2 |
| LEAF3 | Eth1 | `10.0.0.9/31` | SPINE1 |
| LEAF3 | Eth2 | `10.0.0.11/31` | SPINE2 |

### Service (клиенты)

| Хост | Leaf | VRF | VLAN | Interface | IP | Шлюз |
|---|---|---|---|---|---|---|
| H1 | LEAF1 + LEAF2 | BLUE | 10 | Eth3 (LEAF1), Eth4 (LEAF2) | `10.0.16.101/24` | `10.0.16.1` |
| H2 | LEAF2 + LEAF3 | RED | 20 | Eth3 (LEAF2), Eth3 (LEAF3) | `10.0.17.101/24` | `10.0.17.1` |

---

## EVPN / VXLAN — параметры

### L2VNI (по одному на клиента)

| Параметр | BLUE (H1) | RED (H2) |
|---|---|---|
| VLAN | 10 | 20 |
| L2VNI | 10010 | 10020 |
| L2 RD | `10.0.2.11:10010` | `10.0.2.12:10020` |
| L2 RT | `65000:10010` | `65000:10020` |
| SVI | `Vlan10` | `Vlan20` |
| Подсеть | `10.0.16.0/24` | `10.0.17.0/24` |
| Anycast SVI | `10.0.16.1/24` | `10.0.17.1/24` |

### L3VNI (по одному на VRF)

| Параметр | BLUE | RED |
|---|---|---|
| VRF | BLUE | RED |
| L3VNI | 10000 | 20000 |
| L3 RD | `10.0.2.11:10000` | `10.0.2.12:20000` |
| L3 RT | `65000:10000` | `65000:20000` |
| VTEP | LEAF1 (10.0.2.11) | LEAF2 (10.0.2.12) |

### AS / общие параметры

| Параметр | Значение |
|---|---|
| AS | `65000` |
| Underlay | iBGP, Spine — RR |
| Overlay | iBGP EVPN по Loopback0, Spine — RR |
| VTEP source | Loopback1 |

---

## Схема сети

*(Здесь вставляется ваша картинка схемы из EVE-NG, где H1 подключен к LEAF1 и LEAF2, а H2 — к LEAF2 и LEAF3)*

---

## План работы

### Часть 1. Underlay и базовая настройка
1. Настройка IP-адресации на физических интерфейсах и Loopback0.
2. Настройка iBGP в AS 65000 (Spine — Route Reflector).

### Часть 2. Overlay (EVPN/VXLAN) с L3VNI
3. Настройка Loopback1 (`10.0.2.0/24`) как VTEP на Leaf-ах.
4. Создание VRF BLUE и RED.
5. Настройка iBGP-сессий по Loopback0 для передачи EVPN-маршрутов.
6. Настройка L2VNI: VLAN 10 → VNI 10010 (BLUE), VLAN 20 → VNI 10020 (RED).
7. Настройка L3VNI: BLUE → 10000, RED → 20000.
8. Настройка cross-import RT для маршрутизации между VRF (Route Leaking).
9. Настройка SVI в соответствующих VRF с anycast-адресами (`ip address virtual`).

### Часть 3. EVPN Multihoming (ESI-LAG)
10. Настройка клиентов (Ubuntu 16.04) с агрегацией каналов (`bonding`, режим `balance-xor`).
11. Настройка статических Port-Channel на Leaf-ах (`channel-group mode on`).
12. Привязка уникальных ESI (Ethernet Segment Identifier) к Port-Channel для H1 и H2.
13. Включение `spanning-tree portfast` на клиентских портах.

---

## Конфигурация устройств

### SPINE1

```

hostname SPINE1
!
service routing protocols model multi-agent
!
interface Ethernet1
   description LEAF1
   no switchport
   ip address 10.0.0.0/31
!
interface Ethernet2
   description LEAF2
   no switchport
   ip address 10.0.0.4/31
!
interface Ethernet3
   description LEAF3
   no switchport
   ip address 10.0.0.8/31
!
interface Loopback0
   description Router-ID
   ip address 10.0.1.1/32
!
ip routing
!
route-map RM_connect permit 10
   match interface Loopback0
!
router bgp 65000
   router-id 10.0.1.1
   maximum-paths 10
   neighbor 10.0.0.1 remote-as 65000
   neighbor 10.0.0.1 next-hop-self
   neighbor 10.0.0.1 route-reflector-client
   neighbor 10.0.0.5 remote-as 65000
   neighbor 10.0.0.5 next-hop-self
   neighbor 10.0.0.5 route-reflector-client
   neighbor 10.0.0.9 remote-as 65000
   neighbor 10.0.0.9 next-hop-self
   neighbor 10.0.0.9 route-reflector-client
   neighbor 10.0.1.11 remote-as 65000
   neighbor 10.0.1.11 update-source Loopback0
   neighbor 10.0.1.11 route-reflector-client
   neighbor 10.0.1.11 send-community extended
   neighbor 10.0.1.12 remote-as 65000
   neighbor 10.0.1.12 update-source Loopback0
   neighbor 10.0.1.12 route-reflector-client
   neighbor 10.0.1.12 send-community extended
   neighbor 10.0.1.13 remote-as 65000
   neighbor 10.0.1.13 update-source Loopback0
   neighbor 10.0.1.13 route-reflector-client
   neighbor 10.0.1.13 send-community extended
   redistribute connected route-map RM_connect
   !
   address-family evpn
      neighbor 10.0.1.11 activate
      neighbor 10.0.1.12 activate
      neighbor 10.0.1.13 activate
   !
   address-family ipv4
      neighbor 10.0.0.1 activate
      neighbor 10.0.0.5 activate
      neighbor 10.0.0.9 activate
   exit
exit
end

```

### SPINE2

hostname SPINE2
!
service routing protocols model multi-agent
!
interface Ethernet1
   description LEAF1
   no switchport
   ip address 10.0.0.2/31
!
interface Ethernet2
   description LEAF2
   no switchport
   ip address 10.0.0.6/31
!
interface Ethernet3
   description LEAF3
   no switchport
   ip address 10.0.0.10/31
!
interface Loopback0
   description Router-ID
   ip address 10.0.1.2/32
!
ip routing
!
route-map RM_connect permit 10
   match interface Loopback0
!
router bgp 65000
   router-id 10.0.1.2
   maximum-paths 10
   neighbor 10.0.0.3 remote-as 65000
   neighbor 10.0.0.3 next-hop-self
   neighbor 10.0.0.3 route-reflector-client
   neighbor 10.0.0.7 remote-as 65000
   neighbor 10.0.0.7 next-hop-self
   neighbor 10.0.0.7 route-reflector-client
   neighbor 10.0.0.11 remote-as 65000
   neighbor 10.0.0.11 next-hop-self
   neighbor 10.0.0.11 route-reflector-client
   neighbor 10.0.1.11 remote-as 65000
   neighbor 10.0.1.11 update-source Loopback0
   neighbor 10.0.1.11 route-reflector-client
   neighbor 10.0.1.11 send-community extended
   neighbor 10.0.1.12 remote-as 65000
   neighbor 10.0.1.12 update-source Loopback0
   neighbor 10.0.1.12 route-reflector-client
   neighbor 10.0.1.12 send-community extended
   neighbor 10.0.1.13 remote-as 65000
   neighbor 10.0.1.13 update-source Loopback0
   neighbor 10.0.1.13 route-reflector-client
   neighbor 10.0.1.13 send-community extended
   redistribute connected route-map RM_connect
   !
   address-family evpn
      neighbor 10.0.1.11 activate
      neighbor 10.0.1.12 activate
      neighbor 10.0.1.13 activate
   !
   address-family ipv4
      neighbor 10.0.0.3 activate
      neighbor 10.0.0.7 activate
      neighbor 10.0.0.11 activate
   exit
exit
end
```

### LEAF1

```

hostname LEAF1
!
service routing protocols model multi-agent
!
vlan 10
   name BLUE-ZONE
!
vrf instance BLUE
!
ip routing
ip routing vrf BLUE
!
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
! --- Настройка H1 (Multihoming) ---
interface Ethernet3
   description TO-H1-LINK1
   channel-group 10 mode on
!
interface Port-Channel10
   description H1-MULTIHOMED
   switchport access vlan 10
   spanning-tree portfast
   evpn ethernet-segment
      identifier 0000:0000:0000:0000:0010
      route-target import 00:00:00:00:00:10
!
interface Loopback0
   description Router-ID
   ip address 10.0.1.11/32
!
interface Loopback1
   description VTEP
   ip address 10.0.2.11/32
!
interface Vlan10
   vrf BLUE
   ip address virtual 10.0.16.1/24
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vrf BLUE vni 10000
   vxlan vlan 10 flood vtep 10.0.2.12 10.0.2.13
!
route-map RM_connect permit 10
   match interface Loopback0
!
route-map RM_connect permit 20
   match interface Loopback1
!
router bgp 65000
   router-id 10.0.1.11
   maximum-paths 10
   neighbor 10.0.0.0 remote-as 65000
   neighbor 10.0.0.2 remote-as 65000
   neighbor 10.0.1.1 remote-as 65000
   neighbor 10.0.1.1 update-source Loopback0
   neighbor 10.0.1.1 send-community extended
   neighbor 10.0.1.2 remote-as 65000
   neighbor 10.0.1.2 update-source Loopback0
   neighbor 10.0.1.2 send-community extended
   redistribute connected route-map RM_connect
   !
   vlan 10
      rd 10.0.2.11:10010
      route-target both 65000:10010
      redistribute learned
   !
   vrf BLUE
      rd 10.0.2.11:10000
      route-target both 65000:10000
      route-target import evpn 65000:20000
      redistribute connected
   !
   address-family evpn
      neighbor 10.0.1.1 activate
      neighbor 10.0.1.2 activate
   !
   address-family ipv4
      neighbor 10.0.0.0 activate
      neighbor 10.0.0.2 activate
   exit
exit
end

```

### LEAF2

```

hostname LEAF2
!
service routing protocols model multi-agent
!
vlan 10
   name BLUE-ZONE
vlan 20
   name RED-ZONE
!
vrf instance BLUE
vrf instance RED
!
ip routing
ip routing vrf BLUE
ip routing vrf RED
!
interface Ethernet1
   description SPINE1
   no switchport
   ip address 10.0.0.5/31
!
interface Ethernet2
   description SPINE2
   no switchport
   ip address 10.0.0.7/31
!
! --- Настройка H1 (Multihoming) ---
interface Ethernet4
   description TO-H1-LINK2
   channel-group 10 mode on
!
interface Port-Channel10
   description H1-MULTIHOMED
   switchport access vlan 10
   spanning-tree portfast
   evpn ethernet-segment
      identifier 0000:0000:0000:0000:0010
      route-target import 00:00:00:00:00:10
!
! --- Настройка H2 (Multihoming) ---
interface Ethernet3
   description TO-H2-LINK1
   channel-group 20 mode on
!
interface Port-Channel20
   description H2-MULTIHOMED
   switchport access vlan 20
   spanning-tree portfast
   evpn ethernet-segment
      identifier 0000:0000:0000:0000:0020
      route-target import 00:00:00:00:00:20
!
interface Loopback0
   description Router-ID
   ip address 10.0.1.12/32
!
interface Loopback1
   description VTEP
   ip address 10.0.2.12/32
!
interface Vlan10
   vrf BLUE
   ip address virtual 10.0.16.1/24
!
interface Vlan20
   vrf RED
   ip address virtual 10.0.17.1/24
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
   vxlan vrf BLUE vni 10000
   vxlan vrf RED vni 20000
   vxlan vlan 10 flood vtep 10.0.2.11 10.0.2.13
   vxlan vlan 20 flood vtep 10.0.2.11 10.0.2.13
!
route-map RM_connect permit 10
   match interface Loopback0
!
route-map RM_connect permit 20
   match interface Loopback1
!
router bgp 65000
   router-id 10.0.1.12
   maximum-paths 10
   neighbor 10.0.0.4 remote-as 65000
   neighbor 10.0.0.6 remote-as 65000
   neighbor 10.0.1.1 remote-as 65000
   neighbor 10.0.1.1 update-source Loopback0
   neighbor 10.0.1.1 send-community extended
   neighbor 10.0.1.2 remote-as 65000
   neighbor 10.0.1.2 update-source Loopback0
   neighbor 10.0.1.2 send-community extended
   redistribute connected route-map RM_connect
   !
   vlan 10
      rd 10.0.2.12:10010
      route-target both 65000:10010
      redistribute learned
   !
   vlan 20
      rd 10.0.2.12:10020
      route-target both 65000:10020
      redistribute learned
   !
   vrf BLUE
      rd 10.0.2.12:10000
      route-target both 65000:10000
      route-target import evpn 65000:20000
      redistribute connected
   !
   vrf RED
      rd 10.0.2.12:20000
      route-target both 65000:20000
      route-target import evpn 65000:10000
      redistribute connected
   !
   address-family evpn
      neighbor 10.0.1.1 activate
      neighbor 10.0.1.2 activate
   !
   address-family ipv4
      neighbor 10.0.0.4 activate
      neighbor 10.0.0.6 activate
   exit
exit
end

```

### LEAF3

```

hostname LEAF3
!
service routing protocols model multi-agent
!
vlan 20
   name RED-ZONE
!
vrf instance RED
!
ip routing
ip routing vrf RED
!
interface Ethernet1
   description SPINE1
   no switchport
   ip address 10.0.0.9/31
!
interface Ethernet2
   description SPINE2
   no switchport
   ip address 10.0.0.11/31
!
! --- Настройка H2 (Multihoming) ---
interface Ethernet3
   description TO-H2-LINK2
   channel-group 20 mode on
!
interface Port-Channel20
   description H2-MULTIHOMED
   switchport access vlan 20
   spanning-tree portfast
   evpn ethernet-segment
      identifier 0000:0000:0000:0000:0020
      route-target import 00:00:00:00:00:20
!
interface Loopback0
   description Router-ID
   ip address 10.0.1.13/32
!
interface Loopback1
   description VTEP
   ip address 10.0.2.13/32
!
interface Vlan20
   vrf RED
   ip address virtual 10.0.17.1/24
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 20 vni 10020
   vxlan vrf RED vni 20000
   vxlan vlan 20 flood vtep 10.0.2.11 10.0.2.12
!
route-map RM_connect permit 10
   match interface Loopback0
!
route-map RM_connect permit 20
   match interface Loopback1
!
router bgp 65000
   router-id 10.0.1.13
   maximum-paths 10
   neighbor 10.0.0.8 remote-as 65000
   neighbor 10.0.0.10 remote-as 65000
   neighbor 10.0.1.1 remote-as 65000
   neighbor 10.0.1.1 update-source Loopback0
   neighbor 10.0.1.1 send-community extended
   neighbor 10.0.1.2 remote-as 65000
   neighbor 10.0.1.2 update-source Loopback0
   neighbor 10.0.1.2 send-community extended
   redistribute connected route-map RM_connect
   !
   vlan 20
      rd 10.0.2.13:10020
      route-target both 65000:10020
      redistribute learned
   !
   vrf RED
      rd 10.0.2.13:20000
      route-target both 65000:20000
      route-target import evpn 65000:10000
      redistribute connected
   !
   address-family evpn
      neighbor 10.0.1.1 activate
      neighbor 10.0.1.2 activate
   !
   address-family ipv4
      neighbor 10.0.0.8 activate
      neighbor 10.0.0.10 activate
   exit
exit
end

```

##Соседство и маршруты
