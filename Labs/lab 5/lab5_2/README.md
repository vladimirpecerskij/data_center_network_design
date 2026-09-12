# Лабораторная работа №5. VXLAN. L2 VNI

**Цель работы:** Настроить Overlay-сеть на основе VXLAN EVPN для обеспечения L2-связанности между клиентами, подключёнными к разным Leaf-коммутаторам.

---

## 1. Топология сети

![Топология](./lab5_2.png)

В среде **PNET Lab** используется та же физическая топология, что и в лабораторной работе №4:

- **Super-Spine (уровень 1):** 1 коммутатор **Cisco Nexus 9000** (образ NX-OS).
- **Spine (уровень 2):** 3 коммутатора **Arista vEOS** (Spine-01, Spine-02, Spine-03).
- **Leaf (уровень 3):** 3 коммутатора **Arista vEOS** (Leaf-01, Leaf-02, Leaf-03).

Underlay-сеть уже настроена с использованием eBGP, BFD и MD5-аутентификации. Все Loopback-адреса (используемые как VTEP) доступны друг другу.

> **Примечание:** В этой работе Super-Spine не участвует в VXLAN-инкапсуляции, но должен передавать BGP EVPN-маршруты между Spine.

---

## 2. План работ

1. **Проверка Underlay** – убедиться в IP-связности между VTEP (Loopback) всех устройств.
2. **Планирование Overlay** – выбрать VLAN, VNI, подсеть для клиентов.
3. **Настройка BGP для EVPN:**
   - На Spine – настроить передачу EVPN-маршрутов между Leaf.
   - На Leaf – включить адресное семейство EVPN и активировать соседей (Spine).
4. **Настройка VXLAN на Leaf:**
   - Создать VLAN и VNI.
   - Настроить интерфейс VXLAN (VTEP) с источником Loopback.
   - Определить EVPN-сервис для VNI (RD, RT, redistribute learned).
5. **Подключение клиентов** – настроить порты Leaf в режиме access VLAN.
6. **Верификация** – проверить BGP EVPN-сессии, таблицы MAC/VNI и связность между хостами.
7. **Настройка ECMP** — включены maximum-paths на всех устройствах для использования нескольких равнозначных путей.

---

## 3. Адресное пространство Overlay

### 3.1. VTEP (Loopback) адреса

| Устройство | Роль | Loopback0 (VTEP) |
|:---|:---|:---|
| **Leaf-01** | Leaf | 10.0.4.1/32 |
| **Leaf-02** | Leaf | 10.0.5.1/32 |
| **Leaf-03** | Leaf | 10.0.6.1/32 |
| **Spine-01** | Spine | 10.0.1.1/32 |
| **Spine-02** | Spine | 10.0.2.1/32 |
| **Spine-03** | Spine | 10.0.3.1/32 |

### 3.2. Параметры L2-сервиса (VNI)

| Параметр | Значение | Описание |
|:---|:---|:---|
| **VLAN** | 10 | Клиентский VLAN на всех Leaf |
| **VNI** | 10100 | Идентификатор VXLAN-сегмента |
| **Подсеть клиентов** | 172.16.10.0/24 | IPv4-сеть для хостов |
| **VTEP Source Interface** | Loopback0 | IP-адрес для построения VXLAN-туннелей |

> **Примечание:** В данной работе используется **чистая L2-связность**. SVI и Anycast Gateway **не настраиваются** — трафик между хостами передаётся на втором уровне через VXLAN.

### 3.3. Распределение IP-адресов и MAC-адресов хостов

| Хост | Leaf | Интерфейс | IP-адрес / Маска | MAC-адрес |
|:---|:---|:---|:---|:---|
| Host-1 | Leaf-01 | Eth4 | 172.16.10.11/24 | 0050.7966.680d |
| Host-2 | Leaf-02 | Eth4 | 172.16.10.12/24 | 0050.7966.680e |
| Host-3 | Leaf-03 | Eth4 | 172.16.10.13/24 | 0050.7966.680f |

