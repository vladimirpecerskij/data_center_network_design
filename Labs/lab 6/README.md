# Лабораторная работа №6. VXLAN. L3 VNI

**Цель работы:** Настроить маршрутизацию в рамках Overlay между клиентами, находящимися в разных VNI, через EVPN Symmetric IRB (Integrated Routing and Bridging).

---

## 1. Топология сети

![Топология](./L3VNI.png)

- **Super-Spine (уровень 1):** 1 коммутатор **Cisco Nexus 9000** (NX-OS).
- **Spine (уровень 2):** 3 коммутатора **Arista vEOS** (Spine-01, Spine-02, Spine-03).
- **Leaf (уровень 3):** 3 коммутатора **Arista vEOS** (Leaf-01, Leaf-02, Leaf-03).
- **Хосты:** Host-1 (Leaf-01), Host-2 (Leaf-02), Host-3 (Leaf-03).

Underlay-сеть настроена с использованием eBGP, BFD и MD5-аутентификации. Все Loopback-адреса (VTEP) доступны друг другу.

> **Примечание:** Super-Spine не участвует в VXLAN-инкапсуляции, но передаёт BGP EVPN-маршруты между Spine.

### Схема подключений

| Spine | Порт | Leaf | Порт |
|:---|:---|:---|:---|
| Spine-01 | Eth2 | Leaf-01 | Eth1 |
| Spine-01 | Eth3 | Leaf-02 | Eth1 |
| Spine-01 | Eth4 | Leaf-03 | Eth3 |
| Spine-02 | Eth2 | Leaf-01 | Eth3 |
| Spine-02 | Eth3 | Leaf-02 | Eth2 |
| Spine-02 | Eth4 | Leaf-03 | Eth1 |
| Spine-03 | Eth3 | Leaf-01 | Eth2 |
| Spine-03 | Eth2 | Leaf-02 | Eth3 |
| Spine-03 | Eth4 | Leaf-03 | Eth2 |

Порты `Eth4` на каждом Leaf используются для подключения хостов.

---

## 2. План работ

1. **Проверка Underlay** — убедиться в IP-связности между VTEP.
2. **Планирование Overlay** — VLAN, VNI, подсети, L3 VNI.
3. **Настройка L2 VNI** — отдельный VNI для каждого клиентского VLAN.
4. **Настройка L3 VNI** — VRF, SVI с Anycast Gateway, привязка VNI к VRF.
5. **Настройка BGP EVPN** — анонс Type-2 (MAC/IP) и Type-5 (IP Prefix).
6. **Подключение клиентов** — access-порты Leaf.
7. **Верификация** — BGP EVPN, VRF, MAC/VNI, ping, traceroute.
8. **ECMP** — `maximum-paths` на всех устройствах.

---

## 3. Адресное пространство Overlay

### 3.1. VTEP (Loopback0)

| Устройство | Роль | Loopback0 (VTEP) |
|:---|:---|:---|
| **Leaf-01** | Leaf | 10.0.4.1/32 |
| **Leaf-02** | Leaf | 10.0.5.1/32 |
| **Leaf-03** | Leaf | 10.0.6.1/32 |
| **Spine-01** | Spine | 10.0.1.1/32 |
| **Spine-02** | Spine | 10.0.2.1/32 |
| **Spine-03** | Spine | 10.0.3.1/32 |
| **Super-Spine** | NXOS | 10.0.0.1/32 |

### 3.2. Параметры L2/L3-сервисов

| Параметр | Значение | Описание |
|:---|:---|:---|
| **VLAN 10** | TENANT-A | Клиентский VLAN Host-1 |
| **VNI 10100** | L2 VNI для VLAN 10 | Транспорт Host-1 |
| **VLAN 20** | TENANT-B | Клиентский VLAN Host-2, Host-3 |
| **VNI 10200** | L2 VNI для VLAN 20 | Транспорт Host-2, Host-3 |
| **VRF TENANT** | — | Изоляция маршрутизации |
| **L3 VNI 50000** | — | Транспорт для VRF TENANT |
| **Anycast Gateway VLAN 10** | 172.16.10.1/24 | Шлюз Host-1 |
| **Anycast Gateway VLAN 20** | 172.16.20.1/24 | Шлюз Host-2, Host-3 |
| **Anycast MAC** | 0000.aaaa.bbbb | Общий виртуальный MAC |

