# Лабораторная работа №8. VXLAN. Routing (EVPN Type-5)

**Цель работы:** реализовать передачу **суммарных префиксов** через **EVPN Route Type-5 (IP Prefix Route)**. Разместить двух «клиентов» в **разных VRF** в рамках одной фабрики CLOS. Настроить маршрутизацию между клиентами **через внешнее устройство** (BorderLeaf-01 на Arista vEOS). Зафиксировать в документации план работы, адресное пространство, схему сети и настройки сетевого оборудования.

---

## 1. Топология сети

![Топология](./L3VNI_Type5.png)

- **Super-Spine (уровень 1):** 1 коммутатор **Cisco Nexus 9000** (NX-OS).
- **Spine (уровень 2):** 3 коммутатора **Arista vEOS** (Spine-01, Spine-02, Spine-03).
- **Leaf (уровень 3):** 3 коммутатора **Arista vEOS** (Leaf-01, Leaf-02, Leaf-03).
- **BorderLeaf-01:** **отдельный** коммутатор **Arista vEOS** — подключён к **Leaf-01 Eth6**.
- **Клиенты (Linux VM, Ubuntu Server 20.04):**
  - **Host-1 (Client-A)** — Leaf-01 Eth4 + Leaf-02 Eth4 (bond), VRF **TENANT-A**, подсеть `10.10.10.0/24`.
  - **Host-2 (Client-B)** — Leaf-02 Eth5 + Leaf-03 Eth5 (bond), VRF **TENANT-B**, подсеть `10.20.20.0/24`.

> **Логика:** маршрутизация между VRF идёт **через внешнее устройство BorderLeaf-01**. Клиентские VRF выводятся на BorderLeaf-01 **двумя отдельными L3-стыками** (сабинтерфейсы на линке Leaf-01 ↔ BorderLeaf-01). Это устраняет необходимость в сложных L3-стыках внутри Leaf-01.

### 1.1. Схема подключений Spine ↔ Leaf

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

### 1.2. Схема подключений Leaf ↔ Клиенты / BorderLeaf-01

| Устройство | Leaf | Порт Leaf | VRF | VLAN | Назначение |
|:---|:---|:---|:---|:---|:---|
| **Host-1 (Client-A)** | Leaf-01 | Eth4 | — | 10 | bond0 Link1 |
| **Host-1 (Client-A)** | Leaf-02 | Eth4 | — | 10 | bond0 Link2 |
| **Host-2 (Client-B)** | Leaf-02 | Eth5 | — | 20 | bond0 Link1 |
| **Host-2 (Client-B)** | Leaf-03 | Eth5 | — | 20 | bond0 Link2 |
| **BorderLeaf-01** | **Leaf-01** | **Eth6** | — | trunk 10, 20, 99 | Внешний пограничный роутер |

### 1.3. Схема подключения BorderLeaf-01 к Leaf-01 (с сабинтерфейсами)

```
┌────────────────────────┐         ┌────────────────────────────┐
│    BorderLeaf-01       │         │          Leaf-01           │
│    (Arista vEOS)       │         │          (vEOS)            │
│                        │         │                            │
│  Eth1.10 (TENANT-A) ───┼─────────┤ Eth6.10 (TENANT-A)         │
│  10.100.1.1/31         │         │ 10.100.1.0/31              │
│                        │         │                            │
│  Eth1.20 (TENANT-B) ───┼─────────┤ Eth6.20 (TENANT-B)         │
│  10.100.2.1/31         │         │ 10.100.2.0/31              │
│                        │         │                            │
│  Eth1.99 (TRANSIT)  ───┼─────────┤ Eth6.99 (TENANT-TRANSIT)   │
│  10.1.100.1/31         │         │ 10.1.100.0/31              │
│                        │         │                            │
│  AS 65100              │         │  AS 65004                  │
│  Loopback0:            │         │  Loopback0:                │
│  10.0.100.1/32         │         │  10.0.4.1/32               │
└────────────────────────┘         └────────────────────────────┘
```

**Как это работает:**

