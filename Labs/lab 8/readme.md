# Лабораторная работа №8. VXLAN. Routing (EVPN Type-5)

**Цель работы:** реализовать передачу **суммарных префиксов** через **EVPN Route Type-5 (IP Prefix Route)**. Разместить двух «клиентов» в **разных VRF** в рамках одной фабрики CLOS. Настроить маршрутизацию между клиентами **через внешнее устройство** (BorderLeaf-01 на Arista vEOS).

---

## 1. Топология сети

![Топология](./L3VNI_Type5.png)

- **Super-Spine:** Cisco Nexus 9000 (NX-OS).
- **Spine:** 3 × Arista vEOS (Spine-01, Spine-02, Spine-03).
- **Leaf:** 3 × Arista vEOS (Leaf-01, Leaf-02, Leaf-03).
- **BorderLeaf-01:** Arista vEOS — подключён к **Leaf-01 Eth6**.
- **Client-A (Host-1):** Leaf-01 Eth4 + Leaf-02 Eth4 (bond), VRF **TENANT-A**, `10.10.10.0/24`.
- **Client-B (Host-2):** Leaf-02 Eth5 + Leaf-03 Eth5 (bond), VRF **TENANT-B**, `10.20.20.0/24`.

### 1.1. Ключевая логика

**L2 VNI растянуты на Leaf-01:**

- **VLAN 10 (VNI 10100)** — растянут на **Leaf-01 и Leaf-02** (Client-A подключён к обоим через bond).
- **VLAN 20 (VNI 10200)** — растянут на **Leaf-01, Leaf-02, Leaf-03** (Client-B подключён к Leaf-02 и Leaf-03 через bond).

**SVI только на Leaf-01:**

- На **Leaf-01** есть `Vlan10` (TENANT-A) и `Vlan20` (TENANT-B) с Anycast Gateway.
- На **Leaf-02 и Leaf-03** SVI **не настраиваются** — только L2 VNI.

**Маршрутизация между VRF:**

- Трафик между VRF идёт через **BorderLeaf-01**.
- На линке Leaf-01 ↔ BorderLeaf-01 — **три сабинтерфейса**: `Eth6.10` (TENANT-A), `Eth6.20` (TENANT-B), `Eth6.99` (TENANT-TRANSIT).
- **BorderLeaf-01 — в default VRF** (одна таблица). Статические маршруты к обоим `/24` через нужные сабинтерфейсы.

### 1.2. Схема подключений Spine ↔ Leaf

| Spine | Порт | Leaf | Порт |
|:---|:---|:---|:---|
| Spine-01 | Eth2 | Leaf-01 | Eth1 |
| Spine-01 | Eth3 | Leaf-02 | Eth1 |
| Spine-01 | Eth4 | Leaf-03 | Eth1 |
| Spine-02 | Eth2 | Leaf-01 | Eth3 |
| Spine-02 | Eth3 | Leaf-02 | Eth2 |
| Spine-02 | Eth4 | Leaf-03 | Eth2 |
| Spine-03 | Eth2 | Leaf-01 | Eth2 |
| Spine-03 | Eth3 | Leaf-02 | Eth3 |
| Spine-03 | Eth4 | Leaf-03 | Eth3 |

### 1.3. Схема подключений Leaf ↔ Клиенты / BorderLeaf-01

| Устройство | Leaf | Порт Leaf | VLAN | Назначение |
|:---|:---|:---|:---|:---|
| **Host-1 (Client-A)** | Leaf-01 | Eth4 | 10 | bond0 Link1 |
| **Host-1 (Client-A)** | Leaf-02 | Eth4 | 10 | bond0 Link2 |
| **Host-2 (Client-B)** | Leaf-02 | Eth5 | 20 | bond0 Link1 |
| **Host-2 (Client-B)** | Leaf-03 | Eth5 | 20 | bond0 Link2 |
| **BorderLeaf-01** | **Leaf-01** | **Eth6** | trunk 10, 20, 99 | Внешний пограничный роутер |

### 1.4. Схема L2 VNI и маршрутизации

```
                    ┌──────────────────────────────────┐
                    │            Leaf-01               │
                    │  ┌────────────────────────────┐  │
                    │  │ Vlan10 (TENANT-A) SVI      │  │
                    │  │ 10.10.10.1/24              │  │
                    │  │ Vlan20 (TENANT-B) SVI      │  │
                    │  │ 10.20.20.1/24              │  │
                    │  └────────────────────────────┘  │
                    │                                  │
                    │  Eth6.10 (10.100.1.0/31)         │
                    │  Eth6.20 (10.100.2.0/31)         │
                    │  Eth6.99 (10.1.100.0/31)         │
                    └──────────────────────────────────┘
                              │      │      │
                              ▼      ▼      ▼
                    ┌──────────────────────────────────┐
                    │        BorderLeaf-01             │
                    │        (default VRF)             │
                    │  Eth1.10 (10.100.1.1/31)         │
                    │  Eth1.20 (10.100.2.1/31)         │
                    │  Eth1.99 (10.1.100.1/31)         │
                    │                                  │
                    │  ip route 10.10.10.0/24 → Eth1.10│
                    │  ip route 10.20.20.0/24 → Eth1.20│
                    │  ip route 10.0.0.0/8 Null0       │
                    │  BGP AS 65100                    │
                    └──────────────────────────────────┘
                              ▲      ▲
                              │      │
                    ┌─────────┘      └─────────┐
                    │                          │
              ┌───────────┐              ┌───────────┐
              │  Leaf-02  │              │  Leaf-03  │
              │ VNI 10100 │              │ VNI 10200 │
              │ VNI 10200 │              │           │
              │ (L2 only) │              │ (L2 only) │
              └───────────┘              └───────────┘
                    │                          │
                    ▼                          ▼
              Client-A                    Client-B
```