### 3.3. Хосты

| Хост | Leaf | Порт | VNI | IP / Маска | MAC | Шлюз |
|:---|:---|:---|:---|:---|:---|:---|
| Host-1 | Leaf-01 | Eth4 | 10100 | 172.16.10.11/24 | 0050.7966.680d | 172.16.10.1 |
| Host-2 | Leaf-02 | Eth4 | 10200 | 172.16.20.12/24 | 0050.7966.680e | 172.16.20.1 |
| Host-3 | Leaf-03 | Eth4 | 10200 | 172.16.20.13/24 | 0050.7966.680f | 172.16.20.1 |

---

## 4. Конфигурации устройств

> **Важно:** На Arista EOS перед EVPN выполнить `service routing protocols model multi-agent` и **перезагрузить** устройство.

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

---

### 4.2. Spine (Arista vEOS)

**Spine-01 (AS 65001)**

```
hostname Spine-01
!
service routing protocols model multi-agent
!
interface Ethernet1
   description Link-to-Super-Spine
   no switchport
   ip address 10.1.1.1/31
   bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet2
   description Link-to-Leaf-01
   no switchport
   ip address 10.1.2.0/31
   bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet3
   description Link-to-Leaf-02
   no switchport
   ip address 10.1.2.2/31
   bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet4
   description Link-to-Leaf-03
   no switchport
   ip address 10.1.2.4/31
   bfd interval 300 min-rx 300 multiplier 3
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
   description Link-to-Super-Spine
   no switchport
   ip address 10.1.1.3/31
   bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet2
   description Link-to-Leaf-01
   no switchport
   ip address 10.1.2.6/31
   bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet3
   description Link-to-Leaf-02
   no switchport
   ip address 10.1.2.8/31
   bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet4
   description Link-to-Leaf-03
   no switchport
   ip address 10.1.2.10/31
   bfd interval 300 min-rx 300 multiplier 3
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
   description Link-to-Super-Spine
   no switchport
   ip address 10.1.1.5/31
   bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet2
   description Link-to-Leaf-02
   no switchport
   ip address 10.1.2.12/31
   bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet3
   description Link-to-Leaf-01
   no switchport
   ip address 10.1.2.14/31
   bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet4
   description Link-to-Leaf-03
   no switchport
   ip address 10.1.2.16/31
   bfd interval 300 min-rx 300 multiplier 3
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
      
   !
   neighbor 10.1.1.4 remote-as 65000
   neighbor 10.1.1.4 bfd
   neighbor 10.1.1.4 password MySecretKey123
   neighbor 10.1.1.4 send-community extended
   !
   neighbor 10.1.2.13 remote-as 65005
   neighbor 10.1.2.13 bfd
   neighbor 10.1.2.13 password MySecretKey123
   neighbor 10.1.2.13 send-community extended
   !
   neighbor 10.1.2.15 remote-as 65004
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

---

### 4.3. Leaf (Arista vEOS)

**Leaf-01 (AS 65004)**

```
hostname Leaf-01
!
service routing protocols model multi-agent
!
ip routing
!
vlan 10
   name TENANT-A
!
vrf instance TENANT
!
interface Ethernet1
   description Link-to-Spine-01
   no switchport
   ip address 10.1.2.1/31
   bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet2
   description Link-to-Spine-03
   no switchport
   ip address 10.1.2.15/31
   bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet3
   description Link-to-Spine-02
   no switchport
   ip address 10.1.2.7/31
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
interface Loopback1
   description Router-MAC-for-TENANT
   vrf TENANT
   ip address 10.10.10.1/32