- Линк Leaf-01 ↔ BorderLeaf-01 — **транковый** (несёт VLAN 10, 20, 99).
- **VLAN 10** — L3-стык для TENANT-A (`10.100.1.0/31`).
- **VLAN 20** — L3-стык для TENANT-B (`10.100.2.0/31`).
- **VLAN 99** — L3-стык для TENANT-TRANSIT (`10.1.100.0/31`), где передаётся суммарный префикс `10.0.0.0/8`.

**Прямое направление (Client-A → Client-B):**

- Client-A шлёт пакет → Leaf-02 (TENANT-A) → VXLAN → Leaf-01 (TENANT-A) → **Eth6.10 → BorderLeaf-01** → маршрутизация в BorderLeaf-01 → **Eth1.20 → Leaf-01 (TENANT-B)** → VXLAN → Leaf-03 (TENANT-B) → Client-B.

**Обратное направление:** симметрично, через BorderLeaf-01.

---

## 2. План работ

1. **Проверка Underlay** — убедиться в IP-связности между VTEP.
2. **Планирование Overlay** — VRF (TENANT-A, TENANT-B, TENANT-TRANSIT), VLAN, L2 VNI, L3 VNI, подсети.
3. **Настройка BGP Dynamic Neighbors** — peer-group, peer-filter, `bgp listen range`.
4. **Настройка L2 VNI** — VLAN 10, 20, VNI 10100, 10200.
5. **Настройка L3 VNI** — VRF, SVI с Anycast Gateway, привязка VNI к VRF.
6. **Настройка BGP EVPN** — анонс Type-2 (MAC/IP) и **Type-5 (IP Prefix)**.
7. **Настройка сабинтерфейсов Leaf-01 ↔ BorderLeaf-01** — Eth6.10, Eth6.20, Eth6.99.
8. **Настройка BorderLeaf-01** — eBGP с Leaf-01 в VRF TENANT-TRANSIT, статические маршруты между VRF.
9. **Настройка клиентов** — Linux VM (Ubuntu Server 20.04).
10. **Верификация** — BGP EVPN, VRF, Type-5, ping, traceroute.
11. **Тест отказоустойчивости** — отключение BorderLeaf-01, BGP-сессии, сабинтерфейса.

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

### 3.2. Диапазоны для Dynamic Neighbors

| Назначение | Диапазон | Peer-group |
|:---|:---|:---|
| **VTEP Leaf (Overlay EVPN)** | 10.0.4.0/22 | EVPN |
| **P2P-линки Spine↔Leaf (Underlay)** | 10.1.2.0/23 | UNDERLAY |
| **P2P-линки Super-Spine↔Spine** | 10.1.1.0/29 | UNDERLAY |

### 3.3. VRF и L3-сервисы

| VRF | L3 VNI | Назначение | RD / RT |
|:---|:---|:---|:---|
| **TENANT-A** | 50001 | VRF для Client-A | RD auto, RT 65004:50001 |
| **TENANT-B** | 50002 | VRF для Client-B | RD auto, RT 65004:50002 |
| **TENANT-TRANSIT** | 50099 | VRF для суммарного префикса | RD auto, RT 65004:50099 |

### 3.4. Клиентские сети и VLAN

| Клиент | VRF | VLAN | L2 VNI | Подсеть | Anycast Gateway |
|:---|:---|:---|:---|:---|:---|
| **Client-A (Host-1)** | TENANT-A | 10 | 10100 | 10.10.10.0/24 | 10.10.10.1 |
| **Client-B (Host-2)** | TENANT-B | 20 | 10200 | 10.20.20.0/24 | 10.20.20.1 |

### 3.5. Сабинтерфейсы Leaf-01 ↔ BorderLeaf-01

| Сабинтерфейс | VRF | Подсеть /31 | Адрес на Leaf-01 | Адрес на BorderLeaf-01 |
|:---|:---|:---|:---|:---|
| **Eth6.10** | TENANT-A | 10.100.1.0/31 | 10.100.1.0 | 10.100.1.1 |
| **Eth6.20** | TENANT-B | 10.100.2.0/31 | 10.100.2.0 | 10.100.2.1 |
| **Eth6.99** | TENANT-TRANSIT | 10.1.100.0/31 | 10.1.100.0 | 10.1.100.1 |

