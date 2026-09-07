# Лабораторная работа №4. Underlay. BGP (Super-Spine + 3 Spine + 3 Leaf)

**Цель работы:** Настроить протокол BGP в Underlay-сети для обеспечения автоматической IP-связности между всеми сетевыми устройствами фабрики CLOS с использованием eBGP, BFD и MD5-аутентификации.

---

## 1. Топология сети

![Топология](./BGP.png)

В среде **PNET Lab** собрана следующая архитектура:

- **Super-Spine (уровень 1):** 1 коммутатор **Cisco Nexus 5000** (образ NX-OS) — центральный маршрутизатор.
- **Spine (уровень 2):** 3 коммутатора **Arista vEOS** (Spine-01, Spine-02, Spine-03).
- **Leaf (уровень 3):** 3 коммутатора **Arista vEOS** (Leaf-01, Leaf-02, Leaf-03) — каждый подключён к каждому Spine (полносвязная топология).

### Схема подключений

| От (устройство) | К (устройство) | Интерфейс (от) | Интерфейс (к) |
|:---|:---|:---|:---|
| Super-Spine (Nexus 5000) | Spine-01 | E2/1 | Eth1 |
| Super-Spine (Nexus 5000) | Spine-02 | E2/2 | Eth1 |
| Super-Spine (Nexus 5000) | Spine-03 | E2/3 | Eth1 |
| Spine-01 | Leaf-01 | Eth2 | Eth1 |
| Spine-01 | Leaf-02 | Eth3 | Eth1 |
| Spine-01 | Leaf-03 | Eth4 | Eth1 |
| Spine-02 | Leaf-01 | Eth2 | Eth2 |
| Spine-02 | Leaf-02 | Eth3 | Eth2 |
| Spine-02 | Leaf-03 | Eth4 | Eth2 |
| Spine-03 | Leaf-01 | Eth2 | Eth3 |
| Spine-03 | Leaf-02 | Eth3 | Eth3 |
| Spine-03 | Leaf-03 | Eth4 | Eth3 |

---

## 2. План работ

1. **Анализ топологии и планирование адресации.** Назначить Loopback и P2P-адреса для всех линков.
2. **Настройка базовых параметров L3.** Включить IP-адреса на интерфейсах, отключить L2-режим.
3. **Настройка BGP:**
   - Включить BGP на всех устройствах (`feature bgp` на Nexus 5000, `router bgp` на Arista).
   - Настроить eBGP-пиринг между Super-Spine и каждым Spine, а также между каждым Spine и каждым Leaf.
   - Использовать отдельные AS для каждого устройства (65000 – Super-Spine, 65001–65003 – Spine, 65004–65006 – Leaf).
4. **Настройка дополнительных механизмов:**
   - **BFD** с таймерами 50 мс для быстрого обнаружения отказов.
   - **MD5-аутентификация** для защиты BGP-сессий.
5. **Верификация.** Проверить установку BGP-соседств, BFD-сессий, маршруты в таблицах и связность между всеми Loopback-адресами.
6. **Документирование.** Зафиксировать все конфигурации и результаты проверки.
7. **Настройка ECMP:**
   - Включены `maximum-paths` на всех устройствах для использования нескольких равнозначных путей.

---

## 3. Адресное пространство Underlay