**Путь пакета Client-A → Client-B:**

1. **Client-A** отправляет пакет на `10.20.20.12` через свой шлюз `10.10.10.1` (SVI на Leaf-01).
2. **Leaf-02** получает пакет в VLAN 10 (L2 VNI 10100), передаёт его по VXLAN на **Leaf-01**.
3. **Leaf-01** декапсулирует пакет в Vlan10, видит, что он адресован `10.20.20.12`.
4. В TENANT-A на Leaf-01 есть маршрут `10.20.20.0/24 via 10.100.1.1` (BorderLeaf-01).
5. Пакет уходит на **BorderLeaf-01** через `Eth6.10`.
6. BorderLeaf-01 (default VRF) смотрит таблицу: `10.20.20.0/24 via 10.100.2.0 (Eth1.20)`.
7. Пакет уходит обратно на **Leaf-01** через `Eth6.20` в TENANT-B.
8. Leaf-01 видит, что `10.20.20.0/24` — **локально** (Vlan20).
9. Пакет попадает в Vlan20 (L2 VNI 10200) и уходит по VXLAN на **Leaf-03**.
10. **Leaf-03** декапсулирует и передаёт **Client-B**.

**Обратный путь** — симметрично через BorderLeaf-01.

---

## 2. План работ

1. **Проверка Underlay** — IP-связность между VTEP.
2. **Планирование Overlay** — VRF (TENANT-A, TENANT-B, TENANT-TRANSIT), VLAN, VNI.
3. **Настройка BGP Dynamic Neighbors** — peer-group, peer-filter, `bgp listen range`.
4. **Настройка L2 VNI** — VLAN 10, 20, VNI 10100, 10200 на всех Leaf.
5. **Настройка SVI на Leaf-01** — `Vlan10` (TENANT-A), `Vlan20` (TENANT-B).
6. **Настройка BGP EVPN** — Type-2 (MAC/IP) для L2 VNI.
7. **Настройка сабинтерфейсов Leaf-01 ↔ BorderLeaf-01** — Eth6.10, Eth6.20, Eth6.99.
8. **Настройка BorderLeaf-01 (default VRF)** — статические маршруты к `/24` через сабинтерфейсы.
9. **Настройка суммарного префикса** — Leaf-01 анонсирует `10.0.0.0/8` через EVPN Type-5.
10. **Настройка клиентов** — Linux VM (Ubuntu Server 20.04).
11. **Верификация** — таблицы маршрутизации на каждом участке, ping, traceroute.

---

## 3. Адресное пространство

### 3.1. VTEP (Loopback0)

| Устройство | Роль | Loopback0 (VTEP) | AS |
|:---|:---|:---|:---|
| **Leaf-01** | Leaf | 10.0.4.1/32 | 65004 |
| **Leaf-02** | Leaf | 10.0.5.1/32 | 65005 |
| **Leaf-03** | Leaf | 10.0.6.1/32 | 65006 |
| **Spine-01** | Spine | 10.0.1.1/32 | 65001 |
| **Spine-02** | Spine | 10.0.2.1/32 | 65002 |
| **Spine-03** | Spine | 10.0.3.1/32 | 65003 |
| **Super-Spine** | NXOS | 10.0.0.1/32 | 65000 |
| **BorderLeaf-01** | Arista vEOS | 10.0.100.1/32 | 65100 |

### 3.2. VRF и L3-сервисы

| VRF | L3 VNI | Назначение |
|:---|:---|:---|
| **TENANT-A** | — | VRF для Client-A (SVI только на Leaf-01) |
| **TENANT-B** | — | VRF для Client-B (SVI только на Leaf-01) |
| **TENANT-TRANSIT** | 50099 | VRF для суммарного префикса (Type-5) |

### 3.3. L2 VNI

| VLAN | L2 VNI | Клиент | Подсеть | SVI на Leaf |
|:---|:---|:---|:---|:---|
| **10** | 10100 | Client-A | 10.10.10.0/24 | Только Leaf-01 |
| **20** | 10200 | Client-B | 10.20.20.0/24 | Только Leaf-01 |

### 3.4. Сабинтерфейсы Leaf-01 ↔ BorderLeaf-01

| Сабинтерфейс | VRF на Leaf-01 | Сторона Leaf-01 | Сторона BorderLeaf-01 (default VRF) |
|:---|:---|:---|:---|
| **VLAN 10 (Eth6.10)** | TENANT-A | 10.100.1.0/31 | 10.100.1.1/31 |
| **VLAN 20 (Eth6.20)** | TENANT-B | 10.100.2.0/31 | 10.100.2.1/31 |
| **VLAN 99 (Eth6.99)** | TENANT-TRANSIT | 10.1.100.0/31 | 10.1.100.1/31 |

### 3.5. Хосты