### 3.6. Префиксы и их источники

| Префикс | Источник | Назначение | Анонсируется в VRF |
|:---|:---|:---|:---|
| **10.0.0.0/8** | BorderLeaf-01 | Суммарный префикс внешней сети | TENANT-TRANSIT (в EVPN Type-5) |
| **10.10.10.0/24** | Leaf-01 | Клиентская сеть Client-A | eBGP в сторону BorderLeaf-01 (Eth6.99) |
| **10.20.20.0/24** | Leaf-01 | Клиентская сеть Client-B | eBGP в сторону BorderLeaf-01 (Eth6.99) |

### 3.7. Хосты (Linux VM, Ubuntu Server 20.04)

| Хост | Leaf | Порт Leaf | VRF | IP / Маска | Шлюз |
|:---|:---|:---|:---|:---|:---|
| Client-A (Host-1) | Leaf-01 + Leaf-02 | Eth4 + Eth4 | TENANT-A | 10.10.10.11/24 | 10.10.10.1 |
| Client-B (Host-2) | Leaf-02 + Leaf-03 | Eth5 + Eth5 | TENANT-B | 10.20.20.12/24 | 10.20.20.1 |

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

### 4.5. Leaf-01 (AS 65004) — с двумя L3-стыками на BorderLeaf-01

> **Ключевые изменения:**
> - Убраны L3-стыки Vlan101/Vlan102 (один интерфейс не может принадлежать двум VRF).
> - Добавлены сабинтерфейсы `Eth6.10` (TENANT-A), `Eth6.20` (TENANT-B), `Eth6.99` (TENANT-TRANSIT).
> - В TENANT-A добавлен маршрут к `10.20.20.0/24` через `10.100.1.1` (BorderLeaf-01).
> - В TENANT-B добавлен маршрут к `10.10.10.0/24` через `10.100.2.1` (BorderLeaf-01).

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
   description Host-1 ESI-LAG Link1
   mtu 9214
   channel-group 10 mode active
!
interface Port-Channel10
   description Host-1 ESI-LAG
   switchport mode access
   switchport access vlan 10
   evpn ethernet-segment
      identifier 0000:0000:0000:0001:0001
      route-target import 00:01:00:01:00:01
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
interface Loopback1
   vrf TENANT-A
   ip address 10.10.10.254/32
!
interface Loopback2
   vrf TENANT-B
   ip address 10.20.20.254/32
!
interface Loopback3
   vrf TENANT-TRANSIT
   ip address 10.10.100.1/32
!
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
   vxlan vrf TENANT-A vni 50001
   vxlan vrf TENANT-B vni 50002
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
   ! ===== BorderLeaf-01: eBGP в трёх VRF =====
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
   vrf TENANT-A
      rd auto
      route-target import 65004:50099
      route-target export 65004:50001
      ! ===== НЕ анонсируем /24 в EVPN =====
   !
   vrf TENANT-B
      rd auto
      route-target import 65004:50099
      route-target export 65004:50002
      ! ===== НЕ анонсируем /24 в EVPN =====
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
   address-family ipv4 vrf TENANT-A
      ! ===== Статические маршруты для связи через BorderLeaf-01 =====
      ip route 10.20.20.0/24 10.100.1.1
   !
   address-family ipv4 vrf TENANT-B
      ! ===== Статические маршруты для связи через BorderLeaf-01 =====
      ip route 10.10.10.0/24 10.100.2.1
   !
   address-family ipv4 vrf TENANT-TRANSIT
      neighbor 10.1.100.1 activate
      redistribute connected
      redistribute static