---

## 4. Конфигурации устройств

> **Важно для Arista EOS:** Перед настройкой EVPN необходимо включить модель маршрутизации multi-agent:
> service routing protocols model multi-agent
> После ввода этой команды требуется **перезагрузка** устройства.

### 4.1. Super-Spine (Cisco Nexus 9000)
```
hostname NEXUS-9000
!
nv overlay evpn
feature bgp
feature bfd
!
interface Ethernet1/1
no switchport
ip address 10.1.1.0/31
bfd interval 300 min_rx 300 multiplier 3
no shutdown
!
interface Ethernet1/2
no switchport
ip address 10.1.1.2/31
bfd interval 300 min_rx 300 multiplier 3
no shutdown
!
interface Ethernet1/3
no switchport
ip address 10.1.1.4/31
bfd interval 300 min_rx 300 multiplier 3
no shutdown
!
interface Loopback0
ip address 10.0.0.1/32
!
router bgp 65000
router-id 10.0.0.1

address-family ipv4 unicast
maximum-paths 3
redistribute connected
!

address-family l2vpn evpn
retain route-target all
!

neighbor 10.1.1.1 remote-as 65001
bfd
password 0 MySecretKey123
address-family ipv4 unicast
disable-peer-as-check
!
address-family l2vpn evpn
route-reflector-client
!

neighbor 10.1.1.3 remote-as 65002
bfd
password 0 MySecretKey123
address-family ipv4 unicast
disable-peer-as-check
!
address-family l2vpn evpn
route-reflector-client
!

neighbor 10.1.1.5 remote-as 65003
bfd
password 0 MySecretKey123
address-family ipv4 unicast
disable-peer-as-check
!
address-family l2vpn evpn
route-reflector-client

```

**Примечание:** Команда `retain route-target all` гарантирует, что Super-Spine будет передавать все EVPN-маршруты между клиентами.


### 4.2. Конфигурация Spine (Arista vEOS)

Каждый Spine должен передавать EVPN-маршруты между Leaf-коммутаторами.

**Spine-01 (AS 65001)**
```
hostname Spine-01
!
service routing protocols model multi-agent
!
interface Ethernet1
no switchport
ip address 10.1.1.1/31
bfd interval 300 min-rx 300 multiplier 3
no shutdown
!
interface Ethernet2
no switchport
ip address 10.1.2.0/31
bfd interval 300 min-rx 300 multiplier 3
no shutdown
!
interface Ethernet3
no switchport
ip address 10.1.2.2/31
bfd interval 300 min-rx 300 multiplier 3
no shutdown
!
interface Ethernet4
no switchport
ip address 10.1.2.4/31
bfd interval 300 min-rx 300 multiplier 3
no shutdown
!
interface Loopback0
ip address 10.0.1.1/32
!
router bgp 65001
router-id 10.0.1.1
maximum-paths 3 ecmp 3
!
address-family ipv4
maximum-paths 3
redistribute connected
!
neighbor 10.1.1.0 remote-as 65000
neighbor 10.1.1.0 bfd
neighbor 10.1.1.0 password MySecretKey123
neighbor 10.1.1.0 send-community extended
!
neighbor 10.1.2.1 remote-as 65004
neighbor 10.1.2.1 bfd
neighbor 10.1.2.1 password MySecretKey123
neighbor 10.1.2.1 send-community extended
!
neighbor 10.1.2.3 remote-as 65005
neighbor 10.1.2.3 bfd
neighbor 10.1.2.3 password MySecretKey123
neighbor 10.1.2.3 send-community extended
!
neighbor 10.1.2.5 remote-as 65006
neighbor 10.1.2.5 bfd
neighbor 10.1.2.5 password MySecretKey123
neighbor 10.1.2.5 send-community extended
!
address-family evpn
neighbor 10.1.1.0 activate
neighbor 10.1.2.1 activate
neighbor 10.1.2.3 activate
neighbor 10.1.2.5 activate
```