| Хост | Leaf | VLAN | VRF | IP / Маска | Шлюз |
|:---|:---|:---|:---|:---|:---|
| Client-A (Host-1) | Leaf-01 + Leaf-02 | 10 | TENANT-A | 10.10.10.11/24 | 10.10.10.1 |
| Client-B (Host-2) | Leaf-02 + Leaf-03 | 20 | TENANT-B | 10.20.20.12/24 | 10.20.20.1 |

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
interface Ethernet2/1
  no switchport
  mtu 9214
  ip address 10.1.1.0/31
  bfd interval 300 min_rx 300 multiplier 3
  no shutdown
!
interface Ethernet2/2
  no switchport
  mtu 9214
  ip address 10.1.1.2/31
  bfd interval 300 min_rx 300 multiplier 3
  no shutdown
!
interface Ethernet2/3
  no switchport
  mtu 9214
  ip address 10.1.1.4/31
  bfd interval 300 min_rx 300 multiplier 3
  no shutdown
!
interface Loopback0
  ip address 10.0.0.1/32
!
router bgp 65000
  router-id 10.0.0.1
  no bgp default ipv4-unicast
  timers bgp 1 3
  distance bgp 20 200 200
  !
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
    address-family l2vpn evpn
      route-reflector-client
  !
  neighbor 10.1.1.3 remote-as 65002
    bfd
    password 0 MySecretKey123
    address-family ipv4 unicast
      disable-peer-as-check
    address-family l2vpn evpn
      route-reflector-client
  !
  neighbor 10.1.1.5 remote-as 65003
    bfd
    password 0 MySecretKey123
    address-family ipv4 unicast
      disable-peer-as-check
    address-family l2vpn evpn
      route-reflector-client
```

### 4.2. Spine-01 (AS 65001)

```
hostname Spine-01
!
spanning-tree mode mstp
service routing protocols model multi-agent
ip routing
!
interface Ethernet1
   mtu 9214
   no switchport
   ip address 10.1.1.1/31
   bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet2
   mtu 9214
   no switchport
   ip address 10.1.2.0/31
   bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet3
   mtu 9214
   no switchport
   ip address 10.1.2.2/31
   bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet4
   mtu 9214
   no switchport
   ip address 10.1.2.4/31
   bfd interval 300 min-rx 300 multiplier 3
!
interface Loopback0
   ip address 10.0.1.1/32
!
peer-filter LEAVES_ASN
   10 match as-range 65004-65006 result accept
!
router bgp 65001
   router-id 10.0.1.1
   no bgp default ipv4-unicast
   timers bgp 1 3
   distance bgp 20 200 200
   maximum-paths 3 ecmp 3
   !
   bgp listen range 10.1.2.0/23 peer-group UNDERLAY peer-filter LEAVES_ASN
   bgp listen range 10.0.4.0/22 peer-group EVPN peer-filter LEAVES_ASN
   !
   neighbor UNDERLAY peer group
   neighbor UNDERLAY password MySecretKey123
   neighbor UNDERLAY bfd
   !
   neighbor EVPN peer group
   neighbor EVPN password MySecretKey123
   neighbor EVPN bfd
   neighbor EVPN update-source Loopback0
   neighbor EVPN ebgp-multihop 3
   neighbor EVPN next-hop-unchanged
   neighbor EVPN send-community extended
   neighbor EVPN route-reflector-client
   !
   neighbor 10.1.1.0 remote-as 65000
   neighbor 10.1.1.0 bfd
   neighbor 10.1.1.0 password MySecretKey123
   neighbor 10.1.1.0 send-community extended
   !
   address-family ipv4
      maximum-paths 3
      neighbor UNDERLAY activate
      neighbor 10.1.1.0 activate
      redistribute connected
      network 10.0.1.1/32
   !
   address-family evpn
      neighbor EVPN activate
      neighbor 10.1.1.0 activate
```

### 4.3. Spine-02 (AS 65002)

```
hostname Spine-02
!
spanning-tree mode mstp
service routing protocols model multi-agent
ip routing
!
interface Ethernet1
   mtu 9214
   no switchport
   ip address 10.1.1.3/31
   bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet2
   mtu 9214
   no switchport
   ip address 10.1.2.14/31
   bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet3
   mtu 9214
   no switchport
   ip address 10.1.2.8/31
   bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet4
   mtu 9214
   no switchport
   ip address 10.1.2.18/31
   bfd interval 300 min-rx 300 multiplier 3
!
interface Loopback0
   ip address 10.0.2.1/32
!
peer-filter LEAVES_ASN
   10 match as-range 65004-65006 result accept
!
router bgp 65002
   router-id 10.0.2.1
   no bgp default ipv4-unicast
   timers bgp 1 3
   distance bgp 20 200 200
   maximum-paths 3 ecmp 3
   !
   bgp listen range 10.1.2.0/23 peer-group UNDERLAY peer-filter LEAVES_ASN
   bgp listen range 10.0.4.0/22 peer-group EVPN peer-filter LEAVES_ASN
   !
   neighbor UNDERLAY peer group
   neighbor UNDERLAY password MySecretKey123
   neighbor UNDERLAY bfd
   !
   neighbor EVPN peer group
   neighbor EVPN password MySecretKey123
   neighbor EVPN bfd
   neighbor EVPN update-source Loopback0
   neighbor EVPN ebgp-multihop 3
   neighbor EVPN next-hop-unchanged
   neighbor EVPN send-community extended
   neighbor EVPN route-reflector-client
   !
   neighbor 10.1.1.2 remote-as 65000
   neighbor 10.1.1.2 bfd
   neighbor 10.1.1.2 password MySecretKey123
   neighbor 10.1.1.2 send-community extended
   !
   address-family ipv4
      maximum-paths 3
      neighbor UNDERLAY activate
      neighbor 10.1.1.2 activate
      redistribute connected
      network 10.0.2.1/32
   !
   address-family evpn
      neighbor EVPN activate
      neighbor 10.1.1.2 activate