| Устройство | Роль | AS | Интерфейс | IP-адрес / Маска | Назначение |
|:---|:---|:---|:---|:---|:---|
| **Nexus 5000** | Super-Spine | 65000 | Loopback0 | 10.0.0.1/32 | Router ID |
| | | | E2/1 | 10.1.1.0/31 | к Spine-01 |
| | | | E2/2 | 10.1.1.2/31 | к Spine-02 |
| | | | E2/3 | 10.1.1.4/31 | к Spine-03 |
| **Spine-01** | Spine | 65001 | Loopback0 | 10.0.1.1/32 | Router ID |
| | | | Eth1 | 10.1.1.1/31 | к Super-Spine |
| | | | Eth2 | 10.1.2.0/31 | к Leaf-01 |
| | | | Eth3 | 10.1.2.2/31 | к Leaf-02 |
| | | | Eth4 | 10.1.2.4/31 | к Leaf-03 |
| **Spine-02** | Spine | 65002 | Loopback0 | 10.0.2.1/32 | Router ID |
| | | | Eth1 | 10.1.1.3/31 | к Super-Spine |
| | | | Eth2 | 10.1.2.6/31 | к Leaf-01 |
| | | | Eth3 | 10.1.2.8/31 | к Leaf-02 |
| | | | Eth4 | 10.1.2.10/31 | к Leaf-03 |
| **Spine-03** | Spine | 65003 | Loopback0 | 10.0.3.1/32 | Router ID |
| | | | Eth1 | 10.1.1.5/31 | к Super-Spine |
| | | | Eth2 | 10.1.2.12/31 | к Leaf-01 |
| | | | Eth3 | 10.1.2.14/31 | к Leaf-02 |
| | | | Eth4 | 10.1.2.16/31 | к Leaf-03 |
| **Leaf-01** | Leaf | 65004 | Loopback0 | 10.0.4.1/32 | Router ID |
| | | | Eth1 | 10.1.2.1/31 | к Spine-01 |
| | | | Eth2 | 10.1.2.7/31 | к Spine-02 |
| | | | Eth3 | 10.1.2.13/31 | к Spine-03 |
| **Leaf-02** | Leaf | 65005 | Loopback0 | 10.0.5.1/32 | Router ID |
| | | | Eth1 | 10.1.2.3/31 | к Spine-01 |
| | | | Eth2 | 10.1.2.9/31 | к Spine-02 |
| | | | Eth3 | 10.1.2.15/31 | к Spine-03 |
| **Leaf-03** | Leaf | 65006 | Loopback0 | 10.0.6.1/32 | Router ID |
| | | | Eth1 | 10.1.2.5/31 | к Spine-01 |
| | | | Eth2 | 10.1.2.11/31 | к Spine-02 |
| | | | Eth3 | 10.1.2.17/31 | к Spine-03 |

---

## 4. Конфигурации устройств

**Важно:** Перед настройкой BGP на Cisco Nexus 5000 активируйте льготный период лицензирования:
```
switch# configure terminal
switch(config)# license grace-period
```

### 4.1. Super-Spine (Cisco Nexus 5000)
```
hostname NEXUS-5000
!
feature bfd
feature bgp
!
interface Ethernet2/1
no switchport
ip address 10.1.1.0/31
bfd interval 50 min_rx 50 multiplier 3
no shutdown
!
interface Ethernet2/2
no switchport
ip address 10.1.1.2/31
bfd interval 50 min_rx 50 multiplier 3
no shutdown
!
interface Ethernet2/3
no switchport
ip address 10.1.1.4/31
bfd interval 50 min_rx 50 multiplier 3
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
neighbor 10.1.1.1 remote-as 65001
bfd
password 0 MySecretKey123
address-family ipv4 unicast
disable-peer-as-check
neighbor 10.1.1.3 remote-as 65002
bfd
password 0 MySecretKey123
address-family ipv4 unicast
disable-peer-as-check
neighbor 10.1.1.5 remote-as 65003
bfd
password 0 MySecretKey123
address-family ipv4 unicast
disable-peer-as-check
```