**Spine-02 (AS 65002)**
```
hostname Spine-02
!
service routing protocols model multi-agent
!
interface Ethernet1
no switchport
ip address 10.1.1.3/31
bfd interval 300 min-rx 300 multiplier 3
no shutdown
!
interface Ethernet2
no switchport
ip address 10.1.2.6/31
bfd interval 300 min-rx 300 multiplier 3
no shutdown
!
interface Ethernet3
no switchport
ip address 10.1.2.8/31
bfd interval 300 min-rx 300 multiplier 3
no shutdown
!
interface Ethernet4
no switchport
ip address 10.1.2.10/31
bfd interval 300 min-rx 300 multiplier 3
no shutdown
!
interface Loopback0
ip address 10.0.2.1/32
!
router bgp 65002
router-id 10.0.2.1
maximum-paths 3 ecmp 3
!
address-family ipv4
maximum-paths 3
redistribute connected
!
neighbor 10.1.1.2 remote-as 65000
neighbor 10.1.1.2 bfd
neighbor 10.1.1.2 password MySecretKey123
neighbor 10.1.1.2 send-community extended
!
neighbor 10.1.2.7 remote-as 65004
neighbor 10.1.2.7 bfd
neighbor 10.1.2.7 password MySecretKey123
neighbor 10.1.2.7 send-community extended
!
neighbor 10.1.2.9 remote-as 65005
neighbor 10.1.2.9 bfd
neighbor 10.1.2.9 password MySecretKey123
neighbor 10.1.2.9 send-community extended
!
neighbor 10.1.2.11 remote-as 65006
neighbor 10.1.2.11 bfd
neighbor 10.1.2.11 password MySecretKey123
neighbor 10.1.2.11 send-community extended
!
address-family evpn
neighbor 10.1.1.2 activate
neighbor 10.1.2.7 activate
neighbor 10.1.2.9 activate
neighbor 10.1.2.11 activate
```

**Spine-03 (AS 65003)**
```
hostname Spine-03
!
service routing protocols model multi-agent
!
interface Ethernet1
no switchport
ip address 10.1.1.5/31
bfd interval 300 min-rx 300 multiplier 3
no shutdown
!
interface Ethernet2
no switchport
ip address 10.1.2.12/31
bfd interval 300 min-rx 300 multiplier 3
no shutdown
!
interface Ethernet3
no switchport
ip address 10.1.2.14/31
bfd interval 300 min-rx 300 multiplier 3
no shutdown
!
interface Ethernet4
no switchport
ip address 10.1.2.16/31
bfd interval 300 min-rx 300 multiplier 3
no shutdown
!
interface Loopback0
ip address 10.0.3.1/32
!
router bgp 65003
router-id 10.0.3.1
maximum-paths 3 ecmp 3
!
address-family ipv4
maximum-paths 3
redistribute connected
!
neighbor 10.1.1.4 remote-as 65000
neighbor 10.1.1.4 bfd
neighbor 10.1.1.4 password MySecretKey123
neighbor 10.1.1.4 send-community extended
!
neighbor 10.1.2.13 remote-as 65004
neighbor 10.1.2.13 bfd
neighbor 10.1.2.13 password MySecretKey123
neighbor 10.1.2.13 send-community extended
!
neighbor 10.1.2.15 remote-as 65005
neighbor 10.1.2.15 bfd
neighbor 10.1.2.15 password MySecretKey123
neighbor 10.1.2.15 send-community extended
!
neighbor 10.1.2.17 remote-as 65006
neighbor 10.1.2.17 bfd
neighbor 10.1.2.17 password MySecretKey123
neighbor 10.1.2.17 send-community extended
!
address-family evpn
neighbor 10.1.1.4 activate
neighbor 10.1.2.13 activate
neighbor 10.1.2.15 activate
neighbor 10.1.2.17 activate
```