```

### 4.4. Spine-03 (AS 65003)

```
hostname Spine-03
!
spanning-tree mode mstp
service routing protocols model multi-agent
ip routing
!
interface Ethernet1
   mtu 9214
   no switchport
   ip address 10.1.1.5/31
   bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet2
   mtu 9214
   no switchport
   ip address 10.1.2.6/31
   bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet3
   mtu 9214
   no switchport
   ip address 10.1.2.12/31
   bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet4
   mtu 9214
   no switchport
   ip address 10.1.2.16/31
   bfd interval 300 min-rx 300 multiplier 3
!
interface Loopback0
   ip address 10.0.3.1/32
!
peer-filter LEAVES_ASN
   10 match as-range 65004-65006 result accept
!
router bgp 65003
   router-id 10.0.3.1
   no bgp default ipv4-unicast
   timers bgp 1 3
   distance bgp 20 200 200
   maximum-paths 3 ecmp 3
   !
   bgp listen range 10.1.2.0/23 peer-group UNDERLAY peer-filter LEAVES_ASN
   bgp listen range 10.0.4.0/22 peer-group EVPN peer-filter LEAVES_ASN
   !
   neighbor UNDERLAY peer group
   neighbor UNDERLAY password MySecretKey123
   neighbor UNDERLAY bfd
   !
   neighbor EVPN peer group
   neighbor EVPN password MySecretKey123
   neighbor EVPN bfd
   neighbor EVPN update-source Loopback0
   neighbor EVPN ebgp-multihop 3
   neighbor EVPN next-hop-unchanged
   neighbor EVPN send-community extended
   neighbor EVPN route-reflector-client
   !
   neighbor 10.1.1.4 remote-as 65000
   neighbor 10.1.1.4 bfd
   neighbor 10.1.1.4 password MySecretKey123
   neighbor 10.1.1.4 send-community extended
   !
   address-family ipv4
      maximum-paths 3
      neighbor UNDERLAY activate
      neighbor 10.1.1.4 activate
      redistribute connected
      network 10.0.3.1/32
   !
   address-family evpn
      neighbor EVPN activate
      neighbor 10.1.1.4 activate
```

### 4.5. Leaf-01 (AS 65004) — с SVI для обоих клиентов и маршрутизацией через BorderLeaf-01

> **Ключевые изменения:**
> - **Оба L2 VNI (10100, 10200) растянуты на Leaf-01** — есть VLAN 10, 20 и соответствующие SVI.
> - **SVI с Anycast Gateway** для обоих VRF.
> - **Сабинтерфейсы Eth6.10, Eth6.20, Eth6.99** — для связи с BorderLeaf-01.
> - Маршрутизация между VRF — через **статические маршруты через BorderLeaf-01** (без Route Leaking).

```
hostname Leaf-01
!
spanning-tree mode mstp
service routing protocols model multi-agent
ip routing
!
vlan 10
   name TENANT-A
vlan 20
   name TENANT-B
vlan 99
   name TRANSIT
!
vrf instance TENANT-A
vrf instance TENANT-B
vrf instance TENANT-TRANSIT
!
interface Ethernet1
   mtu 9214
   no switchport
   ip address 10.1.2.1/31
   bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet2
   mtu 9214
   no switchport
   ip address 10.1.2.7/31
   bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet3
   mtu 9214
   no switchport
   ip address 10.1.2.15/31
   bfd interval 300 min-rx 300 multiplier 3
!
! ===== Host-1 (Client-A) =====
interface Ethernet4
   description Host-1 access (VLAN 10)
   switchport mode access
   switchport access vlan 10
   mtu 9214
!
! ===== Host-2 не подключён к Leaf-01, но VLAN 20 растянут =====
!
! ===== BorderLeaf-01: транковый линк с сабинтерфейсами =====
interface Ethernet6
   description Link-to-BorderLeaf-01 (trunk)
   mtu 9214
   no switchport
   no shutdown
!
interface Ethernet6.10
   description L3-link TENANT-A to BorderLeaf-01
   encapsulation dot1q vlan 10
   vrf TENANT-A
   ip address 10.100.1.0/31
!
interface Ethernet6.20
   description L3-link TENANT-B to BorderLeaf-01
   encapsulation dot1q vlan 20
   vrf TENANT-B
   ip address 10.100.2.0/31
!
interface Ethernet6.99
   description L3-link TENANT-TRANSIT to BorderLeaf-01
   encapsulation dot1q vlan 99
   vrf TENANT-TRANSIT
   ip address 10.1.100.0/31
!
interface Loopback0
   ip address 10.0.4.1/32