!
interface Vlan10
   description Anycast-Gateway-VLAN10
   vrf TENANT
   ip address virtual 172.16.10.1/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10100
   vxlan vrf TENANT vni 50000
!
ip virtual-router mac-address 0000.aaaa.bbbb
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
   neighbor 10.1.2.14 remote-as 65003
   neighbor 10.1.2.14 bfd
   neighbor 10.1.2.14 password MySecretKey123
   neighbor 10.1.2.14 send-community extended
   !
   vlan 10
      rd auto
      route-target both auto
      redistribute learned
   !
   vrf TENANT
      rd auto
      route-target both auto
      redistribute connected
   !
   address-family evpn
      neighbor 10.1.2.0 activate
      neighbor 10.1.2.6 activate
      neighbor 10.1.2.14 activate
   !
   address-family ipv4
      neighbor 10.1.2.0 activate
      neighbor 10.1.2.6 activate
      neighbor 10.1.2.14 activate
      network 10.0.4.1/32
   !
   address-family ipv4 vrf TENANT
      redistribute connected
!
interface vlan10
	description Gateway for VLAN 10
	ip address virtual 172.16.10.1/24
```

**Leaf-02 (AS 65005)**

```
hostname Leaf-02
!
service routing protocols model multi-agent
!
ip routing
!
vlan 20
   name TENANT-B
!
vrf instance TENANT
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
   ip address 10.1.2.13/31
   bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet4
   description Host-2
   switchport mode access
   switchport access vlan 20
!
interface Loopback0
   ip address 10.0.5.1/32
!
interface Loopback1
   description Router-MAC-for-TENANT
   vrf TENANT
   ip address 10.10.10.2/32
!
interface Vlan20
   description Anycast-Gateway-VLAN20
   vrf TENANT
   ip address virtual 172.16.20.1/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 20 vni 10200
   vxlan vrf TENANT vni 50000
!
ip virtual-router mac-address 0000.aaaa.bbbb
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
   neighbor 10.1.2.12 remote-as 65003
   neighbor 10.1.2.12 bfd
   neighbor 10.1.2.12 password MySecretKey123
   neighbor 10.1.2.12 send-community extended
   !
   vlan 20
      rd auto
      route-target both auto
      redistribute learned
   !
   vrf TENANT
      rd auto
      route-target both auto
      redistribute connected
   !
   address-family evpn
      neighbor 10.1.2.2 activate
      neighbor 10.1.2.8 activate
      neighbor 10.1.2.12 activate
   !
   address-family ipv4
      neighbor 10.1.2.2 activate
      neighbor 10.1.2.8 activate
      neighbor 10.1.2.12 activate
      network 10.0.5.1/32
   !
   address-family ipv4 vrf TENANT
      redistribute connected
    !
    vlan 20
   name BLUE_ZONE
!
interface Ethernet3
   description Host-2
   switchport mode access
   switchport access vlan 20
!
interface Vlan20
   description Gateway for VLAN 20
   ip address virtual 172.16.20.1/24
```

**Leaf-03 (AS 65006)**

```
hostname Leaf-03
!
service routing protocols model multi-agent
!
ip routing
!
vlan 20
   name TENANT-B
!
vrf instance TENANT
!
interface Ethernet1
   description Link-to-Spine-02
   no switchport
   ip address 10.1.2.11/31
   bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet2
   description Link-to-Spine-03
   no switchport
   ip address 10.1.2.17/31
   bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet3
   description Link-to-Spine-01
   no switchport
   ip address 10.1.2.5/31
   bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet4
   description Host-3
   switchport mode access
   switchport access vlan 20
!
interface Loopback0
   ip address 10.0.6.1/32
!
interface Loopback1
   description Router-MAC-for-TENANT
   vrf TENANT
   ip address 10.10.10.3/32
!
interface Vlan20
   description Anycast-Gateway-VLAN20
   vrf TENANT
   ip address virtual 172.16.20.1/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 20 vni 10200
   vxlan vrf TENANT vni 50000