### 4.2. Spine-01 (Arista vEOS, AS 65001)
```
hostname Spine-01
!
interface Ethernet1
no switchport
ip address 10.1.1.1/31
bfd interval 50 min_rx 50 multiplier 3
no shutdown
!
interface Ethernet2
no switchport
ip address 10.1.2.0/31
bfd interval 50 min_rx 50 multiplier 3
no shutdown
!
interface Ethernet3
no switchport
ip address 10.1.2.2/31
bfd interval 50 min_rx 50 multiplier 3
no shutdown
!
interface Ethernet4
no switchport
ip address 10.1.2.4/31
bfd interval 50 min_rx 50 multiplier 3
no shutdown
!
interface Loopback0
ip address 10.0.1.1/32
!
router bgp 65001
router-id 10.0.1.1
maximum-paths 3 ecmp 3
address-family ipv4
maximum-paths 3
redistribute connected
neighbor 10.1.1.0 remote-as 65000
bfd
password MySecretKey123
address-family ipv4
neighbor 10.1.2.1 remote-as 65004
bfd
password MySecretKey123
address-family ipv4
neighbor 10.1.2.3 remote-as 65005
bfd
password MySecretKey123
address-family ipv4
neighbor 10.1.2.5 remote-as 65006
bfd
password MySecretKey123
address-family ipv4
```
### 4.3. Spine-02 (Arista vEOS, AS 65002)
```
hostname Spine-02
!
interface Ethernet1
no switchport
ip address 10.1.1.3/31
bfd interval 50 min_rx 50 multiplier 3
no shutdown
!
interface Ethernet2
no switchport
ip address 10.1.2.6/31
bfd interval 50 min_rx 50 multiplier 3
no shutdown
!
interface Ethernet3
no switchport
ip address 10.1.2.8/31
bfd interval 50 min_rx 50 multiplier 3
no shutdown
!
interface Ethernet4
no switchport
ip address 10.1.2.10/31
bfd interval 50 min_rx 50 multiplier 3
no shutdown
!
interface Loopback0
ip address 10.0.2.1/32
!
router bgp 65002
router-id 10.0.2.1
maximum-paths 3 ecmp 3
address-family ipv4
maximum-paths 3
redistribute connected
neighbor 10.1.1.2 remote-as 65000
bfd
password MySecretKey123
address-family ipv4
neighbor 10.1.2.7 remote-as 65004
bfd
password MySecretKey123
address-family ipv4
neighbor 10.1.2.9 remote-as 65005
bfd
password MySecretKey123
address-family ipv4
neighbor 10.1.2.11 remote-as 65006
bfd
password MySecretKey123
address-family ipv4
```
### 4.4. Spine-03 (Arista vEOS, AS 65003)
```
hostname Spine-03
!
interface Ethernet1
no switchport
ip address 10.1.1.5/31
bfd interval 50 min_rx 50 multiplier 3
no shutdown
!
interface Ethernet2
no switchport
ip address 10.1.2.12/31
bfd interval 50 min_rx 50 multiplier 3
no shutdown
!
interface Ethernet3
no switchport
ip address 10.1.2.14/31
bfd interval 50 min_rx 50 multiplier 3
no shutdown
!
interface Ethernet4
no switchport
ip address 10.1.2.16/31
bfd interval 50 min_rx 50 multiplier 3
no shutdown
!
interface Loopback0
ip address 10.0.3.1/32
!
router bgp 65003
router-id 10.0.3.1
maximum-paths 3 ecmp 3
address-family ipv4
maximum-paths 3
redistribute connected
neighbor 10.1.1.4 remote-as 65000
bfd
password MySecretKey123
address-family ipv4
neighbor 10.1.2.13 remote-as 65004
bfd
password MySecretKey123
address-family ipv4
neighbor 10.1.2.15 remote-as 65005
bfd
password MySecretKey123
address-family ipv4
neighbor 10.1.2.17 remote-as 65006
bfd
password MySecretKey123
address-family ipv4
```

### 4.5. Leaf-01 (Arista vEOS, AS 65004)
```
hostname Leaf-01
!
interface Ethernet1
no switchport
ip address 10.1.2.1/31
bfd interval 50 min_rx 50 multiplier 3
no shutdown
!
interface Ethernet2
no switchport
ip address 10.1.2.7/31
bfd interval 50 min_rx 50 multiplier 3
no shutdown
!
interface Ethernet3
no switchport
ip address 10.1.2.13/31
bfd interval 50 min_rx 50 multiplier 3
no shutdown
!
interface Loopback0
ip address 10.0.4.1/32
!
router bgp 65004
router-id 10.0.4.1
maximum-paths 3
address-family ipv4
redistribute connected
neighbor 10.1.2.0 remote-as 65001
bfd
password MySecretKey123
address-family ipv4
neighbor 10.1.2.6 remote-as 65002
bfd
password MySecretKey123
address-family ipv4
neighbor 10.1.2.12 remote-as 65003
bfd
password MySecretKey123
address-family ipv4
```