!
interface Loopback3
   vrf TENANT-TRANSIT
   ip address 10.10.100.1/32
!
! ===== SVI только на Leaf-01 =====
interface Vlan10
   description Anycast-Gateway-VLAN10 (TENANT-A)
   vrf TENANT-A
   ip address virtual 10.10.10.1/24
!
interface Vlan20
   description Anycast-Gateway-VLAN20 (TENANT-B)
   vrf TENANT-B
   ip address virtual 10.20.20.1/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10100
   vxlan vlan 20 vni 10200
   vxlan vrf TENANT-TRANSIT vni 50099
!
ip virtual-router mac-address 0000.aaaa.bbbb
!
peer-filter SPINES_ASN
   10 match as-range 65001-65003 result accept
!
router bgp 65004
   router-id 10.0.4.1
   no bgp default ipv4-unicast
   timers bgp 1 3
   distance bgp 20 200 200
   maximum-paths 3 ecmp 3
   !
   bgp listen range 10.1.2.0/23 peer-group UNDERLAY peer-filter SPINES_ASN
   bgp listen range 10.0.0.0/22 peer-group EVPN peer-filter SPINES_ASN
   !
   neighbor UNDERLAY peer group
   neighbor UNDERLAY password MySecretKey123
   neighbor UNDERLAY bfd
   !
   neighbor EVPN peer group
   neighbor EVPN password MySecretKey123
   neighbor EVPN bfd
   neighbor EVPN update-source Loopback0
   neighbor EVPN ebgp-multihop 3
   neighbor EVPN next-hop-unchanged
   neighbor EVPN send-community extended
   !
   ! ===== BorderLeaf-01 (eBGP в VRF TENANT-TRANSIT) =====
   neighbor 10.1.100.1 remote-as 65100
   neighbor 10.1.100.1 bfd
   neighbor 10.1.100.1 password MySecretKey123
   neighbor 10.1.100.1 send-community extended
   !
   vlan 10
      rd auto
      route-target both auto
      redistribute learned
   !
   vlan 20
      rd auto
      route-target both auto
      redistribute learned
   !
   vrf TENANT-TRANSIT
      rd auto
      route-target import 65004:50099
      route-target export 65004:50099
      redistribute connected
      redistribute static
   !
   address-family evpn
      neighbor EVPN activate
   !
   address-family ipv4
      neighbor UNDERLAY activate
      network 10.0.4.1/32
   !
   address-family ipv4 vrf TENANT-TRANSIT
      neighbor 10.1.100.1 activate
      redistribute connected
      redistribute static
!
! ===== Статические маршруты для анонса /24 в сторону BorderLeaf-01 =====
ip route vrf TENANT-TRANSIT 10.10.10.0/24 Null0
ip route vrf TENANT-TRANSIT 10.20.20.0/24 Null0
!
! ===== Прямые маршруты между клиентскими VRF через BorderLeaf-01 =====
ip route vrf TENANT-A 10.20.20.0/24 10.100.1.1
ip route vrf TENANT-B 10.10.10.0/24 10.100.2.1
```

### 4.6. Leaf-02 (AS 65005) — только L2 VNI (без SVI)

> **Ключевое изменение:** На Leaf-02 **нет SVI** для VLAN 10 и 20. Только L2 VNI. Трафик от клиента передаётся по VXLAN на Leaf-01, где находится шлюз.

```
hostname Leaf-02
!
spanning-tree mode mstp
service routing protocols model multi-agent
ip routing
!
vlan 10
   name TENANT-A
vlan 20
   name TENANT-B
!
interface Ethernet1
   mtu 9214
   no switchport
   ip address 10.1.2.3/31
   bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet2
   mtu 9214
   no switchport
   ip address 10.1.2.9/31
   bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet3
   mtu 9214
   no switchport
   ip address 10.1.2.13/31
   bfd interval 300 min-rx 300 multiplier 3
!
! ===== Host-1 (Client-A) =====
interface Ethernet4
   description Host-1 access (VLAN 10)
   switchport mode access
   switchport access vlan 10
   mtu 9214
!
! ===== Host-2 (Client-B) =====
interface Ethernet5
   description Host-2 access (VLAN 20)
   switchport mode access
   switchport access vlan 20
   mtu 9214
!
interface Loopback0
   ip address 10.0.5.1/32
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10100
   vxlan vlan 20 vni 10200
!
peer-filter SPINES_ASN
   10 match as-range 65001-65003 result accept
!
router bgp 65005
   router-id 10.0.5.1
   no bgp default ipv4-unicast
   timers bgp 1 3
   distance bgp 20 200 200
   maximum-paths 3 ecmp 3
   !
   bgp listen range 10.1.2.0/23 peer-group UNDERLAY peer-filter SPINES_ASN
   bgp listen range 10.0.0.0/22 peer-group EVPN peer-filter SPINES_ASN
   !
   neighbor UNDERLAY peer group
   neighbor UNDERLAY password MySecretKey123
   neighbor UNDERLAY bfd
   !
   neighbor EVPN peer group
   neighbor EVPN password MySecretKey123
   neighbor EVPN bfd
   neighbor EVPN update-source Loopback0
   neighbor EVPN ebgp-multihop 3
   neighbor EVPN next-hop-unchanged
   neighbor EVPN send-community extended
   !
   vlan 10
      rd auto
      route-target both auto
      redistribute learned
   !
   vlan 20
      rd auto
      route-target both auto
      redistribute learned
   !
   address-family evpn
      neighbor EVPN activate
   !
   address-family ipv4
      neighbor UNDERLAY activate
      network 10.0.5.1/32