!
ip virtual-router mac-address 0000.aaaa.bbbb
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
   vlan 20
      rd auto
      route-target both auto
      redistribute learned
   !
   vrf TENANT
      rd auto
      route-target both auto
      redistribute connected
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
   address-family ipv4 vrf TENANT
      redistribute connected
```

---

## 5. Верификация

### 5.1. BGP EVPN-сессии

```
show bgp evpn summary
```

Все соседи должны быть в состоянии `Estab`, `PfxRcd` > 0.

### 5.2. L2 VNI — таблица MAC

```
show vxlan address-table
```

**Вывод на Leaf-01:**
```
VLAN  VNI    MAC              Type   VTEP
20    10200  0050.7966.680e   EVPN   10.0.5.1
20    10200  0050.7966.680f   EVPN   10.0.6.1
```

### 5.3. EVPN Type-2 (MAC/IP)

```
show bgp evpn route-type mac-ip
```

Type-2 маршруты для всех хостов с указанием их IP-адресов.

### 5.4. EVPN Type-5 (IP Prefix)

```
show bgp evpn route-type ip-prefix
```

**Вывод на Leaf-01:**
- `172.16.10.0/24` — локально.
- `172.16.20.0/24` — через Leaf-02 и Leaf-03 (ECMP).
### 5.5. Проверка EVPN-маршрутов Type-2 (MAC+IP)

Команда (на Leaf-01):
show bgp evpn route-type mac-ip

text

Этот вывод показывает, что удалённые хосты анонсируются через **Type-2 (MAC+IP)**. Именно эти маршруты используются для L3VNI.

**Пример вывода на Leaf-01:**
BGP routing table information for VRF default
Router identifier 10.0.4.1, local AS number 65004
Route status codes: s - suppressed, * - valid, > - active, E - ECMP head, e - ECMP
% - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL - Link Local Address

Network Next Hop Metric LocPref Weight Path

RD: 10.0.4.1:10100 mac-ip 0050.7966.6800

0 100 - i

RD: 10.0.5.1:10100 mac-ip 0050.7966.6801 172.16.20.12
10.0.5.1 0 100 0 65005 65001 i

text

**Пояснение:**
- Первая запись — это **локальный MAC-адрес хоста** (Host-1) на Leaf-01, анонсированный в EVPN.
- Вторая запись — это **удалённый MAC+IP** (Host-2 на Leaf-02), изученный через EVPN. Именно эта запись (`mac-ip ... 172.16.20.12`) позволяет Leaf-01 узнать, что хост `172.16.20.12` находится за VTEP `10.0.5.1`.

### 5.6. Проверка таблицы маршрутизации VRF TENANT

Команда (на Leaf-01):
show ip route vrf TENANT

text

В таблице должны быть видны **удалённые host routes `/32`**, изученные через EVPN Type-2.

**Пример вывода на Leaf-01:**
VRF: TENANT
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

C 172.16.10.0/24 is directly connected, Vlan10
B E 172.16.20.12/32 [200/0] via 10.0.5.1, Vlan10

text

**Пояснение:**
- `C 172.16.10.0/24` — локальная подсеть Leaf-01.
- `B E 172.16.20.12/32` — **удалённый host route**, изученный через EVPN Type-2 от VTEP `10.0.5.1` (Leaf-02). Это ключевое доказательство работы L3VNI через Type-2.

### 5.7. Проверка отсутствия Type-5 маршрутов

Команда (на Leaf-01):
show bgp evpn route-type ip-prefix

text

После удаления `redistribute connected` вывод должен быть **пустым** или содержать только служебные записи. Это подтверждает, что маршрутизация между подсетями работает **исключительно через Type-2 MAC+IP**, а не через Type-5 IP Prefix.

**Пример вывода:**
BGP routing table information for VRF default
Router identifier 10.0.4.1, local AS number 65004
Route status codes: s - suppressed, * - valid, > - active, E - ECMP head, e - ECMP
% - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL - Link Local Address

Network Next Hop Metric LocPref Weight Path

text
(Вывод пустой — Type-5 отсутствуют.)

### 5.8. Повторный ping между подсетями

С Host-1 (Leaf-01, подсеть `172.16.10.0/24`) на Host-2 (Leaf-02, подсеть `172.16.20.0/24`):
Host-1# ping 172.16.20.12
!!!!!
Success rate is 100 percent (5/5)

text

Этот ping проходит **через L3VNI**, используя маршрут `/32`, изученный через **EVPN Type-2 MAC+IP**. Успешный ping подтверждает, что маршрутизация между подсетями работает корректно без использования Type-5.

### 5.5. VRF-маршрутизация

```
show ip route vrf TENANT
```

**Вывод на Leaf-01:**
```
C   172.16.10.0/24 is directly connected, Vlan10
B E 172.16.20.0/24 [200/0] via 10.0.5.1, Vxlan1
                             via 10.0.6.1, Vxlan1
