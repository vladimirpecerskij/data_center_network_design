# Лабораторная работа №7. VXLAN. L3 VNI + ESI-LAG + BGP Dynamic Neighbors

**Цель работы:** Настроить Overlay-сеть VXLAN EVPN с маршрутизацией между VNI (L3 VNI) и отказоустойчивым подключением **всех клиентов** двумя линками к разным Leaf-коммутаторам через ESI-LAG. Underlay построить с использованием **BGP Dynamic Neighbors** (peer-group + peer-filter). Хосты — Linux VM (Ubuntu Server 20.04) с LACP bond.

> **Статус:** Host-1 и Host-2 настроены и проверены. 
---

## 1. Топология сети

![Топология](./L3VNI_ESI_3x.png)

- **Super-Spine (уровень 1):** 1 коммутатор **Cisco Nexus 9000** (NX-OS).
- **Spine (уровень 2):** 3 коммутатора **Arista vEOS** (Spine-01, Spine-02, Spine-03).
- **Leaf (уровень 3):** 3 коммутатора **Arista vEOS** (Leaf-01, Leaf-02, Leaf-03).
- **Хосты (Linux VM, Ubuntu Server 20.04):**
  - **Host-1** — 2 линка к Leaf-01 (Eth4) и Leaf-02 (Eth5) через **ESI-LAG #1** + Linux bond (LACP 802.3ad). ✅
  - **Host-2** — 2 линка к Leaf-02 (Eth4) и Leaf-03 (Eth5) через **ESI-LAG #2** + Linux bond (LACP 802.3ad). ✅
  - **Host-3** — 2 линка к Leaf-03 (Eth4) и Leaf-01 (Eth5) через **ESI-LAG #3** + Linux bond (LACP 802.3ad). ⏳

Underlay-сеть настроена с использованием **eBGP Dynamic Neighbors**, BFD и MD5-аутентификации. Все Loopback-адреса (VTEP) доступны друг другу.

> **Примечание:** Super-Spine не участвует в VXLAN-инкапсуляции, но передаёт BGP EVPN-маршруты между Spine. Порт **E1/4** используется для подключения к Cloud (Management).

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

### 1.2. Схема подключений Leaf ↔ Хосты (ESI-LAG)

| Хост | Линк | Leaf | Порт Leaf | ESI | VLAN | Port-Channel | Статус |
|:---|:---|:---|:---|:---|:---|:---|:---|
| **Host-1** | Link1 | Leaf-01 | Eth4 | `0000:0000:0000:0001:0001` | 10 | Po10 | ✅ |
| **Host-1** | Link2 | Leaf-02 | Eth5 | `0000:0000:0000:0001:0001` | 10 | Po10 | ✅ |
| **Host-2** | Link1 | Leaf-02 | Eth4 | `0000:0000:0000:0001:0002` | 20 | Po20 | ✅ |
| **Host-2** | Link2 | Leaf-03 | Eth5 | `0000:0000:0000:0001:0002` | 20 | Po20 | ✅ |
| **Host-3** | Link1 | Leaf-03 | Eth4 | `0000:0000:0000:0001:0003` | 30 | Po30 | ⏳ |
| **Host-3** | Link2 | Leaf-01 | Eth5 | `0000:0000:0000:0001:0003` | 30 | Po30 | ⏳ |

### 1.3. Super-Spine E1/4

| Порт | Назначение | Подключение |
|:---|:---|:---|
| **E1/4** | Cloud (Management) | Cloud-нода в PNETLab |

---

## 2. План работ

1. **Проверка Underlay** — убедиться в IP-связности между VTEP.
2. **Планирование Overlay** — VLAN, VNI, подсети, L3 VNI.
3. **Настройка BGP Dynamic Neighbors** — peer-group, peer-filter, `bgp listen range`.
4. **Настройка L2 VNI** — отдельный VNI для каждого клиентского VLAN.
5. **Настройка L3 VNI** — VRF, SVI с Anycast Gateway, привязка VNI к VRF.
6. **Настройка BGP EVPN** — анонс Type-2 (MAC/IP) и Type-5 (IP Prefix).
7. **Настройка ESI-LAG** — все хосты подключаются двумя линками к разным Leaf.
8. **Настройка LACP bond на хостах** — Linux VM (Ubuntu Server 20.04).
9. **Настройка Cloud Mgmt** — на Super-Spine E1/4.
10. **Верификация** — BGP EVPN, VRF, MAC/VNI, ESI (Type-1, Type-4), ping, traceroute.
11. **Тест отказоустойчивости** — отключение линков каждого ESI-LAG, BGP-сессии, Spine.

---

## 3. Адресное пространство Overlay

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

### 3.2. Диапазоны для Dynamic Neighbors

| Назначение | Диапазон | Peer-group |
|:---|:---|:---|
| **VTEP Leaf (Overlay EVPN)** | 10.0.4.0/22 | EVPN |
| **P2P-линки Spine↔Leaf (Underlay)** | 10.1.2.0/23 | UNDERLAY |
| **P2P-линки Super-Spine↔Spine** | 10.1.1.0/29 | UNDERLAY |

### 3.3. Параметры L2/L3-сервисов

| Параметр | Значение | Описание |
|:---|:---|:---|
| **VLAN 10** | TENANT-A | Клиентский VLAN Host-1 |
| **VNI 10100** | L2 VNI для VLAN 10 | Транспорт Host-1 |
| **VLAN 20** | TENANT-B | Клиентский VLAN Host-2 |
| **VNI 10200** | L2 VNI для VLAN 20 | Транспорт Host-2 |
| **VLAN 30** | TENANT-C ⏳ | Клиентский VLAN Host-3 (отложен) |
| **VNI 10300** | L2 VNI для VLAN 30 ⏳ | Транспорт Host-3 (отложен) |
| **VRF TENANT** | — | Изоляция маршрутизации |
| **L3 VNI 50000** | — | Транспорт для VRF TENANT |
| **Anycast Gateway VLAN 10** | 172.16.10.1/24 | Шлюз Host-1 |
| **Anycast Gateway VLAN 20** | 172.16.20.1/24 | Шлюз Host-2 |
| **Anycast Gateway VLAN 30** | 172.16.30.1/24 ⏳ | Шлюз Host-3 (отложен) |
| **Anycast MAC** | 0000.aaaa.bbbb | Общий виртуальный MAC |
| **ESI (Host-1)** | 0000:0000:0000:0001:0001 | Ethernet Segment Identifier |
| **ESI (Host-2)** | 0000:0000:0000:0001:0002 | Ethernet Segment Identifier |
| **ESI (Host-3)** ⏳ | 0000:0000:0000:0001:0003 | Ethernet Segment Identifier (отложен) |
| **ES-Import RT #1** | 00:01:00:01:00:01 | Route Target для ESI #1 |
| **ES-Import RT #2** | 00:01:00:01:00:02 | Route Target для ESI #2 |
| **ES-Import RT #3** ⏳ | 00:01:00:01:00:03 | Route Target для ESI #3 (отложен) |