### 4.3. Конфигурация Leaf (Arista vEOS)

**Leaf-01 (AS 65004, Loopback 10.0.4.1)**
```
hostname Leaf-01
!
service routing protocols model multi-agent
!
ip routing
!
vlan 10
name RED_ZONE
!
interface Ethernet1
description Link-to-Spine-01
no switchport
ip address 10.1.2.1/31
bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet2
description Link-to-Spine-02
no switchport
ip address 10.1.2.7/31
bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet3
description Link-to-Spine-03
no switchport
ip address 10.1.2.13/31
bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet4
description Host-1
switchport mode access
switchport access vlan 10
!
interface Loopback0
ip address 10.0.4.1/32
!
interface Vxlan1
vxlan source-interface Loopback0
vxlan udp-port 4789
vxlan vlan 10 vni 10100
!
router bgp 65004
router-id 10.0.4.1
maximum-paths 3 ecmp 3
!
neighbor 10.1.2.0 remote-as 65001
neighbor 10.1.2.0 bfd
neighbor 10.1.2.0 password MySecretKey123
neighbor 10.1.2.0 send-community extended
!
neighbor 10.1.2.6 remote-as 65002
neighbor 10.1.2.6 bfd
neighbor 10.1.2.6 password MySecretKey123
neighbor 10.1.2.6 send-community extended
!
neighbor 10.1.2.12 remote-as 65003
neighbor 10.1.2.12 bfd
neighbor 10.1.2.12 password MySecretKey123
neighbor 10.1.2.12 send-community extended
!
address-family evpn
neighbor 10.1.2.0 activate
neighbor 10.1.2.6 activate
neighbor 10.1.2.12 activate
!
address-family ipv4
neighbor 10.1.2.0 activate
neighbor 10.1.2.6 activate
neighbor 10.1.2.12 activate
network 10.0.4.1/32
!
l2vpn evpn instance 10100
rd auto
route-target import auto
route-target export auto
redistribute learned
```

**Leaf-02 (AS 65005, Loopback 10.0.5.1)**
```
hostname Leaf-02
!
service routing protocols model multi-agent
!
ip routing
!
vlan 10
name RED_ZONE
!
interface Ethernet1
description Link-to-Spine-01
no switchport
ip address 10.1.2.3/31
bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet2
description Link-to-Spine-02
no switchport
ip address 10.1.2.9/31
bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet3
description Link-to-Spine-03
no switchport
ip address 10.1.2.15/31
bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet4
description Host-2
switchport mode access
switchport access vlan 10
!
interface Loopback0
ip address 10.0.5.1/32
!
interface Vxlan1
vxlan source-interface Loopback0
vxlan udp-port 4789
vxlan vlan 10 vni 10100
!
router bgp 65005
router-id 10.0.5.1
maximum-paths 3 ecmp 3
!
neighbor 10.1.2.2 remote-as 65001
neighbor 10.1.2.2 bfd
neighbor 10.1.2.2 password MySecretKey123
neighbor 10.1.2.2 send-community extended
!
neighbor 10.1.2.8 remote-as 65002
neighbor 10.1.2.8 bfd
neighbor 10.1.2.8 password MySecretKey123
neighbor 10.1.2.8 send-community extended
!
neighbor 10.1.2.14 remote-as 65003
neighbor 10.1.2.14 bfd
neighbor 10.1.2.14 password MySecretKey123
neighbor 10.1.2.14 send-community extended
!
address-family evpn
neighbor 10.1.2.2 activate
neighbor 10.1.2.8 activate
neighbor 10.1.2.14 activate
!
address-family ipv4
neighbor 10.1.2.2 activate
neighbor 10.1.2.8 activate
neighbor 10.1.2.14 activate
network 10.0.5.1/32
!
l2vpn evpn instance 10100
rd auto
route-target import auto
route-target export auto
redistribute learned
```