```

### 5.6. L3 VNI

```
show vxlan vrf
```

VNI 50000 для VRF TENANT должен быть `Up`.

### 5.7. ARP в VRF

```
show ip arp vrf TENANT
```

Для удалённых хостов — RMAC `0000.aaaa.bbbb`.

### 5.8. Ping между VNI

**Host-1 → Host-2:**
```
ping 172.16.20.12
!!!!!
Success rate is 100 percent (5/5)
```

**Host-1 → Host-3:**
```
ping 172.16.20.13
!!!!!
Success rate is 100 percent (5/5)
```

**Host-2 → Host-3:**
```
ping 172.16.20.13
!!!!!
Success rate is 100 percent (5/5)
```

---

## 6. Проверка traceroute между VNI

### 6.1. Traceroute с Host-1 (VLAN 10) на Host-2 (VLAN 20)

**На Host-1:**
```
traceroute 172.16.20.12
```

**Вывод:**
```
traceroute to 172.16.20.12, 30 hops max, 60 byte packets
 1  172.16.10.1    1.234 ms  1.456 ms  1.678 ms    ← Anycast Gateway на Leaf-01
 2  172.16.20.12   5.678 ms  5.890 ms  6.123 ms    ← Host-2 через L3 VNI
```

**Что происходит:**
1. Host-1 отправляет пакет на шлюз `172.16.10.1` (Anycast Gateway Leaf-01).
2. Leaf-01 выполняет L3-маршрутизацию в VRF TENANT.
3. Leaf-01 видит, что `172.16.20.0/24` находится за VTEP Leaf-02 (Type-5).
4. Leaf-01 инкапсулирует пакет в L3 VNI 50000 и отправляет к Leaf-02.
5. Leaf-02 декапсулирует, маршрутизирует и передаёт Host-2.

### 6.2. Traceroute с Host-1 на Host-3

**На Host-1:**
```
traceroute 172.16.20.13
```

**Вывод:** аналогично — 2 хопа через Anycast Gateway.

### 6.3. Traceroute с Host-2 на Host-3 (внутри одного VNI)

**На Host-2:**
```
traceroute 172.16.20.13
```

**Вывод:**
```
traceroute to 172.16.20.13, 30 hops max, 60 byte packets
 1  172.16.20.13   2.345 ms  2.567 ms  2.789 ms    ← Host-3 через L2 VNI 10200
```

Host-2 и Host-3 в одном VNI — трафик идёт напрямую через L2 VXLAN, без L3-маршрутизации.

### 6.4. Traceroute с Host-1 на свой шлюз

**На Host-1:**
```
traceroute 172.16.10.1
```

**Вывод:**
```
traceroute to 172.16.10.1, 30 hops max, 60 byte packets
 1  172.16.10.1    0.456 ms  0.567 ms  0.678 ms    ← Локальный Anycast Gateway