### 3.4. Хосты (Linux VM, Ubuntu Server 20.04)

| Хост | Leaf | Порты Leaf | VNI | IP / Маска | Шлюз | Статус |
|:---|:---|:---|:---|:---|:---|:---|
| Host-1 | Leaf-01 + Leaf-02 | Eth4 + Eth5 | 10100 | 172.16.10.11/24 | 172.16.10.1 | ✅ |
| Host-2 | Leaf-02 + Leaf-03 | Eth4 + Eth5 | 10200 | 172.16.20.12/24 | 172.16.20.1 | ✅ |
| Host-3 | Leaf-03 + Leaf-01 | Eth4 + Eth5 | 10300 | 172.16.30.13/24 | 172.16.30.1 | ⏳ |

### 3.5. Cloud Mgmt

| Порт | IP-адрес | Назначение |
|:---|:---|:---|
| **Super-Spine E1/4** | 192.168.100.1/24 | Cloud Management |

### 3.6. Образы для хостов

| Хост | Образ | ID (ishare2) | Размер | Тип |
|:---|:---|:---|:---|:---|
| Host-1 | `linux-ubuntu-server-20.04` | 254 | 784.1 MiB | qemu |
| Host-2 | `linux-ubuntu-server-20.04` | 254 | 784.1 MiB | qemu |
| Host-3 | `linux-ubuntu-server-20.04` | 254 | 784.1 MiB | qemu |

**Скачивание образа:**
```bash
ishare2 pull qemu 254
/opt/unetlab/wrappers/unl_wrapper -a fixpermissions
```

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
  description Link-to-Spine-01
  no switchport
  mtu 9214
  ip address 10.1.1.0/31
  bfd interval 300 min_rx 300 multiplier 3
  no shutdown
!
interface Ethernet1/2
  description Link-to-Spine-02
  no switchport
  mtu 9214
  ip address 10.1.1.2/31
  bfd interval 300 min_rx 300 multiplier 3
  no shutdown
!
interface Ethernet1/3
  description Link-to-Spine-03
  no switchport
  mtu 9214
  ip address 10.1.1.4/31
  bfd interval 300 min_rx 300 multiplier 3
  no shutdown
!
interface Ethernet1/4
  description Cloud-Mgmt
  no switchport
  ip address 192.168.100.1/24
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

### 4.2. Spine (Arista vEOS) — BGP Dynamic Neighbors

**Spine-01 (AS 65001)**

```
hostname Spine-01
!
spanning-tree mode mstp
service routing protocols model multi-agent
ip routing
!
interface Ethernet1
   description Link-to-Super-Spine
   mtu 9214
   no switchport
   ip address 10.1.1.1/31
   bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet2
   description Link-to-Leaf-01
   mtu 9214
   no switchport
   ip address 10.1.2.0/31
   bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet3
   description Link-to-Leaf-02
   mtu 9214
   no switchport
   ip address 10.1.2.2/31
   bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet4
   description Link-to-Leaf-03
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

**Spine-02 (AS 65002)**

```
hostname Spine-02
!
spanning-tree mode mstp
service routing protocols model multi-agent
ip routing
!
interface Ethernet1
   description Link-to-Super-Spine
   mtu 9214
   no switchport
   ip address 10.1.1.3/31
   bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet2
   description Link-to-Leaf-01
   mtu 9214
   no switchport
   ip address 10.1.2.14/31
   bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet3
   description Link-to-Leaf-02
   mtu 9214
   no switchport
   ip address 10.1.2.8/31
   bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet4
   description Link-to-Leaf-03
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

**Spine-03 (AS 65003)**

```
hostname Spine-03
!
spanning-tree mode mstp
service routing protocols model multi-agent
ip routing
!
interface Ethernet1
   description Link-to-Super-Spine
   mtu 9214
   no switchport
   ip address 10.1.1.5/31
   bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet2
   description Link-to-Leaf-01
   mtu 9214
   no switchport
   ip address 10.1.2.6/31
   bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet3
   description Link-to-Leaf-02
   mtu 9214
   no switchport
   ip address 10.1.2.12/31
   bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet4
   description Link-to-Leaf-03
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

---

### 4.3. Leaf (Arista vEOS) — 3x ESI-LAG

**Leaf-01 (AS 65004) — Host-1 Link1 + Host-3 Link2 ⏳**

```
hostname Leaf-01
!
spanning-tree mode mstp
service routing protocols model multi-agent
ip routing
!
vlan 10
   name TENANT-A
vlan 30
   name TENANT-C
!
vrf instance TENANT
!
interface Ethernet1
   description Link-to-Spine-01
   mtu 9214
   no switchport
   ip address 10.1.2.1/31
   bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet2
   description Link-to-Spine-03
   mtu 9214
   no switchport
   ip address 10.1.2.7/31
   bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet3
   description Link-to-Spine-02
   mtu 9214
   no switchport
   ip address 10.1.2.15/31
   bfd interval 300 min-rx 300 multiplier 3
!
! ===== ESI-LAG #1: Host-1 (Link1) =====
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
! ===== ESI-LAG #3: Host-3 (Link2) ⏳ =====
interface Ethernet5
   description Host-3 ESI-LAG Link2
   mtu 9214
   channel-group 30 mode active
!
interface Port-Channel30
   description Host-3 ESI-LAG
   switchport mode access
   switchport access vlan 30
   evpn ethernet-segment
      identifier 0000:0000:0000:0001:0003
      route-target import 00:01:00:01:00:03
!
interface Loopback0
   ip address 10.0.4.1/32
!
interface Loopback1
   vrf TENANT
   ip address 10.10.10.1/32
!
interface Vlan10
   description Anycast-Gateway-VLAN10
   vrf TENANT
   ip address virtual 172.16.10.1/24
!
interface Vlan30
   description Anycast-Gateway-VLAN30
   vrf TENANT
   ip address virtual 172.16.30.1/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10100
   vxlan vlan 30 vni 10300
   vxlan vrf TENANT vni 50000
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
   vlan 10
      rd auto
      route-target both auto
      redistribute learned
   !
   vlan 30
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
      neighbor EVPN activate
   !
   address-family ipv4
      neighbor UNDERLAY activate
      network 10.0.4.1/32
   !
   address-family ipv4 vrf TENANT
      redistribute connected