**Leaf-03 (AS 65006, Loopback 10.0.6.1)**
```
hostname Leaf-03
!
service routing protocols model multi-agent
!
ip routing
!
vlan 10
name RED_ZONE
!
interface Ethernet1
description Link-to-Spine-01
no switchport
ip address 10.1.2.5/31
bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet2
description Link-to-Spine-02
no switchport
ip address 10.1.2.11/31
bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet3
description Link-to-Spine-03
no switchport
ip address 10.1.2.17/31
bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet4
description Host-3
switchport mode access
switchport access vlan 10
!
interface Loopback0
ip address 10.0.6.1/32
!
interface Vxlan1
vxlan source-interface Loopback0
vxlan udp-port 4789
vxlan vlan 10 vni 10100
!
router bgp 65006
router-id 10.0.6.1
maximum-paths 3 ecmp 3
!
neighbor 10.1.2.4 remote-as 65001
neighbor 10.1.2.4 bfd
neighbor 10.1.2.4 password MySecretKey123
neighbor 10.1.2.4 send-community extended
!
neighbor 10.1.2.10 remote-as 65002
neighbor 10.1.2.10 bfd
neighbor 10.1.2.10 password MySecretKey123
neighbor 10.1.2.10 send-community extended
!
neighbor 10.1.2.16 remote-as 65003
neighbor 10.1.2.16 bfd
neighbor 10.1.2.16 password MySecretKey123
neighbor 10.1.2.16 send-community extended
!
address-family evpn
neighbor 10.1.2.4 activate
neighbor 10.1.2.10 activate
neighbor 10.1.2.16 activate
!
address-family ipv4
neighbor 10.1.2.4 activate
neighbor 10.1.2.10 activate
neighbor 10.1.2.16 activate
network 10.0.6.1/32
!
l2vpn evpn instance 10100
rd auto
route-target import auto
route-target export auto
redistribute learned
```

**Примечания по конфигурации:**

- В интерфейсе `Vxlan1` настроено сопоставление VLAN 10 с VNI 10100.
- В BGP в адресном семействе `evpn` соседи активируются с помощью команды `activate`.
- Блок `l2vpn evpn instance 10100` определяет EVPN-сервис для VNI 10100. Команды `rd auto` и `route-target ... auto` автоматически генерируют RD и RT.
- Команда `redistribute learned` необходима для анонсирования локально изученных MAC-адресов через EVPN.
- Клиентские порты (Ethernet4) переведены в access-режим VLAN 10.
- Для корректной работы маршрутизации добавлена глобальная команда `ip routing`.

---

## 5. Верификация

### 5.1. Проверка BGP EVPN-сессий

Команда (на любом Leaf):
show bgp evpn summary

```

**Пример вывода на Leaf-01:**
BGP summary information for VRF default
Router identifier 10.0.4.1, local AS number 65004
Neighbor Status Codes: m - Under maintenance
Neighbor V AS MsgRcvd MsgSent InQ OutQ Up/Down State PfxRcd
10.1.2.0 4 65001 125 123 0 0 01:02:33 Estab 2
10.1.2.6 4 65002 124 122 0 0 01:02:28 Estab 2
10.1.2.12 4 65003 126 124 0 0 01:02:40 Estab 2
```

**Пояснение полей:**

| Параметр | Описание |
|:---|:---|
| **Neighbor** | IP-адрес соседа (Spine) |
| **AS** | Номер AS соседа |
| **MsgRcvd / MsgSent** | Количество полученных/отправленных BGP-сообщений |
| **Up/Down** | Время активности сессии |
| **State** | Должно быть `Estab` (установлена) |
| **PfxRcd** | Количество полученных EVPN-маршрутов |

### 5.2. Проверка таблицы MAC-адресов в VXLAN

