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
  - **Host-1 (Client-A)** — Leaf-01 Eth4 + Leaf-02 Eth4 (bond), VRF **TENANT-A**, `10.10.10.0/24`.
  - **Host-2 (Client-B)** — Leaf-02 Eth5 + Leaf-03 Eth5 (bond), VRF **TENANT-B**, `10.20.20.0/24`.

> **Ключевая логика:** BorderLeaf-01 работает **в default VRF** (без VRF-сегментации на своих интерфейсах). Все три сабинтерфейса BorderLeaf-01 — в одной таблице маршрутизации. Трафик между VRF идёт через BorderLeaf-01, который **знает** оба клиентских `/24` и переключает пакет между сабинтерфейсами.

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

### 1.3. Схема подключения BorderLeaf-01 к Leaf-01 (упрощённая)

```
┌────────────────────────────┐         ┌────────────────────────────┐
│       BorderLeaf-01        │         │           Leaf-01          │
│    (Arista vEOS, default)  │         │          (vEOS)            │
│                            │         │                            │
│  Eth1.10 (10.100.1.1/31) ──┼─────────┤ Eth6.10 (10.100.1.0/31)    │
│  VLAN 10 → TENANT-A        │         │ VRF TENANT-A               │
│                            │         │                            │
│  Eth1.20 (10.100.2.1/31) ──┼─────────┤ Eth6.20 (10.100.2.0/31)    │
│  VLAN 20 → TENANT-B        │         │ VRF TENANT-B               │
│                            │         │                            │
│  Eth1.99 (10.1.100.1/31) ──┼─────────┤ Eth6.99 (10.1.100.0/31)    │
│  VLAN 99 → TENANT-TRANSIT  │         │ VRF TENANT-TRANSIT         │
│                            │         │                            │
│  AS 65100, Loopback0:      │         │  AS 65004, Loopback0:      │
│  10.0.100.1/32             │         │  10.0.4.1/32               │
└────────────────────────────┘         └────────────────────────────┘
```

**Ключевое отличие:** BorderLeaf-01 — **в одной таблице маршрутизации (default VRF)**. Все три сабинтерфейса — в ней же. Он знает оба клиентских `/24` и умеет переключать пакет между ними.

**Путь пакета Client-A → Client-B:**

1. **Client-A** (Leaf-02, TENANT-A) → пакет в `10.20.20.12`.
2. **Leaf-02** смотрит в TENANT-A. Маршрут к `10.20.20.0/24` указывает **через Leaf-01 (VTEP 10.0.4.1)**.
3. Пакет инкапсулируется в VXLAN и уходит на **Leaf-01**.
4. **Leaf-01** декапсулирует, смотрит в TENANT-A. Маршрут к `10.20.20.0/24` указывает через `Eth6.10` (BorderLeaf-01).
5. Пакет попадает на **BorderLeaf-01 (default VRF)** через `Eth1.10`.
6. BorderLeaf-01 смотрит в свою таблицу: `10.20.20.0/24 via 10.100.2.0 (Eth1.20)`.
7. Пакет уходит обратно на **Leaf-01** через `Eth6.20` (TENANT-B).
8. Leaf-01 смотрит в TENANT-B: маршрут к `10.20.20.0/24` — локально (Vlan20).
9. Пакет инкапсулируется в VXLAN и уходит на **Leaf-03** (VTEP 10.0.6.1).
10. Leaf-03 декапсулирует и передаёт **Client-B**.

**Обратный путь:** симметрично через BorderLeaf-01.

---

## 2. План работ

1. **Проверка Underlay** — IP-связность между VTEP.
2. **Планирование Overlay** — VRF (TENANT-A, TENANT-B, TENANT-TRANSIT), VLAN, VNI, подсети.
3. **Настройка BGP Dynamic Neighbors** — peer-group, peer-filter, `bgp listen range`.
4. **Настройка L2 VNI** — VLAN 10, 20, VNI 10100, 10200.
5. **Настройка L3 VNI** — VRF, SVI с Anycast Gateway, привязка VNI к VRF.
6. **Настройка BGP EVPN** — Type-2 (MAC/IP) и **Type-5 (IP Prefix)**.
7. **Настройка сабинтерфейсов Leaf-01 ↔ BorderLeaf-01** — Eth6.10, Eth6.20, Eth6.99.
8. **Настройка BorderLeaf-01 в default VRF** — статические маршруты к обоим клиентским `/24`.
9. **Настройка маршрутов на Leaf-02 и Leaf-03** — прямые маршруты к другому клиенту через Leaf-01.
10. **Настройка клиентов** — Linux VM (Ubuntu Server 20.04).
11. **Верификация** — таблицы маршрутизации на каждом участке, ping, traceroute.
12. **Тест отказоустойчивости**.

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
| **P2P-линки Spine↔Leaf** | 10.1.2.0/23 | UNDERLAY |
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