### 4.6. Leaf-02 (Arista vEOS, AS 65005)
```
hostname Leaf-02
!
interface Ethernet1
no switchport
ip address 10.1.2.3/31
bfd interval 50 min_rx 50 multiplier 3
no shutdown
!
interface Ethernet2
no switchport
ip address 10.1.2.9/31
bfd interval 50 min_rx 50 multiplier 3
no shutdown
!
interface Ethernet3
no switchport
ip address 10.1.2.15/31
bfd interval 50 min_rx 50 multiplier 3
no shutdown
!
interface Loopback0
ip address 10.0.5.1/32
!
router bgp 65005
router-id 10.0.5.1
maximum-paths 3
address-family ipv4
redistribute connected
neighbor 10.1.2.2 remote-as 65001
bfd
password MySecretKey123
address-family ipv4
neighbor 10.1.2.8 remote-as 65002
bfd
password MySecretKey123
address-family ipv4
neighbor 10.1.2.14 remote-as 65003
bfd
password MySecretKey123
address-family ipv4
```

### 4.7. Leaf-03 (Arista vEOS, AS 65006)
```
hostname Leaf-03
!
interface Ethernet1
no switchport
ip address 10.1.2.5/31
bfd interval 50 min_rx 50 multiplier 3
no shutdown
!
interface Ethernet2
no switchport
ip address 10.1.2.11/31
bfd interval 50 min_rx 50 multiplier 3
no shutdown
!
interface Ethernet3
no switchport
ip address 10.1.2.17/31
bfd interval 50 min_rx 50 multiplier 3
no shutdown
!
interface Loopback0
ip address 10.0.6.1/32
!
router bgp 65006
router-id 10.0.6.1
maximum-paths 3
address-family ipv4
redistribute connected
neighbor 10.1.2.4 remote-as 65001
bfd
password MySecretKey123
address-family ipv4
neighbor 10.1.2.10 remote-as 65002
bfd
password MySecretKey123
address-family ipv4
neighbor 10.1.2.16 remote-as 65003
bfd
password MySecretKey123
address-family ipv4
```

**Примечания по конфигурации:**
- Пароль MD5 `MySecretKey123` используется на всех BGP-сессиях – он должен быть одинаковым на обоих концах каждого пиринга.
- На Nexus 5000 команда `disable-peer-as-check` позволяет Super-Spine передавать маршруты между разными Spine-коммутаторами (иначе eBGP не будет анонсировать маршруты с AS, отличным от своей). Это необходимо для связности между Leaf через Super-Spine.
- BFD с таймерами 50 мс и множителем 3 даёт таймаут 150 мс.

---

## 5. Верификация

### 5.1. Проверка BGP-соседств

**На Super-Spine (Nexus 5000)**
Команда:
show ip bgp summary

```
Вывод:
BGP summary information for VRF default, address family IPv4 Unicast
BGP router identifier 10.0.0.1, local AS number 65000
Neighbor V AS MsgRcvd MsgSent TblVer InQ OutQ Up/Down State/PfxRcd
10.1.1.1 4 65001 25 25 25 0 0 00:12:30 6
10.1.1.3 4 65002 24 24 25 0 0 00:12:28 6
10.1.1.5 4 65003 26 26 25 0 0 00:12:35 6

```
**Пояснение полей:**

| Параметр | Значение | Описание |
|:---|:---|:---|
| **Neighbor** | 10.1.1.1, 10.1.1.3, 10.1.1.5 | IP-адрес соседа (Spine-01, Spine-02, Spine-03) |
| **V** | 4 | Версия BGP (4) |
| **AS** | 65001, 65002, 65003 | Номер AS соседа |
| **MsgRcvd / MsgSent** | ~25 | Количество полученных и отправленных BGP-сообщений |
| **Up/Down** | 00:12:30 | Время установленного соседства |
| **State/PfxRcd** | 6 | Количество префиксов, полученных от соседа (каждый Spine анонсирует свои Loopback, линки и Loopback трёх Leaf) |

**На Leaf-01 (Arista)**
Команда:
show ip bgp summary

```
Вывод:
BGP summary information for VRF default, address family IPv4 Unicast
BGP router identifier 10.0.4.1, local AS number 65004
Neighbor V AS MsgRcvd MsgSent TblVer InQ OutQ Up/Down State/PfxRcd
10.1.2.0 4 65001 20 20 20 0 0 00:10:45 8
10.1.2.6 4 65002 19 19 20 0 0 00:10:40 8
10.1.2.12 4 65003 21 21 20 0 0 00:10:55 8
```
**Пояснение:**

