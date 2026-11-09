# Миграция сети ЦОД с L2 на Clos: OSPF + EVPN/VXLAN + Multi-Site DCI

> Контрольная работа и лабораторный стенд EVE-NG на **Arista EOS 4.29.2F**. Рассматривается переход от древовидной L2-топологии к Leaf–Spine (Clos) с OSPF underlay, BGP EVPN/VXLAN overlay, ESI multihoming и VXLAN между двумя площадками.

## Материалы работы

| Материал | Ссылка |
|---|---|
| **Контрольная работа (DOCX)** | [Скачать документ](docs/Контрольная_работа_Clos_VXLAN_между_Site.docx) |
| **Конфигурации Arista EOS** | [Каталог конфигураций](configs/arista/) |
| **Скрипты тестовых Ubuntu 16.04** | [Каталог скриптов](configs/ubuntu/) |
| **Итоговая топология** | [Посмотреть схему](docs/topology.png) |
| **Карта физических соединений** | [lab_connections.csv](docs/lab_connections.csv) |
| **Методика испытаний** | [validation.md](docs/validation.md) |
| **Дополнительное текстовое описание** | [assignment.md](docs/assignment.md) |
| **Скриншоты проверки** | [Раздел «Подтверждение работы»](#подтверждение-работы-стенда) |

![Итоговая схема Clos с двумя Site](docs/topology.png)

## Постановка задачи

Исходная сеть ЦОД построена по древовидной L2-топологии с отдельными VLAN для **SSH**, **IPMI** и **DATA**. Шлюзы VLAN размещаются на L3-коммутаторах (SVI), взаимодействие между площадками обеспечивается BGP. Большие широковещательные домены, сложность L2-резервирования и зависимость от STP затрудняют горизонтальное масштабирование.

**Цель:** спроектировать миграцию к Clos и продемонстрировать передачу трафика между двумя площадками **через VXLAN**, не только через обычную IPv4-маршрутизацию.

## Архитектура стенда

| Компонент | Site 1 | Site 2 |
|---|---|---|
| BGP AS | `65001` | `65002` |
| Spine (EVPN Route Reflector) | 2 | 1 |
| Leaf / VTEP | 3 | 1 |
| Тестовые Linux-хосты | 3 | 1 |
| Underlay | OSPF Area 0 | OSPF Area 0 |
| Overlay внутри площадки | iBGP EVPN / VXLAN | iBGP EVPN / VXLAN |
| DATA VLAN | `130` | `130` |
| DATA подсеть | `10.1.18.0/24` | `10.2.18.0/24` |
| Anycast Gateway | `10.1.18.1` | `10.2.18.1` |
| DATA L2VNI | `10130` | `20130` |
| Общий L3VNI / RT | `50130` / `65000:50130` | `50130` / `65000:50130` |

**DCI:** `S1-LEAF3 Ethernet4 (172.31.12.0/31)` ↔ `S2-LEAF1 Ethernet2 (172.31.12.1/31)`.

- **Control plane на DCI:** eBGP EVPN (маршруты Type-5) и отдельный eBGP IPv4 для доступности VTEP `/32`.
- **Data plane на DCI:** VXLAN/UDP 4789, **L3VNI 50130**; IP-префиксы DATA передаются в EVPN, а не как обычные IPv4 DATA-маршруты DCI.
- **Site 1:** ESI all-active multihoming сервера HOST-ESI к S1-LEAF1 и S1-LEAF2 по LACP.
- **Изоляция:** SSH (VLAN 110) и IPMI (VLAN 120) остаются локальными; L2 DATA между площадками не растягивается.

### Адреса тестовых хостов

| Хост | Адрес DATA | Подключение |
|---|---|---|
| HOST-ESI | `10.1.18.10/24` | два Leaf, LACP / ESI |
| TEST1 | `10.1.18.21/24` | S1-LEAF1 |
| TEST2 | `10.1.18.22/24` | S1-LEAF3 |
| S2-TEST | `10.2.18.21/24` | S2-LEAF1 |

**Важно для S2-TEST:** должен быть установлен маршрут по умолчанию через `10.2.18.1`. В ходе диагностики отсутствие default route мешало ответу на ping из Site 1; после его добавления межсайтовый ping заработал:

```bash
sudo ip route replace default via 10.2.18.1 dev "$DEV"
```

Здесь `$DEV` — фактический интерфейс S2-TEST, например `eth0`. После запуска проверьте `ip route`.

## Конфигурации и развёртывание

- **Site 1:** [SPINE1](configs/arista/SPINE1.cfg), [SPINE2](configs/arista/SPINE2.cfg), [LEAF1](configs/arista/LEAF1.cfg), [LEAF2](configs/arista/LEAF2.cfg), [LEAF3 — Border Leaf](configs/arista/LEAF3.cfg).
- **Site 2:** [S2-SPINE1](configs/arista/S2-SPINE1.cfg), [S2-LEAF1 — Border Leaf](configs/arista/S2-LEAF1.cfg).
- **Linux:** [HOST-ESI](configs/ubuntu/HOST-ESI-bond.sh), [TEST1](configs/ubuntu/TEST1-vlans.sh), [TEST2](configs/ubuntu/TEST2-vlans.sh), [S2-TEST](configs/ubuntu/S2-TEST-data.sh).

Соберите соединения в EVE-NG по [таблице портов](docs/lab_connections.csv), загрузите соответствующие конфигурации на Arista и задайте IP-адреса на хостах. Скрипты Ubuntu рассчитаны на учебную Ubuntu 16.04; названия интерфейсов могут отличаться.

## Подтверждение работы стенда

> **Скриншоты не включены в этот шаблон.** Здесь перечислены реальные команды и имена файлов, которые необходимо сохранить после проверки стенда. Не следует выдавать ожидаемые результаты за полученные. После добавления PNG-файлов в `screenshots/` ссылки ниже станут активными.

| № | Проверка | Где выполнить | Команда / действие | Скриншот |
|---|---|---|---|---|
| 01 | Версия EOS | S1-LEAF3 | `show version` | [01-eos-version.png](screenshots/01-eos-version.png) |
| 02 | OSPF underlay | S1-SPINE1 | `show ip ospf neighbor` | [02-ospf-site1.png](screenshots/02-ospf-site1.png) |
| 03 | OSPF Site 2 | S2-LEAF1 | `show ip ospf neighbor` | [03-ospf-site2.png](screenshots/03-ospf-site2.png) |
| 04 | BGP EVPN на Site 1 | S1-LEAF3 | `show bgp evpn summary` | [04-bgp-evpn-site1.png](screenshots/04-bgp-evpn-site1.png) |
| 05 | Межсайтовый eBGP EVPN | S2-LEAF1 | `show bgp evpn summary` | [05-bgp-evpn-dci.png](screenshots/05-bgp-evpn-dci.png) |
| 06 | EVPN Type-5 | S2-LEAF1 | `show bgp evpn route-type ip-prefix ipv4` | [06-evpn-type5.png](screenshots/06-evpn-type5.png) |
| 07 | Маршрут Site 2 через VXLAN | S1-LEAF3 | `show ip route vrf VRF-DATA` | [07-vrf-route-site1.png](screenshots/07-vrf-route-site1.png) |
| 08 | Обратный маршрут Site 1 | S2-LEAF1 | `show ip route vrf VRF-DATA` | [08-vrf-route-site2.png](screenshots/08-vrf-route-site2.png) |
| 09 | Настройка VXLAN/VNI | S1-LEAF3 | `show vxlan vni` | [09-vxlan-vni.png](screenshots/09-vxlan-vni.png) |
| 10 | ESI/LACP multihoming | S1-LEAF1 / HOST-ESI | `show lacp neighbor`, `cat /proc/net/bonding/bond0` | [10-esi-lacp.png](screenshots/10-esi-lacp.png) |
| 11 | Ping через DCI | TEST2 | `ping -c 5 10.2.18.21` | [11-ping-inter-site.png](screenshots/11-ping-inter-site.png) |
| 12 | Обратный маршрут Ubuntu | S2-TEST | `ip route`, `ping -c 5 10.1.18.22` | [12-s2-host-route-and-ping.png](screenshots/12-s2-host-route-and-ping.png) |
| 13 | **Факт VXLAN между Site** | EVE-NG Capture на DCI | Wireshark: `udp.port == 4789`, раскрыть VNI | [13-wireshark-vxlan-dci.png](screenshots/13-wireshark-vxlan-dci.png) |

### На что обратить внимание при скриншотах

1. **BGP:** `Established` и принимаемые EVPN-префиксы между `172.31.12.0` и `172.31.12.1`.
2. **Type-5:** префиксы `10.1.18.0/24` и `10.2.18.0/24`, корректные VTEP next-hop.
3. **VRF-DATA:** на S1-LEAF3 маршрут в Site 2 через `Vxlan1`, VNI `50130`; на S2-LEAF1 обратный маршрут через VTEP Site 1.
4. **Ubuntu:** корректные адреса TEST1 `.21`, TEST2 `.22`, S2-TEST `10.2.18.21` и default route на обоих концах.
5. **Wireshark (обязательно):** на DCI должен быть именно **UDP 4789**, внешний IP пакет между VTEP и **VNI 50130**, а внутри — ICMP между `10.1.18.22` и `10.2.18.21`. Один только ping доказывает IP-связность, но **не** VXLAN-инкапсуляцию.

После загрузки PNG можно дополнительно отобразить ключевые результаты прямо в README (вместо одних ссылок):

```markdown
![BGP EVPN между Site](screenshots/05-bgp-evpn-dci.png)
![Маршрут EVPN через VXLAN](screenshots/07-vrf-route-site1.png)
![VXLAN UDP 4789, VNI 50130](screenshots/13-wireshark-vxlan-dci.png)
```

## Результат и ограничения

В ходе ручной диагностики были получены `Established` для eBGP EVPN между площадками и маршруты в `VRF-DATA` через VTEP/VNI `50130`. После исправления маршрута по умолчанию на S2-TEST межсайтовый ping заработал. **Захват VXLAN-трафика на DCI пока не приложен**, поэтому отдельно подтверждать инкапсуляцию до добавления pcap/скриншота нельзя.

Site 2 состоит лишь из одного Spine и одного Leaf, DCI одинарный — это учебный стенд, **не отказоустойчивая промышленная архитектура**. Реальная поддержка отдельных функций ESI в конкретном виртуальном образе Arista и тесты отказа требуют самостоятельной проверки.

## Источники

- [Исходный адресный план (GitHub, lab1)](https://github.com/gvozd18/otus/tree/main/lab1#readme)
- [Arista EOS — VXLAN Configuration](https://www.arista.com/en/um-eos/eos-vxlan-configuration)
- [Arista EOS — BGP](https://www.arista.com/en/um-eos/eos-border-gateway-protocol-bgp)