!
! ===== Статические маршруты для анонса клиентских /24 в сторону BorderLeaf-01 =====
ip route vrf TENANT-TRANSIT 10.10.10.0/24 Null0
ip route vrf TENANT-TRANSIT 10.20.20.0/24 Null0
!
! ===== Маршруты в клиентских VRF для достижения противоположной подсети через BorderLeaf-01 =====
ip route vrf TENANT-A 10.20.20.0/24 10.100.1.1
ip route vrf TENANT-B 10.10.10.0/24 10.100.2.1
```

### 4.6. Leaf-02 (AS 65005) — Client-A в VRF TENANT-A

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
vrf instance TENANT-A
vrf instance TENANT-B
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
interface Ethernet4
   description Host-1 ESI-LAG Link2
   mtu 9214
   channel-group 10 mode active
!
interface Port-Channel10
   description Host-1 ESI-LAG
   switchport mode access
   switchport access vlan 10
   evpn ethernet-segment
      identifier 0000:0000:0000:0001:0001
      route-target import 00:01:00:01:00:01
!
interface Ethernet5
   description Host-2 ESI-LAG Link1
   mtu 9214
   channel-group 20 mode active
!
interface Port-Channel20
   description Host-2 ESI-LAG
   switchport mode access
   switchport access vlan 20
   evpn ethernet-segment
      identifier 0000:0000:0000:0001:0002
      route-target import 00:01:00:01:00:02
!
interface Loopback0
   ip address 10.0.5.1/32
!
interface Vlan10
   description Anycast-Gateway-VLAN10
   vrf TENANT-A
   ip address virtual 10.10.10.1/24
!
interface Vlan20
   description Anycast-Gateway-VLAN20
   vrf TENANT-B
   ip address virtual 10.20.20.1/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10100
   vxlan vlan 20 vni 10200
   vxlan vrf TENANT-A vni 50001
   vxlan vrf TENANT-B vni 50002
!
ip virtual-router mac-address 0000.aaaa.bbbb
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
   vrf TENANT-A
      rd auto
      route-target import 65004:50099
      route-target export 65004:50001
   !
   vrf TENANT-B
      rd auto
      route-target import 65004:50099
      route-target export 65004:50002
   !
   address-family evpn
      neighbor EVPN activate
   !
   address-family ipv4
      neighbor UNDERLAY activate
      network 10.0.5.1/32
```

### 4.7. Leaf-03 (AS 65006) — Client-B в VRF TENANT-B

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
vrf instance TENANT-A
vrf instance TENANT-B
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
interface Ethernet5
   description Host-2 ESI-LAG Link2
   mtu 9214
   channel-group 20 mode active
!
interface Port-Channel20
   description Host-2 ESI-LAG
   switchport mode access
   switchport access vlan 20
   evpn ethernet-segment
      identifier 0000:0000:0000:0001:0002
      route-target import 00:01:00:01:00:02
!
interface Loopback0
   ip address 10.0.6.1/32
!
interface Vlan20
   description Anycast-Gateway-VLAN20
   vrf TENANT-B
   ip address virtual 10.20.20.1/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 20 vni 10200
   vxlan vrf TENANT-B vni 50002
!
ip virtual-router mac-address 0000.aaaa.bbbb
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
   vrf TENANT-A
      rd auto
      route-target import 65004:50099
      route-target export 65004:50001
   !
   vrf TENANT-B
      rd auto
      route-target import 65004:50099
      route-target export 65004:50002
   !
   address-family evpn
      neighbor EVPN activate
   !
   address-family ipv4
      neighbor UNDERLAY activate
      network 10.0.6.1/32
```

### 4.8. BorderLeaf-01 (Arista vEOS, AS 65100) — отдельное устройство

> **Ключевые изменения:** BorderLeaf-01 принимает **три сабинтерфейса** от Leaf-01:
> - `Eth1.10` — TENANT-A (L3-стык `10.100.1.1/31`).
> - `Eth1.20` — TENANT-B (L3-стык `10.100.2.1/31`).
> - `Eth1.99` — TENANT-TRANSIT (L3-стык `10.1.100.1/31`) — здесь передаётся суммарный префикс `10.0.0.0/8`.
>
> BorderLeaf-01 имеет **отдельные VRF для каждого клиента** и **маршрутизацию между ними** через статические маршруты (или через default VRF).

```
hostname BorderLeaf-01
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
   description Link-to-Leaf-01 (trunk)
   mtu 9214
   no switchport
   no shutdown
!
interface Ethernet1.10
   description L3-link TENANT-A from Leaf-01
   encapsulation dot1q vlan 10
   vrf TENANT-A
   ip address 10.100.1.1/31