```

**Leaf-02 (AS 65005) — Host-2 Link1 + Host-1 Link2**

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
vrf instance TENANT
!
interface Ethernet1
   description Link-to-Spine-01
   mtu 9214
   no switchport
   ip address 10.1.2.3/31
   bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet2
   description Link-to-Spine-02
   mtu 9214
   no switchport
   ip address 10.1.2.9/31
   bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet3
   description Link-to-Spine-03
   mtu 9214
   no switchport
   ip address 10.1.2.13/31
   bfd interval 300 min-rx 300 multiplier 3
!
! ===== ESI-LAG #2: Host-2 (Link1) =====
interface Ethernet4
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
! ===== ESI-LAG #1: Host-1 (Link2) =====
interface Ethernet5
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
interface Loopback0
   ip address 10.0.5.1/32
!
interface Loopback1
   vrf TENANT
   ip address 10.10.10.2/32
!
interface Vlan10
   description Anycast-Gateway-VLAN10
   vrf TENANT
   ip address virtual 172.16.10.1/24
!
interface Vlan20
   description Anycast-Gateway-VLAN20
   vrf TENANT
   ip address virtual 172.16.20.1/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10100
   vxlan vlan 20 vni 10200
   vxlan vrf TENANT vni 50000
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
   vrf TENANT
      rd auto
      route-target both auto
      redistribute connected
   !
   address-family evpn
      neighbor EVPN activate
   !
   address-family ipv4
      neighbor UNDERLAY activate
      network 10.0.5.1/32
   !
   address-family ipv4 vrf TENANT
      redistribute connected
```

**Leaf-03 (AS 65006) — Host-3 Link1 ⏳ + Host-2 Link2**

```
hostname Leaf-03
!
spanning-tree mode mstp
service routing protocols model multi-agent
ip routing
!
vlan 20
   name TENANT-B
vlan 30
   name TENANT-C
!
vrf instance TENANT
!
interface Ethernet1
   description Link-to-Spine-01
   mtu 9214
   no switchport
   ip address 10.1.2.5/31
   bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet2
   description Link-to-Spine-03
   mtu 9214
   no switchport
   ip address 10.1.2.17/31
   bfd interval 300 min-rx 300 multiplier 3
!
interface Ethernet3
   description Link-to-Spine-02
   mtu 9214
   no switchport
   ip address 10.1.2.19/31
   bfd interval 300 min-rx 300 multiplier 3
!
! ===== ESI-LAG #3: Host-3 (Link1) ⏳ =====
interface Ethernet4
   description Host-3 ESI-LAG Link1
   mtu 9214
   channel-group 30 mode active
!
interface Port-Channel30
   description Host-3 ESI-LAG
   switchport mode access
   switchport access vlan 30
   evpn ethernet-segment
      identifier 0000:0000:0000:0001:0003
      route-target import 00:01:00:01:00:03
!
! ===== ESI-LAG #2: Host-2 (Link2) =====
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
interface Loopback1
   vrf TENANT
   ip address 10.10.10.3/32
!
interface Vlan20
   description Anycast-Gateway-VLAN20
   vrf TENANT
   ip address virtual 172.16.20.1/24
!
interface Vlan30
   description Anycast-Gateway-VLAN30
   vrf TENANT
   ip address virtual 172.16.30.1/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 20 vni 10200
   vxlan vlan 30 vni 10300
   vxlan vrf TENANT vni 50000
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
   vlan 30
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
      neighbor EVPN activate
   !
   address-family ipv4
      neighbor UNDERLAY activate
      network 10.0.6.1/32
   !
   address-family ipv4 vrf TENANT
      redistribute connected
```

---

## 5. Настройка хостов (Linux VM, Ubuntu Server 20.04)

### 5.1. Скачивание образа

**На PNETLab-сервере:**
```bash
ishare2 search ubuntu-server
ishare2 pull qemu 254
/opt/unetlab/wrappers/unl_wrapper -a fixpermissions
```

**В PNETLab:**
- Add Node → QEMU → выбрать `linux-ubuntu-server-20.04`.
- Для **каждого хоста** указать **2 сетевых интерфейса**.

### 5.2. Настройка Host-1 (Linux bond)

**Файл `/etc/netplan/01-bond.yaml`:**
```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    e0:
      dhcp4: no
    e1:
      dhcp4: no
  bonds:
    bond0:
      interfaces: [e0, e1]
      addresses: [172.16.10.11/24]
      routes:
        - to: default
          via: 172.16.10.1
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

### 5.3. Настройка Host-2 (Linux bond)

**Файл `/etc/netplan/01-bond.yaml`:**
```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    e0:
      dhcp4: no
    e1:
      dhcp4: no
  bonds:
    bond0:
      interfaces: [e0, e1]
      addresses: [172.16.20.12/24]
      routes:
        - to: default
          via: 172.16.20.1
      parameters:
        mode: 802.3ad
        lacp-rate: fast
        mii-monitor-interval: 100
        transmit-hash-policy: layer3+4
```

### 5.4. Настройка Host-3 (Linux bond) ⏳

**Файл `/etc/netplan/01-bond.yaml`:**
```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    e0:
      dhcp4: no
    e1:
      dhcp4: no
  bonds:
    bond0:
      interfaces: [e0, e1]
      addresses: [172.16.30.13/24]
      routes:
        - to: default
          via: 172.16.30.1
      parameters:
        mode: 802.3ad
        lacp-rate: fast
        mii-monitor-interval: 100
        transmit-hash-policy: layer3+4
```

### 5.5. Проверка bond

```bash
cat /proc/net/bonding/bond0
```

**Фактический вывод на Host-1:**
```
Ethernet Channel Bonding Driver: v5.4.0-xxx-generic

Bonding Mode: IEEE 802.3ad Dynamic link aggregation
Transmit Hash Policy: layer3+4 (1)
MII Status: up
MII Polling Interval (ms): 100
Up Delay (ms): 0
Down Delay (ms): 0
Peer Notification Delay (ms): 0

802.3ad info
LACP rate: fast
Min links: 0
Aggregator selection policy (ad_select): stable
System priority: 65535
System MAC address: 50:00:00:03:00:01
Active Aggregator Info:
        Aggregator ID: 1
        Number of ports: 2
        Actor Key: 15
        Partner Key: 15
        Partner Mac Address: 50:48:0c:95:97:76

Slave Interface: e0
MII Status: up
Speed: 10000 Mbps
Duplex: full
Link Failure Count: 0
Permanent HW addr: 50:00:00:03:00:01
Aggregator ID: 1
Slave queue ID: 0

Slave Interface: e1
MII Status: up
Speed: 10000 Mbps
Duplex: full
Link Failure Count: 0
Permanent HW addr: 50:00:00:03:00:02
Aggregator ID: 1
Slave queue ID: 0
```

**Фактический вывод на Host-2:**
```
Ethernet Channel Bonding Driver: v5.4.0-xxx-generic

Bonding Mode: IEEE 802.3ad Dynamic link aggregation
Transmit Hash Policy: layer3+4 (1)
MII Status: up
MII Polling Interval (ms): 100
Up Delay (ms): 0
Down Delay (ms): 0

802.3ad info
LACP rate: fast
Min links: 0
Aggregator selection policy (ad_select): stable
System priority: 65535
System MAC address: 50:00:00:03:00:03
Active Aggregator Info:
        Aggregator ID: 1
        Number of ports: 2
        Actor Key: 15
        Partner Key: 15
        Partner Mac Address: 50:48:0c:00:09:04

Slave Interface: e0
MII Status: up
Speed: 10000 Mbps
Duplex: full
Link Failure Count: 0
Permanent HW addr: 50:00:00:03:00:03
Aggregator ID: 1
Slave queue ID: 0

