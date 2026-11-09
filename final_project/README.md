# Миграция сети ЦОД с L2 на Clos

## Материалы работы

| Материал | Ссылка |
|---|---|
| **Контрольная работа (DOCX)** | [Скачать документ](Проектная_работа_Герасимов_ЕП.docx) |
| **Презентация** | [Каталог конфигураций](configs/arista/) |



![Итоговая схема Clos ](%D0%98%D1%82%D0%BE%D0%B3%D0%BE%D0%B2%D0%B0%D1%8F%20%D1%81%D1%85%D0%B5%D0%BC%D0%B0.png)

## Постановка задачи

Исходная сеть ЦОД построена по древовидной L2-топологии с отдельными VLAN для **SSH**, **IPMI** и **DATA**. Шлюзы VLAN размещаются на L3-коммутаторах (SVI), взаимодействие между площадками обеспечивается BGP. Большие широковещательные домены, сложность L2-резервирования и зависимость от STP затрудняют горизонтальное масштабирование.

**Цель:** спроектировать миграцию к Clos и продемонстрировать передачу трафика между двумя площадками **через VXLAN**.

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

- **Site 1:** [SPINE1](S1-SPINE1.txt), [SPINE2](configs/arista/SPINE2.txt), [LEAF1](S1-LEAF1.txt), [LEAF2](S1-LEAF2.txt), [LEAF3 — Border Leaf](S1-LEAF3.txt).
- **Site 2:** [S2-SPINE1](S2-SPINE1.txt), [S2-LEAF1 — Border Leaf](S2-LEAF1.txt).


## Проверка работоспособности

| № | Хост | Скриншот |
|---|---|---|
| 1 |  S1-LEAF1 | [02-ospf-site1.png](screenshots/02-ospf-site1.png) |
| 2 |  S2-LEAF1 | [03-ospf-site2.png](screenshots/03-ospf-site2.png) |
| 3 |  S1-LEAF3 | [04-bgp-evpn-site1.png](screenshots/04-bgp-evpn-site1.png) |
| 4 |  Дамп S1-LEAF1-ETH1 | [13-wireshark-vxlan-dci.png](screenshots/13-wireshark-vxlan-dci.png) |