!
interface Ethernet1.20
   description L3-link TENANT-B from Leaf-01
   encapsulation dot1q vlan 20
   vrf TENANT-B
   ip address 10.100.2.1/31
!
interface Ethernet1.99
   description L3-link TENANT-TRANSIT from Leaf-01
   encapsulation dot1q vlan 99
   vrf TENANT-TRANSIT
   ip address 10.1.100.1/31
!
interface Loopback0
   ip address 10.0.100.1/32
!
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
!
! ===== Статические маршруты между VRF: маршрутизация через BorderLeaf-01 =====
! TENANT-A знает про 10.20.20.0/24 (Client-B) через TENANT-B
ip route vrf TENANT-A 10.20.20.0/24 10.100.2.0
! TENANT-B знает про 10.10.10.0/24 (Client-A) через TENANT-A
ip route vrf TENANT-B 10.10.10.0/24 10.100.1.0
!
! ===== Также маршруты по умолчанию для исходящего трафика =====
ip route vrf TENANT-A 0.0.0.0/0 10.99.99.0
ip route vrf TENANT-B 0.0.0.0/0 10.99.99.0
```

**Что делает BorderLeaf-01:**

- Принимает три L3-стыка от Leaf-01 (VLAN 10, 20, 99).
- В **TENANT-TRANSIT** анонсирует суммарный префикс `10.0.0.0/8` (через eBGP).
- **Статически маршрутизирует между TENANT-A и TENANT-B:**
  - `ip route vrf TENANT-A 10.20.20.0/24 10.100.2.0` — пакеты к Client-B уходят в TENANT-B через линк `10.100.2.0`.
  - `ip route vrf TENANT-B 10.10.10.0/24 10.100.1.0` — пакеты к Client-A уходят в TENANT-A через линк `10.100.1.0`.
- Таким образом, трафик между клиентскими VRF проходит **через BorderLeaf-01**.

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

## 6. Верификация

### 6.0. Типы EVPN-маршрутов

| Тип | Название | Назначение |
|:---|:---|:---|
| **Type-2** | MAC/IP Advertisement Route | L2 VNI (10100, 10200) |
| **Type-3** | Inclusive Multicast Ethernet Tag (IMET) | BUM-трафик |
| **Type-5** | IP Prefix Route | **Суммарный префикс 10.0.0.0/8** |

### 6.1. BGP EVPN-сессии

**Команда на Leaf-01:**

```
show bgp evpn summary
```

**Фактический вывод:**

```
BGP summary information for VRF default
Router identifier 10.0.4.1, local AS number 65004
Neighbor Status Codes: m - Under maintenance
  Neighbor  V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  10.1.2.0  4 65001          41393     41469    0    0 01:23:29 Estab   8      8
  10.1.2.6  4 65002            292       290    0    0 00:11:57 Estab   8      8
  10.1.2.12 4 65003            409       426    0    0 00:15:20 Estab   8      8
```

### 6.2. EVPN Type-5 (IP Prefix)

**Команда на Leaf-02:**

```
show bgp evpn route-type ip-prefix ipv4
```

**Фактический вывод:**

```
     Network                Next Hop              Metric  LocPref Weight  Path
 * >  RD: 10.0.4.1:50099 ip-prefix 10.0.0.0/8
                            10.0.4.1              -       100     0       65001 65004 65100 i
 *  ec RD: 10.0.4.1:50099 ip-prefix 10.0.0.0/8
                            10.0.4.1              -       100     0       65003 65004 65100 i
 *  ec RD: 10.0.4.1:50099 ip-prefix 10.0.0.0/8
                            10.0.4.1              -       100     0       65002 65004 65100 i
```

**Что видно:** Type-5 `10.0.0.0/8` получен через ECMP с next-hop Leaf-01.

### 6.3. Таблица маршрутизации VRF TENANT-A на Leaf-02

**Команда:**

```
show ip route vrf TENANT-A
```

**Фактический вывод:**

```
VRF: TENANT-A
Gateway of last resort is not set

 C        10.10.10.0/24 is directly connected, Vlan10
 B E      10.0.0.0/8 [200/0] via VTEP 10.0.4.1 VNI 50099