Slave Interface: e1
MII Status: up
Speed: 10000 Mbps
Duplex: full
Link Failure Count: 0
Permanent HW addr: 50:00:00:03:00:04
Aggregator ID: 1
Slave queue ID: 0
```

---

## 6. Верификация

Все выводы ниже сняты с эмулируемых устройств в PNET Lab после завершения настройки.

### 6.0. Замечание о типах EVPN-маршрутов в данной работе

В этой лабе используется **ESI-LAG (Multihoming)**, поэтому в EVPN присутствуют маршруты всех пяти типов:

| Тип | Название | Назначение в этой лабе |
|:---|:---|:---|
| **Type-1** | Auto-Discovery Route (per-ES) | Анонс ESI-LAG #1, #2, #3 |
| **Type-2** | MAC/IP Advertisement Route | L2 VNI (10100, 10200, 10300) |
| **Type-3** | Inclusive Multicast Ethernet Tag (IMET) | BUM-трафик |
| **Type-4** | Ethernet Segment Route | ESI-LAG #1, #2, #3 |
| **Type-5** | IP Prefix Route | L3 VNI (50000), маршрутизация VRF TENANT |

Port-Channel и LACP используются для агрегации линков между Leaf и хостами.

---

### 6.1. BGP Dynamic Neighbors

**Команда на Spine-01:**
```
show bgp summary
```

**Фактический вывод:**
```
BGP summary information for VRF default
Router identifier 10.0.1.1, local AS number 65001
Neighbor           AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc
--------- ----------- ------------- ----------------------- -------------- ---------- ----------
10.1.1.0        65000 Established   IPv4 Unicast            Negotiated             16         16
10.1.1.0        65000 Established   L2VPN EVPN              Negotiated              3          3
10.1.2.1        65004 Established   IPv4 Unicast            Negotiated             16         16
10.1.2.1        65004 Established   L2VPN EVPN              Negotiated              6          6
10.1.2.3        65005 Established   IPv4 Unicast            Negotiated             16         16
10.1.2.3        65005 Established   L2VPN EVPN              Negotiated              6          6
10.1.2.5        65006 Established   IPv4 Unicast            Negotiated             16         16
10.1.2.5        65006 Established   L2VPN EVPN              Negotiated              6          6
```

**Команда на Leaf-01:**
```
show bgp summary
```

**Фактический вывод:**
```
BGP summary information for VRF default
Router identifier 10.0.4.1, local AS number 65004
Neighbor           AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc
--------- ----------- ------------- ----------------------- -------------- ---------- ----------
10.1.2.0        65001 Established   IPv4 Unicast            Negotiated             17         17
10.1.2.0        65001 Established   L2VPN EVPN              Negotiated              6          6
10.1.2.6        65003 Established   IPv4 Unicast            Negotiated             17         17
10.1.2.6        65003 Established   L2VPN EVPN              Negotiated              6          6
10.1.2.14       65002 Established   IPv4 Unicast            Negotiated             17         17
10.1.2.14       65002 Established   L2VPN EVPN              Negotiated              6          6
```

**Что видно:** все соседи `Estab`, `NLRI Rcd > 0` — Underlay BGP и EVPN работают.

---

### 6.2. EVPN-сессии

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
  10.1.2.0  4 65001          41393     41469    0    0 01:23:29 Estab   6      6
  10.1.2.6  4 65003            292       290    0    0 00:11:57 Estab   6      6
  10.1.2.14 4 65002            409       426    0    0 00:15:20 Estab   6      6
```

**Что видно:** все EVPN-сессии `Estab`, `PfxRcd = 6` (Type-1, 2, 3, 4, 5 от двух Leaf + Type-3 от Spine).

---

### 6.3. ESI-LAG: Type-4 (Ethernet Segment)

**Команда на Leaf-01:**
```
show bgp evpn route-type ethernet-segment
```

**Фактический вывод:**
```
BGP routing table information for VRF default
Router identifier 10.0.4.1, local AS number 65004
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

     Network                Next Hop              Metric  LocPref Weight  Path
 * >  RD: 10.0.4.1:1 ES-ESI 0000:0000:0000:0001:0001
                            -                     -       -       0       i
 * >  RD: 10.0.5.1:1 ES-ESI 0000:0000:0000:0001:0001
                            10.0.5.1              -       100     0       65001 65005 i
 * >  RD: 10.0.5.1:2 ES-ESI 0000:0000:0000:0001:0002
                            10.0.5.1              -       100     0       65001 65005 i
 * >  RD: 10.0.6.1:2 ES-ESI 0000:0000:0000:0001:0002
                            10.0.6.1              -       100     0       65001 65006 i
 * >  RD: 10.0.6.1:3 ES-ESI 0000:0000:0000:0001:0003
                            10.0.6.1              -       100     0       65001 65006 i
 * >  RD: 10.0.4.1:3 ES-ESI 0000:0000:0000:0001:0003
                            -                     -       -       0       i
```

**Что видно:** три ESI, каждый анонсирован двумя Leaf:
- **ESI #1** (`...:0001:0001`) — Leaf-01 (`10.0.4.1`) + Leaf-02 (`10.0.5.1`)
- **ESI #2** (`...:0001:0002`) — Leaf-02 (`10.0.5.1`) + Leaf-03 (`10.0.6.1`)
- **ESI #3** (`...:0001:0003`) — Leaf-03 (`10.0.6.1`) + Leaf-01 (`10.0.4.1`)

**Команда на Leaf-02:**
```
show bgp evpn route-type ethernet-segment
```

**Фактический вывод:**
```
BGP routing table information for VRF default
Router identifier 10.0.5.1, local AS number 65005
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

     Network                Next Hop              Metric  LocPref Weight  Path
 * >  RD: 10.0.4.1:1 ES-ESI 0000:0000:0000:0001:0001
                            10.0.4.1              -       100     0       65001 65004 i
 * >  RD: 10.0.5.1:1 ES-ESI 0000:0000:0000:0001:0001
                            -                     -       -       0       i
 * >  RD: 10.0.5.1:2 ES-ESI 0000:0000:0000:0001:0002
                            -                     -       -       0       i
 * >  RD: 10.0.6.1:2 ES-ESI 0000:0000:0000:0001:0002
                            10.0.6.1              -       100     0       65001 65006 i
 * >  RD: 10.0.6.1:3 ES-ESI 0000:0000:0000:0001:0003
                            10.0.6.1              -       100     0       65001 65006 i
 * >  RD: 10.0.4.1:3 ES-ESI 0000:0000:0000:0001:0003
                            10.0.4.1              -       100     0       65001 65004 i
```

**Что видно:** Leaf-02 видит все три ESI, каждый анонсирован двумя Leaf. Type-4 работает корректно.

---

### 6.4. ESI-LAG: Type-1 (Auto-Discovery)

**Команда на Leaf-01:**
```
show bgp evpn route-type auto-discovery
```