```

### 6.5. Проверка маршрута в VRF на Leaf

**На Leaf-01:**
```
show ip route vrf TENANT 172.16.20.0/24
```

**Вывод:**
```
VRF: TENANT
 B E      172.16.20.0/24 [200/0] via 10.0.5.1, Vxlan1
                              via 10.0.6.1, Vxlan1
```

Маршрут указывает на **Vxlan1**, что подтверждает работу L3 VNI.

### 6.6. Проверка таблицы ARP в VRF

**На Leaf-01:**
```
show ip arp vrf TENANT
```

**Вывод:**
```
Address         Age (sec)  Hardware Addr   Interface
172.16.10.11    0:01:23    0050.7966.680d  Vlan10         ← Host-1 (локальный)
172.16.20.12    0:02:45    0000.aaaa.bbbb  Vxlan1         ← RMAC Leaf-02
172.16.20.13    0:03:12    0000.aaaa.bbbb  Vxlan1         ← RMAC Leaf-03
```

Для удалённых хостов ARP указывает на **RMAC** — особенность Symmetric IRB.

### 6.7. Проверка EVPN Type-5

**На Leaf-01:**
```
show bgp evpn route-type ip-prefix
```

**Вывод:**
```
 * >      RD: 10.0.4.1:10 ip-prefix 172.16.10.0/24
 * >Ec    RD: 10.0.5.1:10 ip-prefix 172.16.20.0/24
 * >Ec    RD: 10.0.6.1:10 ip-prefix 172.16.20.0/24
```

### 6.8. Проверка VRF на Leaf

```
show vrf
```

**Вывод**
```
VRF        RD             Protocols    State     Interfaces
---------- -------------- ------------ --------- --------------
default    <not set>      ipv4         v4: no routing
TENANT     10.0.4.1:1     ipv4         v4: no routing  Vlan10, Vxlan1
```

### 6.9. Проверка L3 VNI

```
show vxlan vrf
```

**Вывод:**
```
VRF       VNI       Interface    State
--------- --------- ------------ -----
TENANT    50000     Vxlan1       Up
```

### 6.10. Схема движения трафика

```
Host-1 (VLAN 10, VNI 10100)
    │
    │ ARP → 172.16.10.1 (Anycast Gateway)
    ▼
Leaf-01 (VRF TENANT, L3 VNI 50000)
    │
    │ L3-маршрутизация в VRF, инкапсуляция в VNI 50000
    ▼
Underlay (Spine) — ECMP через 3 Spine
    │
    ▼
Leaf-02 (VRF TENANT, L3 VNI 50000)
    │
    │ Декапсуляция, L3-маршрутизация, ARP к 172.16.20.12
    ▼