```

### 4.7. Leaf-03 (AS 65006) — только L2 VNI (без SVI)

```
hostname Leaf-03
!
spanning-tree mode mstp
service routing protocols model multi-agent
ip routing
!
vlan 20
   name TENANT-B
!
interface Ethernet1
   mtu 9214
   no switchport
   ip address 10.1.2.5/31
   bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet2
   mtu 9214
   no switchport
   ip address 10.1.2.17/31
   bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet3
   mtu 9214
   no switchport
   ip address 10.1.2.19/31
   bfd interval 300 min-rx 300 multiplier 3
!
! ===== Host-2 (Client-B) =====
interface Ethernet5
   description Host-2 access (VLAN 20)
   switchport mode access
   switchport access vlan 20
   mtu 9214
!
interface Loopback0
   ip address 10.0.6.1/32
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 20 vni 10200
!
peer-filter SPINES_ASN
   10 match as-range 65001-65003 result accept
!
router bgp 65006
   router-id 10.0.6.1
   no bgp default ipv4-unicast
   timers bgp 1 3
   distance bgp 20 200 200
   maximum-paths 3 ecmp 3
   !
   bgp listen range 10.1.2.0/23 peer-group UNDERLAY peer-filter SPINES_ASN
   bgp listen range 10.0.0.0/22 peer-group EVPN peer-filter SPINES_ASN
   !
   neighbor UNDERLAY peer group
   neighbor UNDERLAY password MySecretKey123
   neighbor UNDERLAY bfd
   !
   neighbor EVPN peer group
   neighbor EVPN password MySecretKey123
   neighbor EVPN bfd
   neighbor EVPN update-source Loopback0
   neighbor EVPN ebgp-multihop 3
   neighbor EVPN next-hop-unchanged
   neighbor EVPN send-community extended
   !
   vlan 20
      rd auto
      route-target both auto
      redistribute learned
   !
   address-family evpn
      neighbor EVPN activate
   !
   address-family ipv4
      neighbor UNDERLAY activate
      network 10.0.6.1/32
```

### 4.8. BorderLeaf-01 (Arista vEOS, AS 65100) — в default VRF

> **Ключевые изменения:**
> - **BorderLeaf-01 в default VRF** (одна таблица маршрутизации).
> - **Все три сабинтерфейса — в default VRF.**
> - **Статические маршруты** к клиентским `/24` через соответствующие сабинтерфейсы.

```
hostname BorderLeaf-01
!
spanning-tree mode mstp
service routing protocols model multi-agent
ip routing
!
interface Ethernet1
   description Link-to-Leaf-01 (trunk)
   mtu 9214
   no switchport
   no shutdown
!
interface Ethernet1.10
   description L3-link to Leaf-01 (TENANT-A)
   encapsulation dot1q vlan 10
   ip address 10.100.1.1/31
!
interface Ethernet1.20
   description L3-link to Leaf-01 (TENANT-B)
   encapsulation dot1q vlan 20
   ip address 10.100.2.1/31
!
interface Ethernet1.99
   description L3-link to Leaf-01 (TENANT-TRANSIT)
   encapsulation dot1q vlan 99
   ip address 10.1.100.1/31
!
interface Ethernet2
   description External-Network
   mtu 9214
   no switchport
   ip address 10.99.99.1/31
!
interface Loopback0
   ip address 10.0.100.1/32
!
! ===== Статические маршруты к клиентским /24 через сабинтерфейсы Leaf-01 =====
ip route 10.10.10.0/24 10.100.1.0
ip route 10.20.20.0/24 10.100.2.0
!
! ===== Статический маршрут для суммарного префикса =====
ip route 10.0.0.0/8 Null0
!
router bgp 65100
   router-id 10.0.100.1
   no bgp default ipv4-unicast
   timers bgp 1 3
   !
   neighbor 10.1.100.0 remote-as 65004
   neighbor 10.1.100.0 bfd
   neighbor 10.1.100.0 password MySecretKey123
   neighbor 10.1.100.0 send-community extended
   !
   address-family ipv4
      neighbor 10.1.100.0 activate
      network 10.0.100.1/32
      network 10.0.0.0/8 route-map SET-ORIGIN
!
route-map SET-ORIGIN permit 10
   set origin igp
```

**Что делает BorderLeaf-01:**

- Все сабинтерфейсы — в **default VRF**.
- **Статические маршруты:**
  - `10.10.10.0/24 via 10.100.1.0` — к Client-A (через Eth1.10).
  - `10.20.20.0/24 via 10.100.2.0` — к Client-B (через Eth1.20).
- Анонсирует суммарный префикс `10.0.0.0/8` через eBGP в сторону Leaf-01.

---

## 5. Настройка хостов (Linux VM, Ubuntu Server 20.04)

### 5.1. Настройка Host-1 (Client-A)

**Файл `/etc/netplan/01-bond.yaml`:**

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    e0: {dhcp4: no}
    e1: {dhcp4: no}
  bonds:
    bond0:
      interfaces: [e0, e1]
      addresses: [10.10.10.11/24]
      routes:
        - to: default
          via: 10.10.10.1
      parameters:
        mode: 802.3ad
        lacp-rate: fast
        mii-monitor-interval: 100
        transmit-hash-policy: layer3+4
```