| Параметр | Значение | Описание |
|:---|:---|:---|
| **Neighbor** | 10.1.2.0, 10.1.2.6, 10.1.2.12 | IP-адреса Spine-01, Spine-02, Spine-03 соответственно |
| **AS** | 65001, 65002, 65003 | AS каждого Spine |
| **State/PfxRcd** | 8 | Количество полученных префиксов (включая маршруты до Super-Spine, других Spine и других Leaf) |

### 5.2. Проверка BFD-сессий

**На Super-Spine (Nexus 5000)**
Команда:
show bfd neighbors
```
Вывод:
OurAddr NeighAddr LD/RD RH/RS Holdown(mult) State Int
10.1.1.0 10.1.1.1 1090519041/0 Up 0(3) Up Eth2/1
10.1.1.2 10.1.1.3 1090519042/0 Up 0(3) Up Eth2/2
10.1.1.4 10.1.1.5 1090519043/0 Up 0(3) Up Eth2/3
```
**Пояснение:**

| Параметр | Значение | Описание |
|:---|:---|:---|
| **OurAddr** | 10.1.1.0, 10.1.1.2, 10.1.1.4 | Локальный IP-адрес интерфейса |
| **NeighAddr** | 10.1.1.1, 10.1.1.3, 10.1.1.5 | IP-адрес соседа (Spine) |
| **State** | Up | Состояние BFD-сессии (должно быть Up) |
| **Int** | Eth2/1, Eth2/2, Eth2/3 | Интерфейс, на котором установлена сессия |

**На Spine-01**
Команда:
show bfd neighbors

```
Вывод:
OurAddr NeighAddr State Int
10.1.1.1 10.1.1.0 Up Eth1
10.1.2.0 10.1.2.1 Up Eth2
10.1.2.2 10.1.2.3 Up Eth3
10.1.2.4 10.1.2.5 Up Eth4
```
Все сессии должны быть в состоянии `Up`.

### 5.3. Проверка таблицы маршрутизации на Leaf-01

Команда:
Leaf-01# show ip route bgp

```
Вывод:
VRF: default
Codes: C - connected, S - static, K - kernel,
O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
N2 - OSPF NSSA external type2, B - Other BGP Routes,
B I - iBGP, B E - eBGP, R - RIP, I L1 - IS-IS level 1,
I L2 - IS-IS level 2, O3 - OSPFv3, A B - BGP Aggregate,
A O - OSPF Summary, NG - Nexthop Group Static Route,
V - VXLAN Control Service, M - Martian,
DH - DHCP client installed default route,
DP - Dynamic Policy Route, L - VRF Leaked,
G - gRIBI, RC - Route Cache Route

B E 10.0.0.1/32 [200/0] via 10.1.2.0, Ethernet1
via 10.1.2.6, Ethernet2
via 10.1.2.12, Ethernet3
B E 10.0.1.1/32 [200/0] via 10.1.2.0, Ethernet1
B E 10.0.2.1/32 [200/0] via 10.1.2.6, Ethernet2
B E 10.0.3.1/32 [200/0] via 10.1.2.12, Ethernet3
B E 10.0.5.1/32 [200/0] via 10.1.2.0, Ethernet1
via 10.1.2.6, Ethernet2
via 10.1.2.12, Ethernet3
B E 10.0.6.1/32 [200/0] via 10.1.2.0, Ethernet1
via 10.1.2.6, Ethernet2
via 10.1.2.12, Ethernet3
B E 10.1.1.0/31 [200/0] via 10.1.2.0, Ethernet1
B E 10.1.1.2/31 [200/0] via 10.1.2.6, Ethernet2
B E 10.1.1.4/31 [200/0] via 10.1.2.12, Ethernet3
B E 10.1.2.2/31 [200/0] via 10.1.2.0, Ethernet1
B E 10.1.2.4/31 [200/0] via 10.1.2.0, Ethernet1
B E 10.1.2.8/31 [200/0] via 10.1.2.6, Ethernet2
B E 10.1.2.10/31 [200/0] via 10.1.2.6, Ethernet2
B E 10.1.2.14/31 [200/0] via 10.1.2.12, Ethernet3
B E 10.1.2.16/31 [200/0] via 10.1.2.12, Ethernet3
```
**Пояснение параметров и демонстрация ECMP:**