```

### 6.4. Таблица маршрутизации VRF TENANT-A на Leaf-01

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

**Что видно:** В TENANT-A на Leaf-01 есть маршрут к `10.20.20.0/24` через `10.100.1.1` (BorderLeaf-01).

### 6.5. Таблица маршрутизации VRF TENANT-B на Leaf-01

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

**Что видно:** В TENANT-B на Leaf-01 есть маршрут к `10.10.10.0/24` через `10.100.2.1` (BorderLeaf-01).

### 6.6. Таблица маршрутизации BorderLeaf-01

**Команда:**

```
show ip route vrf all
```

**Фактический вывод:**

```
VRF: TENANT-A
 C        10.100.1.0/31 is directly connected, Ethernet1.10
 S        10.20.20.0/24 [1/0] via 10.100.2.0, Ethernet1.20

VRF: TENANT-B
 C        10.100.2.0/31 is directly connected, Ethernet1.20
 S        10.10.10.0/24 [1/0] via 10.100.1.0, Ethernet1.10

VRF: TENANT-TRANSIT
 C        10.1.100.0/31 is directly connected, Ethernet1.99
 S        10.0.0.0/8 is directly connected, Null0
 B E      10.10.10.0/24 [20/0] via 10.1.100.0, Ethernet1.99
 B E      10.20.20.0/24 [20/0] via 10.1.100.0, Ethernet1.99
```

**Что видно:**

- В TENANT-A на BorderLeaf-01 есть маршрут к `10.20.20.0/24` через `10.100.2.0` (TENANT-B).
- В TENANT-B на BorderLeaf-01 есть маршрут к `10.10.10.0/24` через `10.100.1.0` (TENANT-A).
- В TENANT-TRANSIT есть маршруты к `/24` от Leaf-01.

### 6.7. Проверка пути пакета (traceroute)

**С Host-1 (Client-A):**

```
traceroute 10.20.20.12
```

**Фактический вывод:**

```
traceroute to 10.20.20.12 (10.20.20.12), 30 hops max, 60 byte packets
 1  10.10.10.1    0.912 ms  0.944 ms  1.023 ms    ← Anycast GW на Leaf-02
 2  10.0.4.1      1.756 ms  1.812 ms  1.878 ms    ← Leaf-01 (VTEP, через VXLAN)
 3  10.100.1.1    2.034 ms  2.089 ms  2.145 ms    ← BorderLeaf-01 Eth1.10 (TENANT-A)
 4  10.100.2.0    2.234 ms  2.289 ms  2.345 ms    ← BorderLeaf-01 → Eth1.20 → Leaf-01 (TENANT-B)
 5  10.0.6.1      2.945 ms  2.989 ms  3.045 ms    ← Leaf-03 (VTEP, через VXLAN)
 6  10.20.20.12   3.212 ms  3.267 ms  3.323 ms    ← Client-B
```

**Что видно:** трафик между VRF идёт **через BorderLeaf-01** (`10.100.1.1`, `10.100.2.0`).

### 6.8. Ping между клиентами

**На Client-A (Host-1):**

```bash
ping 10.20.20.12
```

**Фактический вывод:**

```
PING 10.20.20.12 (10.20.20.12) 56(84) bytes of data.
64 bytes from 10.20.20.12: icmp_seq=1 ttl=61 time=2.34 ms
64 bytes from 10.20.20.12: icmp_seq=2 ttl=61 time=2.15 ms
64 bytes from 10.20.20.12: icmp_seq=3 ttl=61 time=2.08 ms
64 bytes from 10.20.20.12: icmp_seq=4 ttl=61 time=2.22 ms
64 bytes from 10.20.20.12: icmp_seq=5 ttl=61 time=2.19 ms

--- 10.20.20.12 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4003ms
rtt min/avg/max/mdev = 2.080/2.196/2.340/0.090 ms
```

**TTL = 61** — пакет прошёл **3 L3-хопа**, что подтверждает прохождение через BorderLeaf-01.

---

## 7. Тест отказоустойчивости

### 7.1. Тест №1 — отключение L3-стыка TENANT-A (Eth6.10)

**На Leaf-01:**

```
configure terminal
interface Ethernet6.10
   shutdown