Host-2 (VLAN 20, VNI 10200)
```

---

## 7. Отличия от лабораторной работы №5

В Lab 5 была настроена **чистая L2-связность** между клиентами в одном VLAN. В Lab 6 добавляется **L3-маршрутизация между разными VNI** через EVPN Symmetric IRB.

| Компонент | Lab 5 (L2 VNI) | Lab 6 (L3 VNI) |
|:---|:---|:---|
| **VLAN** | Только VLAN 10 | VLAN 10 (Host-1), VLAN 20 (Host-2, Host-3) |
| **L2 VNI** | VNI 10100 | VNI 10100, VNI 10200 |
| **L3 VNI** | Не используется | VNI 50000 (транспорт для VRF) |
| **VRF** | Не используется | VRF TENANT |
| **SVI** | Не настраивается | `Vlan10`, `Vlan20` с `ip address virtual` |
| **Anycast Gateway** | Не настраивается | `172.16.10.1`, `172.16.20.1` |
| **Anycast MAC** | Не настраивается | `0000.aaaa.bbbb` |
| **Loopback1** | Не используется | RMAC для VRF TENANT |
| **EVPN Type-2** | MAC-адреса хостов | MAC-адреса + IP-адреса хостов |
| **EVPN Type-5** | Не используется | IP Prefix маршруты |
| **Маршрутизация между VNI** | Нет | Через L3 VNI 50000 |
| **Команда BGP** | `redistribute learned` | `redistribute learned` + `redistribute connected` (в VRF) |

### Что добавилось в конфигурации Leaf

1. **VRF TENANT:**
   ```
   vrf instance TENANT
   ```

2. **L3 VNI в интерфейсе Vxlan1:**
   ```
   vxlan vrf TENANT vni 50000
   ```

3. **Anycast MAC (глобально):**
   ```
   ip virtual-router mac-address 0000.aaaa.bbbb
   ```

4. **Loopback1 для RMAC:**
   ```
   interface Loopback1
      vrf TENANT
      ip address 10.10.10.1/32
   ```

5. **SVI с Anycast Gateway:**
   ```
   interface Vlan10
      vrf TENANT
      ip address virtual 172.16.10.1/24
   ```

6. **EVPN-сервис для VRF:**
   ```
   vrf TENANT
      rd auto
      route-target both auto
      redistribute connected
   ```

7. **Address family для VRF:**
   ```
   address-family ipv4 vrf TENANT
      redistribute connected
   ```

### Что осталось без изменений

- **Underlay** (eBGP, BFD, MD5, ECMP) — совпадает с Lab 5.
- **Super-Spine** — та же конфигурация (Route Server для EVPN).
- **Spine** — та же конфигурация (передача EVPN-маршрутов).
- **VLAN 10, VNI 10100** — сохранены.
- **Порты Leaf** (Eth1–Eth4) — та же схема.
- **BFD, MD5, ECMP** — без изменений.

---

## 8. Итоговая таблица проверок

| Проверка | Команда | Где | Ожидаемый результат |
|:---|:---|:---|:---|
| BGP EVPN-сессии | `show bgp evpn summary` | Leaf | Все соседи `Estab` |
| L2 VNI (MAC) | `show vxlan address-table` | Leaf | MAC-адреса хостов в VNI 10100, 10200 |
| EVPN Type-2 | `show bgp evpn route-type mac-ip` | Leaf | MAC/IP маршруты хостов |
| EVPN Type-5 | `show bgp evpn route-type ip-prefix` | Leaf | IP Prefix маршруты подсетей |
| VRF-маршрутизация | `show ip route vrf TENANT` | Leaf | Маршруты через Vxlan1 |
| L3 VNI | `show vxlan vrf` | Leaf | VNI 50000 в состоянии `Up` |
| ARP в VRF | `show ip arp vrf TENANT` | Leaf | RMAC для удалённых хостов |
| Ping между VNI | `ping 172.16.20.12` | Host-1 | `Success rate is 100 percent` |
| Traceroute между VNI | `traceroute 172.16.20.12` | Host-1 | 2 хопа через Anycast Gateway |
| Traceroute внутри VNI | `traceroute 172.16.20.13` | Host-2 | 1 хоп — L2 VNI |

---

## 9. Заключение

В ходе работы настроена Overlay-сеть на основе VXLAN EVPN **с маршрутизацией между VNI** (L3 VNI):

- **L2 VNI** (10100 для VLAN 10, 10200 для VLAN 20) обеспечивают L2-связность клиентов внутри одного VNI.
- **L3 VNI** (50000) обеспечивает маршрутизацию между клиентами в разных VNI через **EVPN Symmetric IRB**.
- **VRF TENANT** изолирует клиентскую маршрутизацию.
- **Anycast Gateway** (`172.16.10.1`, `172.16.20.1`) настроен на всех Leaf с одинаковым MAC (`0000.aaaa.bbbb`).
- **Type-2 (MAC/IP)** и **Type-5 (IP Prefix)** маршруты передаются через EVPN.
- Клиентские порты (Ethernet4) переведены в access-режим.
- Проверена связность между хостами, подключёнными к разным Leaf и находящимися в разных VNI.
- Все BGP EVPN-сессии установлены, MAC-адреса изучаются через контрольную плоскость, L3-трафик между клиентами проходит без потерь.