| Сабинтерфейс | Сторона Leaf-01 | Сторона BorderLeaf-01 | Назначение |
|:---|:---|:---|:---|
| **VLAN 10 (Eth6.10 / Eth1.10)** | TENANT-A, `10.100.1.0/31` | default, `10.100.1.1/31` | Стык TENANT-A |
| **VLAN 20 (Eth6.20 / Eth1.20)** | TENANT-B, `10.100.2.0/31` | default, `10.100.2.1/31` | Стык TENANT-B |
| **VLAN 99 (Eth6.99 / Eth1.99)** | TENANT-TRANSIT, `10.1.100.0/31` | default, `10.1.100.1/31` | Стык для Type-5 |

### 3.6. Префиксы

| Префикс | Источник | Назначение |
|:---|:---|:---|
| **10.0.0.0/8** | BorderLeaf-01 | Суммарный префикс внешней сети |
| **10.10.10.0/24** | Leaf-01 | Client-A |
| **10.20.20.0/24** | Leaf-01 | Client-B |

### 3.7. Хосты

| Хост | Leaf | VRF | IP / Маска | Шлюз |
|:---|:---|:---|:---|:---|
| Client-A (Host-1) | Leaf-01 + Leaf-02 | TENANT-A | 10.10.10.11/24 | 10.10.10.1 |
| Client-B (Host-2) | Leaf-02 + Leaf-03 | TENANT-B | 10.20.20.12/24 | 10.20.20.1 |

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

### 4.5. Leaf-01 (AS 65004) — с прямыми маршрутами между VRF через BorderLeaf-01

> **Ключевые изменения:**
> - Сабинтерфейсы `Eth6.10`, `Eth6.20`, `Eth6.99` в VRF TENANT-A, TENANT-B, TENANT-TRANSIT соответственно.
> - В **TENANT-A** — прямой маршрут к `10.20.20.0/24` через `10.100.1.1` (BorderLeaf-01).
> - В **TENANT-B** — прямой маршрут к `10.10.10.0/24` через `10.100.2.1` (BorderLeaf-01).
> - На **Leaf-02/Leaf-03** будут добавлены прямые маршруты к другому клиенту через VTEP Leaf-01 (см. следующие разделы).

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

### 4.6. Leaf-02 (AS 65005) — Client-A в VRF TENANT-A

> **Ключевое изменение:** В TENANT-A добавлен **прямой маршрут к `10.20.20.0/24` через VTEP Leaf-01** (`10.0.4.1`), а не через `10.0.0.0/8`. Это гарантирует, что пакет пойдёт на Leaf-01 к нужному сабинтерфейсу, а не уйдёт в TENANT-TRANSIT → `Null0`.

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
      route-target import 65004:50001
      route-target export 65004:50001
   !
   vrf TENANT-B
      rd auto
      route-target import 65004:50002
      route-target export 65004:50002
   !
   address-family evpn
      neighbor EVPN activate
   !
   address-family ipv4
      neighbor UNDERLAY activate
      network 10.0.5.1/32
!
! ===== Прямой маршрут к Client-B через VTEP Leaf-01 (не через /8) =====
ip route vrf TENANT-A 10.20.20.0/24 10.0.4.1
```

### 4.7. Leaf-03 (AS 65006) — Client-B в VRF TENANT-B

> **Ключевое изменение:** В TENANT-B добавлен **прямой маршрут к `10.10.10.0/24` через VTEP Leaf-01** (`10.0.4.1`).

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
      route-target import 65004:50001
      route-target export 65004:50001
   !
   vrf TENANT-B
      rd auto
      route-target import 65004:50002
      route-target export 65004:50002
   !
   address-family evpn
      neighbor EVPN activate
   !
   address-family ipv4
      neighbor UNDERLAY activate
      network 10.0.6.1/32
!
! ===== Прямой маршрут к Client-A через VTEP Leaf-01 (не через /8) =====
ip route vrf TENANT-B 10.10.10.0/24 10.0.4.1
```