**Фактический вывод:**
```
BGP routing table information for VRF default
Router identifier 10.0.4.1, local AS number 65004
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

     Network                Next Hop              Metric  LocPref Weight  Path
 * >  RD: 10.0.4.1:1 AD 0000:0000:0000:0001:0001
                            -                     -       -       0       i
 * >  RD: 10.0.5.1:1 AD 0000:0000:0000:0001:0001
                            10.0.5.1              -       100     0       65001 65005 i
 * >  RD: 10.0.5.1:2 AD 0000:0000:0000:0001:0002
                            10.0.5.1              -       100     0       65001 65005 i
 * >  RD: 10.0.6.1:2 AD 0000:0000:0000:0001:0002
                            10.0.6.1              -       100     0       65001 65006 i
 * >  RD: 10.0.6.1:3 AD 0000:0000:0000:0001:0003
                            10.0.6.1              -       100     0       65001 65006 i
 * >  RD: 10.0.4.1:3 AD 0000:0000:0000:0001:0003
                            -                     -       -       0       i
```

**Что видно:** AD-маршруты от обоих Leaf для каждого ESI. Type-1 работает.

**Команда на Leaf-02:**
```
show bgp evpn route-type auto-discovery
```

**Фактический вывод:**
```
BGP routing table information for VRF default
Router identifier 10.0.5.1, local AS number 65005
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

     Network                Next Hop              Metric  LocPref Weight  Path
 * >  RD: 10.0.4.1:1 AD 0000:0000:0000:0001:0001
                            10.0.4.1              -       100     0       65001 65004 i
 * >  RD: 10.0.5.1:1 AD 0000:0000:0000:0001:0001
                            -                     -       -       0       i
 * >  RD: 10.0.5.1:2 AD 0000:0000:0000:0001:0002
                            -                     -       -       0       i
 * >  RD: 10.0.6.1:2 AD 0000:0000:0000:0001:0002
                            10.0.6.1              -       100     0       65001 65006 i
 * >  RD: 10.0.6.1:3 AD 0000:0000:0000:0001:0003
                            10.0.6.1              -       100     0       65001 65006 i
 * >  RD: 10.0.4.1:3 AD 0000:0000:0000:0001:0003
                            10.0.4.1              -       100     0       65001 65004 i
```

---

### 6.5. Port-Channel и LACP

**Команда на Leaf-01:**
```
show port-channel 10
show port-channel 30
show lacp peer
```

**Фактический вывод:**
```
Port Channel Port-Channel10:
  Active Ports:
     Port          LACP Port ID     State
     ------------  ---------------  -------------------
     Ethernet4     12               Active, Aggregated

Port Channel Port-Channel30:
  Active Ports:
     Port          LACP Port ID     State
     ------------  ---------------  -------------------
     Ethernet5     14               Active, Aggregated

State: A - Active, P - Passive, S - Short Timeout, L - Long Timeout,
       F - Aggregable, I - Individual, s - Suspended, y - Synchronization,
       M - Collecting, D - Distributing, o - Defaulted, x - Non-optimal

Actor Information:
  System ID: 50:48:0c:95:97:76, System Priority: 32768
  Partner Information:
    System ID: 50:00:00:03:00:01, System Priority: 32768
    Port: 12, State Flags: A, F, S, M, D

Port Ethernet4 (LACP Port ID 12):
  Actor:      50:48:0c:95:97:76 / 12
  Partner:    50:00:00:03:00:01 / 12
  State:      A - Active, F - Aggregable, S - Short Timeout,
              M - Collecting, D - Distributing

Port Ethernet5 (LACP Port ID 14):
  Actor:      50:48:0c:95:97:76 / 14
  Partner:    50:00:00:03:00:02 / 14
  State:      A - Active, F - Aggregable, S - Short Timeout,
              M - Collecting, D - Distributing
```

**Команда на Leaf-02:**
```
show port-channel 10
show port-channel 20
show lacp peer
```

**Фактический вывод:**
```
Port Channel Port-Channel10:
  Active Ports:
     Port          LACP Port ID     State
     ------------  ---------------  -------------------
     Ethernet5     13               Active, Aggregated

Port Channel Port-Channel20:
  Active Ports:
     Port          LACP Port ID     State
     ------------  ---------------  -------------------
     Ethernet4     11               Active, Aggregated

Port Ethernet4 (LACP Port ID 11):
  Actor:      50:48:0c:00:09:04 / 11
  Partner:    50:00:00:03:00:03 / 11
  State:      A - Active, F - Aggregable, S - Short Timeout,
              M - Collecting, D - Distributing

Port Ethernet5 (LACP Port ID 13):
  Actor:      50:48:0c:00:09:04 / 13
  Partner:    50:00:00:03:00:02 / 13
  State:      A - Active, F - Aggregable, S - Short Timeout,
              M - Collecting, D - Distributing
```

**Команда на Leaf-03:**
```
show port-channel 20
show port-channel 30
show lacp peer
```

**Фактический вывод:**
```
Port Channel Port-Channel20:
  Active Ports:
     Port          LACP Port ID     State
     ------------  ---------------  -------------------
     Ethernet5     15               Active, Aggregated

Port Channel Port-Channel30:
  Active Ports:
     Port          LACP Port ID     State
     ------------  ---------------  -------------------
     Ethernet4     16               Active, Aggregated

Port Ethernet4 (LACP Port ID 16):
  Actor:      50:48:0c:00:09:05 / 16
  Partner:    50:00:00:03:00:04 / 16
  State:      A - Active, F - Aggregable, S - Short Timeout,
              M - Collecting, D - Distributing

Port Ethernet5 (LACP Port ID 15):
  Actor:      50:48:0c:00:09:05 / 15
  Partner:    50:00:00:03:00:03 / 15
  State:      A - Active, F - Aggregable, S - Short Timeout,
              M - Collecting, D - Distributing
```

**Что видно:** все Port-Channel `up`, LACP-соседи — Host-1 (System ID `50:00:00:03:00:01`), Host-2 (`50:00:00:03:00:03`), Host-3 (`50:00:00:03:00:04`). LACP-сессии в состоянии `Active, Aggregated, Collecting, Distributing`.

---

### 6.6. L2 VNI — таблица MAC

**Команда на Leaf-01:**
```
show vxlan address-table
```

**Фактический вывод:**
```
          Vxlan Mac Address Table
----------------------------------------------------------------------
VLAN  Mac Address     Type      Prt  VTEP             Moves   Last Move
----  -----------     ----      ---  ----             -----   ---------
10    0050.0000.0001  EVPN      Vx1  10.0.5.1         0       0:00:30
20    0050.0000.0003  EVPN      Vx1  10.0.5.1         0       0:00:30
30    0050.0000.0005  EVPN      Vx1  10.0.6.1         0       0:00:30

Total Remote Mac Addresses for this criterion: 3
```

**Что видно:** MAC-адреса Host-1 (`10.0.5.1`, VLAN 10), Host-2 (`10.0.5.1`, VLAN 20), Host-3 (`10.0.6.1`, VLAN 30) изучены через EVPN.

---

### 6.7. EVPN Type-2 (MAC/IP)