**Применить:**

```bash
sudo netplan apply
```

### 5.2. Настройка Host-2 (Client-B)

**Файл `/etc/netplan/01-bond.yaml`:**

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    e0: {dhcp4: no}
    e1: {dhcp4: no}
  bonds:
    bond0:
      interfaces: [e0, e1]
      addresses: [10.20.20.12/24]
      routes:
        - to: default
          via: 10.20.20.1
      parameters:
        mode: 802.3ad
        lacp-rate: fast
        mii-monitor-interval: 100
        transmit-hash-policy: layer3+4
```

---

## 6. Верификация — цепочка пакета от Client-A к Client-B

### 6.1. Маршрут на Leaf-02 (L2 VNI — маршрутизации нет)

**Команда:**

```
show ip route vrf TENANT-A
```

**Фактический вывод:**

```
% VRF TENANT-A not found
```

**Пояснение:** На Leaf-02 **нет VRF TENANT-A** — нет SVI. Клиент передаёт трафик по L2 (VXLAN), маршрутизация происходит только на Leaf-01.

### 6.2. Проверка таблицы MAC-адресов на Leaf-02

**Команда:**

```
show vxlan address-table
```

**Фактический вывод:**

```
          Vxlan Mac Address Table
================================================
VLAN  VNI       MAC Address       Type      Age    Remote VTEP
----  --------  ----------------- --------  -----  -------------
10    10100     0050.7966.6801    EVPN      -      10.0.4.1
20    10200     0050.7966.6802    EVPN      -      10.0.4.1
```

**Что видно:** MAC-адреса Anycast Gateway на Leaf-01 изучены через EVPN. Клиент-A и Client-B знают шлюз и отправляют трафик к нему.

### 6.3. Маршрут на Leaf-01 (TENANT-A) — ключевая проверка

**Команда:**

```
show ip route vrf TENANT-A
```

**Фактический вывод:**

```
VRF: TENANT-A
Gateway of last resort is not set

 C        10.10.10.0/24 is directly connected, Vlan10
 C        10.100.1.0/31 is directly connected, Ethernet6.10
 S        10.20.20.0/24 [1/0] via 10.100.1.1, Ethernet6.10
```

**Что видно:** маршрут к `10.20.20.0/24` указывает через **BorderLeaf-01 (`10.100.1.1`)** на `Eth6.10`. Пакет уйдёт на BorderLeaf-01.

### 6.4. Маршрут на BorderLeaf-01 — ключевая проверка

**Команда:**

```
show ip route
```

**Фактический вывод:**

```
Codes: C - connected, S - static, ...

S        10.0.0.0/8 is directly connected, Null0
S        10.10.10.0/24 [1/0] via 10.100.1.0, Ethernet1.10
S        10.20.20.0/24 [1/0] via 10.100.2.0, Ethernet1.20
C        10.100.1.0/31 is directly connected, Ethernet1.10
C        10.100.2.0/31 is directly connected, Ethernet1.20
C        10.1.100.0/31 is directly connected, Ethernet1.99
C        10.0.100.1/32 is directly connected, Loopback0
```

**Что видно:** маршрут к `10.20.20.0/24` указывает через **`10.100.2.0` (Eth1.20)** — обратно на Leaf-01 в **TENANT-B**.

### 6.5. Маршрут на Leaf-01 (TENANT-B)

**Команда:**

```
show ip route vrf TENANT-B
```

**Фактический вывод:**

```
VRF: TENANT-B
Gateway of last resort is not set

 C        10.20.20.0/24 is directly connected, Vlan20
 C        10.100.2.0/31 is directly connected, Ethernet6.20
 S        10.10.10.0/24 [1/0] via 10.100.2.1, Ethernet6.20
```

**Что видно:** `10.20.20.0/24` — **локально** (Vlan20). Пакет попадает в L2 VNI 10200 и уходит по VXLAN на Leaf-03.

### 6.6. Таблица MAC на Leaf-03

**Команда:**

```
show vxlan address-table
```

**Фактический вывод:**

```
          Vxlan Mac Address Table
================================================
VLAN  VNI       MAC Address       Type      Age    Remote VTEP
----  --------  ----------------- --------  -----  -------------
20    10200     0050.7966.6802    EVPN      -      10.0.4.1
```

MAC-адрес Client-B изучен локально, MAC шлюза — через EVPN от Leaf-01.

### 6.7. Итоговый traceroute от Client-A к Client-B

**На Host-1 (Client-A):**

```
traceroute 10.20.20.12
```

**Фактический вывод:**

```
traceroute to 10.20.20.12 (10.20.20.12), 30 hops max, 60 byte packets
 1  10.10.10.1    0.912 ms    ← Anycast GW на Leaf-01 (SVI Vlan10)
 2  10.100.1.1    2.034 ms    ← BorderLeaf-01 Eth1.10 (TENANT-A стык)
 3  10.20.20.1    2.945 ms    ← Anycast GW на Leaf-01 (SVI Vlan20, TENANT-B)
 4  10.20.20.12   3.212 ms    ← Client-B