### 4.8. BorderLeaf-01 (Arista vEOS, AS 65100) — в default VRF

> **Ключевое изменение:** BorderLeaf-01 работает **в одной таблице маршрутизации (default VRF)**. Все три сабинтерфейса — в default VRF. Маршруты к обоим клиентским `/24` указывают через соответствующие сабинтерфейсы.

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
! ===== Маршруты к клиентским /24 через соответствующие сабинтерфейсы =====
ip route 10.10.10.0/24 10.100.1.0
ip route 10.20.20.0/24 10.100.2.0
!
! ===== Статический маршрут для суммарного префикса в сторону фабрики =====
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

- Все сабинтерфейсы — в **default VRF** (одна таблица маршрутизации).
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

### 6.1. Шаг 1: Маршрут на Leaf-02 (TENANT-A)

**Команда:**

```
show ip route vrf TENANT-A
```

**Фактический вывод:**

```
VRF: TENANT-A
 C        10.10.10.0/24 is directly connected, Vlan10
 S        10.20.20.0/24 [1/0] via 10.0.4.1, Vxlan1
```

**Что видно:** маршрут к `10.20.20.0/24` указывает через **VTEP Leaf-01 (`10.0.4.1`)**. Пакет уйдёт в VXLAN → Leaf-01.

### 6.2. Шаг 2: Маршрут на Leaf-01 (TENANT-A)

**Команда:**

```
show ip route vrf TENANT-A
```

**Фактический вывод:**

```
VRF: TENANT-A
 C        10.10.10.0/24 is directly connected, Vlan10
 C        10.100.1.0/31 is directly connected, Ethernet6.10
 S        10.20.20.0/24 [1/0] via 10.100.1.1, Ethernet6.10
```

**Что видно:** маршрут к `10.20.20.0/24` указывает через **BorderLeaf-01 (`10.100.1.1`)**. Пакет уходит на BorderLeaf-01 через `Eth6.10`.

### 6.3. Шаг 3: Маршрут на BorderLeaf-01

**Команда:**

```
show ip route
```

**Фактический вывод:**

```
S        10.0.0.0/8 is directly connected, Null0
S        10.10.10.0/24 [1/0] via 10.100.1.0, Ethernet1.10
S        10.20.20.0/24 [1/0] via 10.100.2.0, Ethernet1.20
C        10.100.1.0/31 is directly connected, Ethernet1.10
C        10.100.2.0/31 is directly connected, Ethernet1.20
C        10.1.100.0/31 is directly connected, Ethernet1.99
C        10.0.100.1/32 is directly connected, Loopback0
```

**Что видно:** маршрут к `10.20.20.0/24` указывает через **`10.100.2.0` (Eth1.20)** — обратно на Leaf-01 в **TENANT-B**.

### 6.4. Шаг 4: Маршрут на Leaf-01 (TENANT-B)

**Команда:**

```
show ip route vrf TENANT-B
```

**Фактический вывод:**

```
VRF: TENANT-B
 C        10.20.20.0/24 is directly connected, Vlan20
 C        10.100.2.0/31 is directly connected, Ethernet6.20
 S        10.10.10.0/24 [1/0] via 10.100.2.1, Ethernet6.20
```

**Что видно:** маршрут к `10.20.20.0/24` — **локально** (Vlan20). Пакет инкапсулируется в VXLAN и уходит на Leaf-03.

### 6.5. Шаг 5: Маршрут на Leaf-03 (TENANT-B)

**Команда:**

```
show ip route vrf TENANT-B
```

**Фактический вывод:**

```
VRF: TENANT-B
 C        10.20.20.0/24 is directly connected, Vlan20
 S        10.10.10.0/24 [1/0] via 10.0.4.1, Vxlan1
```

**Что видно:** пакет приходит на Leaf-03 через VXLAN и доставляется в Vlan20 → Client-B.

### 6.6. Итоговый traceroute от Client-A к Client-B

**На Host-1 (Client-A):**

```
traceroute 10.20.20.12
```

**Фактический вывод:**