| Маршрут назначения | Описание узла | Next Hop (Куда шлем трафик) | Интерфейсы | Примечание по ECMP |
|:---|:---|:---|:---|:---|
| `10.0.0.1/32` | Loopback0 Super-Spine (Nexus) | `10.1.2.0`<br>`10.1.2.6`<br>`10.1.2.12` | Ethernet1<br>Ethernet2<br>Ethernet3 | **ECMP активен (3 пути).** Трафик до ядра балансируется через все три Spine одновременно. |
| `10.0.1.1/32` | Loopback0 Spine-01 | `10.1.2.0` | Ethernet1 | Прямой маршрут до Spine-01. |
| `10.0.2.1/32` | Loopback0 Spine-02 | `10.1.2.6` | Ethernet2 | Прямой маршрут до Spine-02. |
| `10.0.3.1/32` | Loopback0 Spine-03 | `10.1.2.12` | Ethernet3 | Прямой маршрут до Spine-03. |
| `10.0.5.1/32` | Loopback0 Leaf-02 | `10.1.2.0`<br>`10.1.2.6`<br>`10.1.2.12` | Ethernet1<br>Ethernet2<br>Ethernet3 | **ECMP активен (3 пути).** Пакеты до соседа Leaf-02 распределяются по всей Spine-фабрике. |
| `10.0.6.1/32` | Loopback0 Leaf-03 | `10.1.2.0`<br>`10.1.2.6`<br>`10.1.2.12` | Ethernet1<br>Ethernet2<br>Ethernet3 | **ECMP активен (3 пути).** Пакеты до соседа Leaf-03 распределяются по всей Spine-фабрике. |

**Вывод:** Тест показывает, что коммутатор `Leaf-01` успешно построил полную карту сети Underlay. Наличие ровно трех путей `via` к целевым Loopback-адресам других Leaf и Super-Spine наглядно доказывает корректность работы механизма ECMP (`maximum-paths 3`) в CLOS-фабрике на операционной системе Arista EOS.

### 5.4. Проверка связности между Loopback-адресами

**С Leaf-01 на Super-Spine**
Команда:
ping 10.0.0.1 source 10.0.4.1

```
Результат:
!!!!!
Success rate is 100 percent (5/5)

```

**С Leaf-01 на Leaf-02**
Команда:
ping 10.0.5.1 source 10.0.4.1

```
Результат:
!!!!!
Success rate is 100 percent (5/5)
```

**С Leaf-01 на Leaf-03**
Команда:
ping 10.0.6.1 source 10.0.4.1

```
Результат:
!!!!!
Success rate is 100 percent (5/5)

```
**С Spine-01 на Spine-03 (через Super-Spine)**
Команда:
ping 10.0.3.1 source 10.0.1.1

```
Результат:
!!!!!
Success rate is 100 percent (5/5)

```

---

## 6. Заключение

В ходе лабораторной работы была успешно развернута и протестирована Underlay-инфраструктура многосвязной фабрики CLOS (Super-Spine + 3 Spine + 3 Leaf):

1. **Маршрутизация на базе eBGP:** Настроена сквозная динамическая маршрутизация с использованием уникальных автономных систем (AS) для каждого сетевого узла. На Cisco Nexus активирована команда `disable-peer-as-check`, что обеспечило корректный обмен префиксами между Spine-коммутаторами на L3-уровне.
2. **Отказоустойчивость (BFD):** Интеграция протокола BFD с агрессивными таймерами (50 мс) позволила снизить время обнаружения физических отказов линков со стандартных минут BGP до минимальных 150 миллисекунд, обеспечив мгновенную сходимость сети.
3. **Безопасность:** Все eBGP-сессии защищены с помощью MD5-аутентификации, что предотвращает возможность несанкционированного подключения к фабрике или подмены маршрутной информации.
4. **Балансировка трафика (ECMP):** Настройка параметра `maximum-paths 3` на всех уровнях фабрики позволила успешно задействовать аппаратную балансировку трафика. Проверка таблицы маршрутизации на Leaf-коммутаторах подтвердила наличие трех равнозначных путей до Loopback-адресов остальных узлов, что гарантирует эффективное распределение сетевой нагрузки и высокую пропускную способность.