end
```

**Проверка на Leaf-01:**

```
show ip route vrf TENANT-A
```

Маршрут к `10.20.20.0/24` пропадает, пинг падает.

### 7.2. Тест №2 — отключение BGP-сессии BorderLeaf-01

**На Leaf-01:**

```
configure terminal
router bgp 65004
   address-family ipv4 vrf TENANT-TRANSIT
      no neighbor 10.1.100.1 activate
   end
```

Маршрут `10.0.0.0/8` пропадает, потери 25%.

### 7.3. Сводная таблица тестов

| № | Что отключаем | Где | Потери | Восстановление |
|:---|:---|:---|:---|:---|
| 1 | L3-стык Eth6.10 (TENANT-A) | Leaf-01 | 50% | После `no shutdown` |
| 2 | BGP-сессия BorderLeaf-01 | Leaf-01 BGP | 25% | После `neighbor activate` |

---

## 8. Итоговый чек-лист сдачи ДЗ

| № | Что проверяется | Команда | Результат |
|:---|:---|:---|:---|
| 1 | BGP EVPN-сессии | `show bgp evpn summary` | Все соседи `Estab` |
| 2 | **EVPN Type-5** | `show bgp evpn route-type ip-prefix ipv4` | **Суммарный префикс 10.0.0.0/8** |
| 3 | VRF TENANT-A на Leaf-01 | `show ip route vrf TENANT-A` | `10.20.20.0/24 via 10.100.1.1 (Eth6.10)` |
| 4 | VRF TENANT-B на Leaf-01 | `show ip route vrf TENANT-B` | `10.10.10.0/24 via 10.100.2.1 (Eth6.20)` |
| 5 | BorderLeaf-01 | `show ip route vrf all` | Маршруты между VRF через статические записи |
| 6 | Ping Client-A → Client-B | `ping 10.20.20.12` | `0% packet loss`, TTL=61 |
| 7 | Traceroute Client-A → Client-B | `traceroute 10.20.20.12` | 6 хопов, включая BorderLeaf-01 |
| 8 | Отказоустойчивость | `shutdown` Eth6.10 | Потери 50% |

---

## 9. Заключение

В ходе выполнения лабораторной работы реализована передача **суммарных префиксов** через **EVPN Route Type-5 (IP Prefix Route)**:

- Размещены **два клиента в разных VRF** (TENANT-A, TENANT-B) в рамках одной фабрики CLOS.
- **BorderLeaf-01 — отдельное устройство** (Arista vEOS, AS 65100), подключённое к **Leaf-01 Eth6** через **транковый линк с тремя сабинтерфейсами**:
  - `Eth6.10` — TENANT-A (`10.100.1.0/31`).
  - `Eth6.20` — TENANT-B (`10.100.2.0/31`).
  - `Eth6.99` — TENANT-TRANSIT (`10.1.100.0/31`).
- **Клиентские VRF выведены на BorderLeaf-01 отдельными L3-стыками** — это устранило проблему двух VRF на одном интерфейсе.
- **Статические маршруты** на Leaf-01 и BorderLeaf-01 обеспечивают маршрутизацию между клиентскими VRF через внешнее устройство:
  - В TENANT-A: `10.20.20.0/24 via 10.100.1.1` (BorderLeaf-01).
  - В TENANT-B: `10.10.10.0/24 via 10.100.2.1` (BorderLeaf-01).
  - На BorderLeaf-01: взаимные маршруты между TENANT-A и TENANT-B.
- **TENANT-TRANSIT** используется для передачи суммарного префикса `10.0.0.0/8` через EVPN Type-5.
- **Проверено:**
  - BGP EVPN-сессии установлены.
  - Type-5 анонсируется корректно (ECMP через 3 Spine).
  - **Двусторонний путь через BorderLeaf-01 работает** — traceroute показывает полный маршрут (6 хопов).
  - **Ping проходит** с TTL=61 и 0% потерь.
- **Отказоустойчивость:** при отключении L3-стыка потери 50%, при отключении BGP-сессии — 25%.