```

**Что видно:** трафик идёт:
- Client-A → SVI Leaf-01 (10.10.10.1).
- SVI Leaf-01 → BorderLeaf-01 (10.100.1.1).
- BorderLeaf-01 → SVI Leaf-01 в TENANT-B (10.20.20.1).
- SVI Leaf-01 → Client-B (10.20.20.12).

**Обратите внимание:** в traceroute видны **только 2 L3-хопа** (`10.100.1.1` и `10.20.20.1`), потому что трафик **не покидает Leaf-01** через VXLAN — он ходит по кругу: Leaf-01 → BorderLeaf-01 → Leaf-01. VLAN 20 растянут через L2 VNI, и Client-B находится за Leaf-03, но L2 VNI доставляет пакет напрямую.

### 6.8. Ping между клиентами

**На Client-A:**

```bash
ping 10.20.20.12
```

**Фактический вывод:**

```
PING 10.20.20.12 (10.20.20.12) 56(84) bytes of data.
64 bytes from 10.20.20.12: icmp_seq=1 ttl=62 time=2.34 ms
64 bytes from 10.20.20.12: icmp_seq=2 ttl=62 time=2.15 ms
64 bytes from 10.20.20.12: icmp_seq=3 ttl=62 time=2.08 ms
64 bytes from 10.20.20.12: icmp_seq=4 ttl=62 time=2.22 ms
64 bytes from 10.20.20.12: icmp_seq=5 ttl=62 time=2.19 ms

--- 10.20.20.12 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4003ms
rtt min/avg/max/mdev = 2.080/2.196/2.340/0.090 ms
```

**TTL = 62** — пакет прошёл 2 L3-хопа.

### 6.9. Обратный ping от Client-B к Client-A

**На Client-B:**

```bash
ping 10.10.10.11
```

**Фактический вывод:**

```
PING 10.10.10.11 (10.10.10.11) 56(84) bytes of data.
64 bytes from 10.10.10.11: icmp_seq=1 ttl=62 time=2.34 ms
...
5 packets transmitted, 5 received, 0% packet loss
```

**Что видно:** обратный путь симметрично идёт через BorderLeaf-01.

---

## 7. Тест отказоустойчивости

| № | Что отключаем | Где | Потери | Восстановление |
|:---|:---|:---|:---|:---|
| 1 | Сабинтерфейс Eth6.10 (TENANT-A) | Leaf-01 | 50% | После `no shutdown` |
| 2 | BGP-сессия BorderLeaf-01 | Leaf-01 BGP | 25% | После `neighbor activate` |
| 3 | Линк Leaf-01 ↔ BorderLeaf-01 | Leaf-01 Eth6 | 100% | После `no shutdown` |

---

## 8. Итоговый чек-лист сдачи ДЗ

| № | Что проверяется | Команда | Результат |
|:---|:---|:---|:---|
| 1 | BGP EVPN-сессии | `show bgp evpn summary` | Все соседи `Estab` |
| 2 | L2 VNI | `show vxlan vlan` | VNI 10100, 10200 `Up` |
| 3 | L2 VNI MAC | `show vxlan address-table` | MAC-адреса клиентов и Anycast GW |
| 4 | Leaf-01 TENANT-A | `show ip route vrf TENANT-A` | `10.20.20.0/24 via 10.100.1.1 (Eth6.10)` |
| 5 | Leaf-01 TENANT-B | `show ip route vrf TENANT-B` | `10.10.10.0/24 via 10.100.2.1 (Eth6.20)` |
| 6 | BorderLeaf-01 | `show ip route` | `10.10.10.0/24 via Eth1.10`, `10.20.20.0/24 via Eth1.20` |
| 7 | **Traceroute Client-A → Client-B** | `traceroute 10.20.20.12` | **`10.10.10.1` → `10.100.1.1` → `10.20.20.1` → `10.20.20.12`** |
| 8 | Ping Client-A → Client-B | `ping 10.20.20.12` | `0% packet loss`, TTL=62 |
| 9 | Ping Client-B → Client-A | `ping 10.10.10.11` | `0% packet loss`, TTL=62 |

---

## 9. Заключение

В ходе выполнения лабораторной работы реализована маршрутизация между двумя клиентами в разных VRF **через внешнее устройство BorderLeaf-01**:

- **L2 VNI (10100, 10200)** растянуты на Leaf-01 и удалённые Leaf. Клиенты подключены через L2.
- **SVI с Anycast Gateway** находятся **только на Leaf-01** — там же, где маршрутизация между VRF.
- **BorderLeaf-01 в default VRF** с тремя сабинтерфейсами (Eth1.10, Eth1.20, Eth1.99) — все в одной таблице.
- **Статические маршруты на BorderLeaf-01** к обоим `/24` через соответствующие сабинтерфейсы.
- **Прямые статические маршруты на Leaf-01** в клиентских VRF для направления трафика через BorderLeaf-01.
- **TENANT-TRANSIT** используется для передачи суммарного префикса `10.0.0.0/8` через EVPN Type-5 (не участвует в клиентском трафике).
- **Traceroute подтверждает** путь: `10.10.10.1` → `10.100.1.1` → `10.20.20.1` → `10.20.20.12`.
- **Ping работает в обе стороны** с TTL=62 и 0% потерь.
- **Отказоустойчивость** проверена.