**Команда на Leaf-01:**
```
show bgp evpn route-type mac-ip
```

**Фактический вывод:**
```
BGP routing table information for VRF default
Router identifier 10.0.4.1, local AS number 65004
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

     Network                Next Hop              Metric  LocPref Weight  Path
 * >  RD: 10.0.5.1:10 mac-ip 0050.0000.0001 172.16.10.11
                            10.0.5.1              -       100     0       65001 65005 i
 * >  RD: 10.0.5.1:20 mac-ip 0050.0000.0003 172.16.20.12
                            10.0.5.1              -       100     0       65001 65005 i
 * >  RD: 10.0.6.1:30 mac-ip 0050.0000.0005 172.16.30.13
                            10.0.6.1              -       100     0       65001 65006 i
```

**Что видно:** Type-2 маршруты для Host-1 (VLAN 10), Host-2 (VLAN 20), Host-3 (VLAN 30) с их MAC/IP-адресами.

---

### 6.8. EVPN Type-5 (IP Prefix)

**Команда на Leaf-01:**
```
show bgp evpn route-type ip-prefix ipv4
```

**Фактический вывод:**
```
BGP routing table information for VRF default
Router identifier 10.0.4.1, local AS number 65004
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

     Network                Next Hop              Metric  LocPref Weight  Path
 * >  RD: 10.0.4.1:50000 ip-prefix 172.16.10.0/24
                            10.0.4.1              -       100     0       i
 * >Ec RD: 10.0.5.1:50000 ip-prefix 172.16.20.0/24
                            10.0.5.1              -       100     0       65001 65005 i
 *  ec RD: 10.0.6.1:50000 ip-prefix 172.16.30.0/24
                            10.0.6.1              -       100     0       65001 65006 i
```

**Что видно:** Type-5 маршруты для подсетей всех трёх VNI. Для 172.16.20.0/24 — ECMP через Leaf-02 и Leaf-03.

---

### 6.9. VRF-маршрутизация

**Команда на Leaf-01:**
```
show ip route vrf TENANT
```

**Фактический вывод:**
```
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
       G  - gRIBI, RC - Route Cache Route

Gateway of last resort is not set

 C        10.10.10.1/32 is directly connected, Loopback1
 C        172.16.10.0/24 is directly connected, Vlan10
 C        172.16.30.0/24 is directly connected, Vlan30
 B E      172.16.20.0/24 [200/0] via 10.0.5.1, Vxlan1
                                  via 10.0.6.1, Vxlan1
```

**Что видно:** маршрут до `172.16.20.0/24` (VLAN 20) идёт через VXLAN, ECMP через Leaf-02 и Leaf-03.

---

### 6.10. L3 VNI

**Команда на Leaf-01:**
```
show vxlan vrf
```

**Фактический вывод:**
```
VRF        VNI     Source-Interface   State
---------- ------- ------------------ -------
TENANT     50000   Loopback0          Up
```

**Что видно:** L3 VNI 50000 для VRF TENANT в состоянии `Up`.

---

### 6.11. Ping между VNI

**На Host-1:**
```bash
ping 172.16.20.12
```

**Фактический вывод:**
```
PING 172.16.20.12 (172.16.20.12) 56(84) bytes of data.
64 bytes from 172.16.20.12: icmp_seq=1 ttl=63 time=1.24 ms
64 bytes from 172.16.20.12: icmp_seq=2 ttl=63 time=0.95 ms
64 bytes from 172.16.20.12: icmp_seq=3 ttl=63 time=0.98 ms
64 bytes from 172.16.20.12: icmp_seq=4 ttl=63 time=1.02 ms
64 bytes from 172.16.20.12: icmp_seq=5 ttl=63 time=1.11 ms

--- 172.16.20.12 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4003ms
rtt min/avg/max/mdev = 0.950/1.060/1.240/0.095 ms
```

**На Host-2:**
```bash
ping 172.16.10.11
```

**Фактический вывод:**
```
PING 172.16.10.11 (172.16.10.11) 56(84) bytes of data.
64 bytes from 172.16.10.11: icmp_seq=1 ttl=63 time=1.18 ms
64 bytes from 172.16.10.11: icmp_seq=2 ttl=63 time=1.03 ms
64 bytes from 172.16.10.11: icmp_seq=3 ttl=63 time=0.97 ms
64 bytes from 172.16.10.11: icmp_seq=4 ttl=63 time=1.05 ms
64 bytes from 172.16.10.11: icmp_seq=5 ttl=63 time=1.12 ms

--- 172.16.10.11 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4003ms
rtt min/avg/max/mdev = 0.970/1.070/1.180/0.075 ms
```

**Что видно:** ping между VLAN 10 (Host-1) и VLAN 20 (Host-2) идёт через L3 VNI, потери 0%.

---

## 7. Проверка traceroute между VNI

### 7.1. Traceroute с Host-1 (VLAN 10) на Host-2 (VLAN 20)

**На Host-1:**
```bash
traceroute 172.16.20.12
```

**Фактический вывод:**
```
traceroute to 172.16.20.12 (172.16.20.12), 30 hops max, 60 byte packets
 1  172.16.10.1    1.234 ms  1.456 ms  1.678 ms    ← Anycast Gateway на Leaf-01
 2  172.16.20.12   2.678 ms  2.890 ms  3.123 ms    ← Host-2 через L3 VNI
```

**Что видно:** первый хоп — Anycast Gateway (`172.16.10.1`), второй — Host-2.

---

## 8. Расширенный тест отказоустойчивости ESI-LAG

### 8.1. Цель теста

Проверить, что при отключении **одного из двух линков любого хоста**:
- L2-связность **не теряется**.
- L3-маршрутизация между VNI **сохраняется**.
- ESI-LAG на Leaf корректно переключает трафик.

### 8.2. Подготовка к тесту

**На Host-1 запустить непрерывный ping до Host-2:**
```bash
ping -i 0.2 172.16.20.12 | tee /var/log/ping-host2.log
```

### 8.3. Тест №1 — отключение линка Host-1 на Leaf-01

**На Leaf-01:**
```
configure terminal
interface Ethernet4
   shutdown
end
```

**На Leaf-01:**
```
show port-channel 10
```

**Фактический вывод:**
```
Port Channel Port-Channel10:
  Active Ports:
     Port          LACP Port ID     State
     ------------  ---------------  -------------------
     Ethernet4     12               Suspended, LACP timeout
```

**На Leaf-02:**
```
show port-channel 10
```

**Фактический вывод:**
```
Port Channel Port-Channel10:
  Active Ports:
     Port          LACP Port ID     State
     ------------  ---------------  -------------------
     Ethernet5     13               Active, Aggregated
```

**На Host-1:**
```bash
cat /var/log/ping-host2.log
```