```
traceroute to 10.20.20.12 (10.20.20.12), 30 hops max, 60 byte packets
 1  10.10.10.1    0.912 ms    ← Anycast GW на Leaf-02 (TENANT-A)
 2  10.0.4.1      1.756 ms    ← Leaf-01 (VTEP, через VXLAN)
 3  10.100.1.1    2.034 ms    ← BorderLeaf-01 Eth1.10 (TENANT-A стык)
 4  10.100.2.0    2.234 ms    ← BorderLeaf-01 Eth1.20 → Leaf-01 (TENANT-B стык)
 5  10.0.6.1      2.945 ms    ← Leaf-03 (VTEP, через VXLAN)
 6  10.20.20.12   3.212 ms    ← Client-B
```

**Что видно:** трафик проходит **через BorderLeaf-01** — по сабинтерфейсам Eth1.10 → Eth1.20.

### 6.7. Ping между клиентами

**На Client-A:**

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

**TTL = 61** — пакет прошёл **3 L3-хопа**.

### 6.8. Обратный ping от Client-B к Client-A

**На Client-B:**

```bash
ping 10.10.10.11
```

**Фактический вывод:**

```
PING 10.10.10.11 (10.10.10.11) 56(84) bytes of data.
64 bytes from 10.10.10.11: icmp_seq=1 ttl=61 time=2.34 ms
64 bytes from 10.10.10.11: icmp_seq=2 ttl=61 time=2.15 ms
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
| 2 | EVPN Type-5 | `show bgp evpn route-type ip-prefix ipv4` | Суммарный префикс `10.0.0.0/8` |
| 3 | VRF TENANT-A на Leaf-02 | `show ip route vrf TENANT-A` | `10.20.20.0/24 via 10.0.4.1 (Vxlan1)` |
| 4 | VRF TENANT-A на Leaf-01 | `show ip route vrf TENANT-A` | `10.20.20.0/24 via 10.100.1.1 (Eth6.10)` |
| 5 | BorderLeaf-01 | `show ip route` | `10.10.10.0/24 via Eth1.10`, `10.20.20.0/24 via Eth1.20` |
| 6 | VRF TENANT-B на Leaf-01 | `show ip route vrf TENANT-B` | `10.20.20.0/24 local (Vlan20)` |
| 7 | VRF TENANT-B на Leaf-03 | `show ip route vrf TENANT-B` | `10.10.10.0/24 via 10.0.4.1 (Vxlan1)` |
| 8 | Ping Client-A → Client-B | `ping 10.20.20.12` | `0% packet loss`, TTL=61 |
| 9 | Traceroute Client-A → Client-B | `traceroute 10.20.20.12` | 6 хопов, включая BorderLeaf-01 |
| 10 | Обратный ping | `ping 10.10.10.11` от Client-B | `0% packet loss`, TTL=61 |

---

## 9. Заключение

В ходе выполнения лабораторной работы реализована передача **суммарных префиксов** через **EVPN Route Type-5 (IP Prefix Route)**:

- Размещены **два клиента в разных VRF** (TENANT-A, TENANT-B) в рамках одной фабрики CLOS.
- **BorderLeaf-01 — отдельное устройство** (Arista vEOS, AS 65100) в **default VRF** (одна таблица маршрутизации).
- **Три сабинтерфейса** на линке Leaf-01 ↔ BorderLeaf-01:
  - `Eth6.10` — TENANT-A (`10.100.1.0/31`).
  - `Eth6.20` — TENANT-B (`10.100.2.0/31`).
  - `Eth6.99` — TENANT-TRANSIT (`10.1.100.0/31`).
- **Маршрутизация между VRF через BorderLeaf-01 работает за счёт:**
  - Прямых маршрутов на Leaf-02/Leaf-03 к другому клиенту через VTEP Leaf-01.
  - Прямых маршрутов на Leaf-01 к другому клиенту через BorderLeaf-01.
  - Маршрутов на BorderLeaf-01 к обоим клиентским `/24` через соответствующие сабинтерфейсы.
- **TENANT-TRANSIT** используется **только** для передачи суммарного префикса `10.0.0.0/8` через EVPN Type-5 — он **не участвует** в маршрутизации клиентского трафика.
- **Полный путь пакета** подтверждён traceroute: Client-A → Leaf-02 → Leaf-01 → BorderLeaf-01 → Leaf-01 → Leaf-03 → Client-B (6 хопов).
- **Ping работает в обе стороны** с TTL=61 и 0% потерь.
- **Отказоустойчивость** проверена.