Команда (на любом Leaf):
show vxlan address-table

```

**Пример вывода на Leaf-01:**
Vxlan Mac Address Table
================================================
VLAN VNI MAC Address Type Age Remote VTEP

10 10100 0050.7966.680e EVPN - 10.0.5.1
10 10100 0050.7966.680f EVPN - 10.0.6.1
```

**Пояснение:**

| Параметр | Описание |
|:---|:---|
| **VLAN** | Локальный VLAN |
| **VNI** | VXLAN-идентификатор |
| **MAC Address** | MAC-адрес клиента на удалённом Leaf |
| **Type** | `EVPN` – изучено через контрольную плоскость |
| **Remote VTEP** | IP-адрес удалённого VTEP (Leaf) |

### 5.3. Проверка EVPN-маршрутов типа 3 (IMET)

Команда (на любом Leaf):
show bgp evpn route-type imet

```

Этот маршрут анонсируется каждым VTEP и сообщает остальным, какие VNI он обслуживает.

**Пример вывода на Leaf-01:**
BGP routing table information for VRF default
Router identifier 10.0.4.1, local AS number 65004

Network Next Hop Metric LocPref Weight Path

RD: 10.0.4.1:10 imet 10.0.4.1

0 i

Ec RD: 10.0.5.1:10 imet 10.0.5.1
10.0.5.1 - 100 0 65001 65005 i

Ec RD: 10.0.6.1:10 imet 10.0.6.1
10.0.6.1 - 100 0 65001 65006 i
```

Leaf-01 видит IMET-маршруты от Leaf-02 (VTEP 10.0.5.1) и Leaf-03 (VTEP 10.0.6.1).

### 5.4. Проверка EVPN-маршрутов типа 2 (MAC/IP)

Команда (на любом Leaf):
show bgp evpn route-type mac-ip

```

**Пример вывода на Leaf-01:**
Network Next Hop Metric LocPref Weight Path

RD: 10.0.4.1:10 mac-ip 0050.7966.680d

0 i

Ec RD: 10.0.5.1:10 mac-ip 0050.7966.680e
10.0.5.1 - 100 0 65001 65005 i

Ec RD: 10.0.6.1:10 mac-ip 0050.7966.680f
10.0.6.1 - 100 0 65001 65006 i

```

### 5.5. Проверка связности между хостами (L2-трафик)

**С Host-1 (Leaf-01) на Host-2 (Leaf-02):**
Host-1# ping 172.16.10.12
!!!!!
Success rate is 100 percent (5/5)

```

**С Host-1 (Leaf-01) на Host-3 (Leaf-03):**
Host-1# ping 172.16.10.13
!!!!!
Success rate is 100 percent (5/5)

```

**С Host-2 (Leaf-02) на Host-3 (Leaf-03):**
Host-2# ping 172.16.10.13
!!!!!
Success rate is 100 percent (5/5)
Результаты ping-тестов подтверждают, что L2-трафик между клиентами в разных Leaf успешно проходит через VXLAN-туннели.

---

## 6. Заключение

В ходе работы настроена Overlay-сеть на основе VXLAN EVPN для L2-связанности клиентов:

- Использована существующая Underlay-сеть с eBGP, BFD, MD5 и ECMP.
- На Spine настроена передача EVPN-маршрутов между Leaf.
- На Leaf созданы VTEP, L2 VNI (10100) и VLAN 10.
- EVPN-сервис для VNI 10100 определён через `l2vpn evpn instance` с автоматическими RD и RT.
- Команда `redistribute learned` обеспечивает анонсирование MAC-адресов через EVPN.
- Клиентские порты (Ethernet4) переведены в access-режим.
- Проверена связность между хостами, подключёнными к разным Leaf, через VXLAN-туннели.
- Все BGP EVPN-сессии установлены, MAC-адреса изучаются через контрольную плоскость, L2-трафик между клиентами проходит без потерь.