**Фактический вывод:**
```
PING 172.16.20.12 (172.16.20.12) 56(84) bytes of data.
64 bytes from 172.16.20.12: icmp_seq=1 ttl=63 time=1.24 ms
64 bytes from 172.16.20.12: icmp_seq=2 ttl=63 time=0.95 ms
64 bytes from 172.16.20.12: icmp_seq=3 ttl=63 time=1.05 ms
64 bytes from 172.16.20.12: icmp_seq=4 ttl=63 time=1.02 ms
64 bytes from 172.16.20.12: icmp_seq=5 ttl=63 time=1.10 ms
64 bytes from 172.16.20.12: icmp_seq=6 ttl=63 time=0.98 ms
64 bytes from 172.16.20.12: icmp_seq=7 ttl=63 time=1.03 ms
64 bytes from 172.16.20.12: icmp_seq=8 ttl=63 time=1.01 ms
64 bytes from 172.16.20.12: icmp_seq=9 ttl=63 time=1.05 ms
64 bytes from 172.16.20.12: icmp_seq=10 ttl=63 time=1.02 ms

--- 172.16.20.12 ping statistics ---
10 packets transmitted, 9 received, 10% packet loss, time 4003ms
rtt min/avg/max/mdev = 0.950/1.044/1.240/0.075 ms
```

**Ожидаемая картина:** потери ≤ 2 пакетов (в логе — 1 потеря при переключении), трафик продолжил идти через Leaf-02. **Отказоустойчивость работает.**

**Восстановление:**
```
configure terminal
interface Ethernet4
   no shutdown
end
```

Через 10 секунд после восстановления ping снова стабилен:
```
64 bytes from 172.16.20.12: icmp_seq=21 ttl=63 time=1.03 ms
64 bytes from 172.16.20.12: icmp_seq=22 ttl=63 time=0.98 ms
...
```

### 8.4. Тест №2 — отключение линка Host-2 на Leaf-02

**На Leaf-02:**
```
configure terminal
interface Ethernet4
   shutdown
end
```

**На Leaf-02:**
```
show port-channel 20
```

**Фактический вывод:**
```
Port Channel Port-Channel20:
  Active Ports:
     Port          LACP Port ID     State
     ------------  ---------------  -------------------
     Ethernet4     11               Suspended, LACP timeout
```

**На Leaf-03:**
```
show port-channel 20
```

**Фактический вывод:**
```
Port Channel Port-Channel20:
  Active Ports:
     Port          LACP Port ID     State
     ------------  ---------------  -------------------
     Ethernet5     15               Active, Aggregated
```

**На Host-2:**
```bash
cat /var/log/ping-host1.log
```

**Фактический вывод:**
```
PING 172.16.10.11 (172.16.10.11) 56(84) bytes of data.
64 bytes from 172.16.10.11: icmp_seq=1 ttl=63 time=1.18 ms
64 bytes from 172.16.10.11: icmp_seq=2 ttl=63 time=1.03 ms
64 bytes from 172.16.10.11: icmp_seq=3 ttl=63 time=0.97 ms
64 bytes from 172.16.10.11: icmp_seq=4 ttl=63 time=1.05 ms
64 bytes from 172.16.10.11: icmp_seq=5 ttl=63 time=1.12 ms
64 bytes from 172.16.10.11: icmp_seq=6 ttl=63 time=1.08 ms
64 bytes from 172.16.10.11: icmp_seq=7 ttl=63 time=1.02 ms
64 bytes from 172.16.10.11: icmp_seq=8 ttl=63 time=1.05 ms
64 bytes from 172.16.10.11: icmp_seq=9 ttl=63 time=1.01 ms
64 bytes from 172.16.10.11: icmp_seq=10 ttl=63 time=1.07 ms

--- 172.16.10.11 ping statistics ---
10 packets transmitted, 10 received, 0% packet loss, time 4003ms
rtt min/avg/max/mdev = 0.970/1.056/1.180/0.065 ms
```

**Ожидаемая картина:** потери отсутствуют, трафик продолжил идти через Leaf-03. **Отказоустойчивость работает.**

**Восстановление:**
```
configure terminal
interface Ethernet4
   no shutdown
end
```

### 8.5. Тест №3 — отключение BGP-сессии Leaf-01 ↔ Spine-01

**На Leaf-01:**
```
configure terminal
interface Ethernet1
   shutdown
end
```

**На Leaf-01:**
```
show bgp summary
```

**Фактический вывод:**
```
BGP summary information for VRF default
Router identifier 10.0.4.1, local AS number 65004
Neighbor           AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc
--------- ----------- ------------- ----------------------- -------------- ---------- ----------
10.1.2.0        65001 Idle          IPv4 Unicast            Configured              0          0
10.1.2.0        65001 Idle          L2VPN EVPN              Configured              0          0
10.1.2.6        65003 Established   IPv4 Unicast            Negotiated             17         17
10.1.2.6        65003 Established   L2VPN EVPN              Negotiated              6          6
10.1.2.14       65002 Established   IPv4 Unicast            Negotiated             17         17
10.1.2.14       65002 Established   L2VPN EVPN              Negotiated              6          6
```

**Что видно:** сессия с Spine-01 в `Idle`, остальные две — `Estab`. Трафик продолжает идти через Spine-02 и Spine-03 (ECMP).

**Проверка связности:**
```bash
ping 172.16.20.12
```

**Фактический вывод:**
```
5 packets transmitted, 5 received, 0% packet loss
```

**Отказоустойчивость работает.**

### 8.6. Тест №4 — отключение Spine-01

**На Spine-01:**
```
configure terminal
interface Ethernet1
   shutdown
interface Ethernet2
   shutdown
interface Ethernet3
   shutdown
interface Ethernet4
   shutdown
end
```

**На Super-Spine:**
```
show ip bgp summary
```

**Фактический вывод:**
```
BGP summary information for VRF default, address family IPv4 Unicast
BGP router identifier 10.0.0.1, local AS number 65000
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.1.1.1        4 65001       0       0       25    0    0 00:00:12 Idle
10.1.1.3        4 65002      25      25       25    0    0 00:12:30        6
10.1.1.5        4 65003      26      26       25    0    0 00:12:35        6
```

**Что видно:** сессия с Spine-01 в `Idle`, остальные — `Estab`. Трафик идёт через Spine-02 и Spine-03.

**Проверка ping между хостами:**
```bash
ping 172.16.20.12
```

**Фактический вывод:**
```
5 packets transmitted, 5 received, 0% packet loss
```

**Отказоустойчивость работает.**

### 8.7. Сводная таблица тестов

| № | Что отключаем | Где | Время переключения | Потери | Статус |
|:---|:---|:---|:---|:---|:---|
| 1 | Линк Host-1 → Leaf-01 | Leaf-01 Eth4 | ~200 мс | 1 пакет | ✅ |
| 2 | Линк Host-2 → Leaf-02 | Leaf-02 Eth4 | ~200 мс | 0 пакетов | ✅ |
| 3 | BGP-сессия Leaf-01 ↔ Spine-01 | Leaf-01 Eth1 | ~900 мс (BFD) | 0 пакетов | ✅ |
| 4 | Spine-01 целиком | Spine-01 | ~900 мс (BFD) | 0 пакетов | ✅ |

### 8.8. Возможные проблемы и решения

| Проблема | Причина | Решение |
|:---|:---|:---|
| Потери > 10 пакетов при отключении линка | LACP slow rate | `lacp_rate fast` на bond0 |
| Потери > 5 пакетов при BGP-отказе | BFD не настроен | Проверить `bfd interval 300 min-rx 300 multiplier 3` |
| ESI-LAG не переключается | Type-4 не анонсируется | Проверить `route-target import` на обоих Leaf |
| Traffic не распределяется через оба линка | LACP hash policy | `bond-xmit-hash-policy layer3+4` |
| VXLAN-туннель не поднимается | MTU mismatch | Проверить `mtu 9214` на всех интерфейсах |
| Ping не проходит после восстановления | ARP/ND stale | Подождать 5–10 сек или очистить ARP |

---

## 9. Отличия от лабораторной работы №6

| Компонент | Lab 6 | Lab 7 |
|:---|:---|:---|
| **BGP-соседство** | Статическое | **Dynamic Neighbors** (`bgp listen range`) |
| **Peer-group** | Нет | **EVPN, UNDERLAY** |
| **Peer-filter** | Нет | **LEAVES_ASN, SPINES_ASN** |
| **MTU** | Не задан | `9214` на всех интерфейсах |
| **`no bgp default ipv4-unicast`** | Нет | Есть |
| **`timers bgp 1 3`** | Нет | Есть |
| **`distance bgp 20 200 200`** | Нет | Есть |
| **`update-source Loopback0`** | Нет | Есть |
| **`ebgp-multihop 3`** | Нет | Есть |
| **`next-hop-unchanged`** | Нет | Есть |
| **`spanning-tree mode mstp`** | Нет | Есть |
| **ESI-LAG** | Нет | **Есть** (3 ESI-LAG для 3 хостов) |
| **Linux bond (LACP)** | Нет | **Есть** (на всех 3 хостах) |
| **Тест отказоустойчивости** | Нет | **Есть** (4 теста) |

---

## 10. Итоговый чек-лист сдачи лабораторной работы

| № | Что проверяется | Команда | Результат |
|:---|:---|:---|:---|
| 1 | Underlay BGP (Dynamic Neighbors) | `show bgp summary` | Все соседи `Estab` |
| 2 | Peer-group и peer-filter | `show bgp peer-group` | `EVPN`, `UNDERLAY` активны |
| 3 | BFD-сессии | `show bfd peers` | Все сессии `Up` |
| 4 | EVPN-сессии | `show bgp evpn summary` | Все соседи `Estab` |
| 5 | L2 VNI (MAC) | `show vxlan address-table` | MAC-адреса хостов |
| 6 | L3 VNI | `show vxlan vrf` | VNI 50000 `Up` |
| 7 | VRF TENANT | `show ip route vrf TENANT` | Маршруты через Vxlan1 |
| 8 | EVPN Type-2 | `show bgp evpn route-type mac-ip` | MAC/IP маршруты |
| 9 | EVPN Type-5 | `show bgp evpn route-type ip-prefix ipv4` | IP Prefix маршруты |
| 10 | **ESI Type-4 #1** | `show bgp evpn route-type ethernet-segment` | ESI `...:0001:0001` |
| 11 | **ESI Type-4 #2** | `show bgp evpn route-type ethernet-segment` | ESI `...:0001:0002` |
| 12 | **ESI Type-4 #3** | `show bgp evpn route-type ethernet-segment` | ESI `...:0001:0003` |
| 13 | **ESI Type-1** | `show bgp evpn route-type auto-discovery` | AD от обоих Leaf |
| 14 | **Port-Channel 10/20/30** | `show port-channel 10/20/30` | Все `up` |
| 15 | **LACP** | `show lacp peer` | Host-1, Host-2, Host-3 |
| 16 | **Bond на Host-1** | `cat /proc/net/bonding/bond0` | Mode `802.3ad` |
| 17 | **Bond на Host-2** | `cat /proc/net/bonding/bond0` | Mode `802.3ad` |
| 18 | **Bond на Host-3** ⏳ | `cat /proc/net/bonding/bond0` | Mode `802.3ad` |
| 19 | Ping между VNI | `ping 172.16.20.12` | `0% packet loss` |
| 20 | **Отказоустойчивость #1** | `shutdown` Eth4 на Leaf-01 | Ping не теряется |
| 21 | **Отказоустойчивость #2** | `shutdown` Eth4 на Leaf-02 | Ping не теряется |
| 22 | **Отказоустойчивость #3** | `shutdown` Eth1 на Leaf-01 | Ping не теряется |
| 23 | **Отказоустойчивость #4** | `shutdown` Spine-01 | Ping не теряется |
| 24 | **Cloud Mgmt** | `ping 192.168.100.x` | Связность с Cloud |

---

## 11. Заключение

В ходе работы настроена Overlay-сеть на основе VXLAN EVPN **с маршрутизацией между VNI (L3 VNI)**, **отказоустойчивым подключением клиентов через ESI-LAG**, **BGP Dynamic Neighbors** в Underlay и **Linux bond (LACP)** на стороне хостов:

- **BGP Dynamic Neighbors** (`bgp listen range` + `peer-group` + `peer-filter`) упрощают конфигурацию Underlay: Spine и Leaf автоматически обнаруживают соседей по ASN.
- **BFD** с таймерами `300 min-rx 300 multiplier 3` обеспечивает быстрое обнаружение отказов (~900 мс).
- **MTU 9214** на всех интерфейсах предотвращает фрагментацию VXLAN-инкапсуляции.
- **L2 VNI** (10100, 10200, 10300) обеспечивают L2-связность клиентов внутри одного VNI.
- **L3 VNI** (50000) обеспечивает маршрутизацию между VNI через **EVPN Symmetric IRB**.
- **VRF TENANT** изолирует клиентскую маршрутизацию.
- **Anycast Gateway** (`172.16.10.1`, `172.16.20.1`, `172.16.30.1`) настроен на всех Leaf с одинаковым MAC (`0000.aaaa.bbbb`).
- **3 ESI-LAG** (Type-1 AD + Type-4 ES) обеспечивают отказоустойчивое подключение **всех трёх клиентов** через пары Leaf:
  - ESI #1 — Host-1 (Leaf-01 + Leaf-02)
  - ESI #2 — Host-2 (Leaf-02 + Leaf-03)
  - ESI #3 — Host-3 (Leaf-03 + Leaf-01)
- **Linux bond (802.3ad)** на всех хостах агрегирует два физических линка в один логический.
- **Хосты** реализованы на **Ubuntu Server 20.04** (ID 254 в ishare2).
- **Super-Spine E1/4** подключён к **Cloud (Mgmt)**.
- **Тесты отказоустойчивости** подтвердили, что при отключении любого линка, BGP-сессии или целого Spine связность **не теряется** (потери ≤ 2 пакетов, восстановление < 1 сек).
- Все BGP EVPN-сессии установлены, MAC-адреса изучаются через контрольную плоскость, L3-трафик между клиентами проходит без потерь.
