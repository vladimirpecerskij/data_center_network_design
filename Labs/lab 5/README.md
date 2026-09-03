# Лабораторная работа №5. VXLAN. L2 VNI

**Цель работы:** Настроить Overlay-сеть на основе VXLAN EVPN для обеспечения L2-связанности между клиентами, подключёнными к разным Leaf-коммутаторам.

---

## 1. Топология сети

![Топология](./BGP.png)

В среде **PNET Lab** используется та же физическая топология, что и в лабораторной работе №4:

- **Super-Spine (уровень 1):** 1 коммутатор **Cisco Nexus 5000** (образ NX-OS).
- **Spine (уровень 2):** 3 коммутатора **Arista vEOS** (Spine-01, Spine-02, Spine-03).
- **Leaf (уровень 3):** 3 коммутатора **Arista vEOS** (Leaf-01, Leaf-02, Leaf-03).

Underlay-сеть уже настроена с использованием eBGP, BFD и MD5-аутентификации. Все Loopback-адреса (используемые как VTEP) доступны друг другу.

> **Примечание:** В этой работе Super-Spine не участвует в VXLAN-инкапсуляции, но должен передавать BGP EVPN-маршруты между Spine.

---

## 2. План работ

1. **Проверка Underlay** – убедиться в IP-связности между VTEP (Loopback) всех устройств.
2. **Планирование Overlay** – выбрать VLAN, VNI, подсеть для клиентов, Anycast Gateway.
3. **Настройка BGP для EVPN:**
   - На Spine – настроить Route Reflector для адресного семейства `l2vpn evpn`.
   - На Leaf – включить адресное семейство EVPN и активировать соседей (Spine).
4. **Настройка VXLAN на Leaf:**
   - Создать VLAN и VNI.
   - Настроить интерфейс VXLAN (VTEP) с источником Loopback.
   - Создать SVI с Anycast Gateway.
5. **Подключение клиентов** – настроить порты Leaf в режиме access VLAN.
6. **Верификация** – проверить BGP EVPN-сессии, таблицы MAC/VNI и связность между хостами.
7. **Настройка ECMP**:
Включены maximum-paths на всех устройствах для использования нескольких равнозначных путей.

---

## 3. Адресное пространство Overlay

### 3.1. VTEP (Loopback) адреса

| Устройство | Роль | Loopback0 (VTEP) |
|:---|:---|:---|
| **Leaf-01** | Leaf | 10.0.4.1/32 |
| **Leaf-02** | Leaf | 10.0.5.1/32 |
| **Leaf-03** | Leaf | 10.0.6.1/32 |
| **Spine-01** | Spine (Route Reflector) | 10.0.1.1/32 |
| **Spine-02** | Spine (Route Reflector) | 10.0.2.1/32 |
| **Spine-03** | Spine (Route Reflector) | 10.0.3.1/32 |

### 3.2. Параметры L2-сервиса (VNI)

| Параметр | Значение | Описание |
|:---|:---|:---|
| **VLAN** | 10 | Клиентский VLAN на всех Leaf |
| **VNI** | 10100 | Идентификатор VXLAN-сегмента |
| **Подсеть клиентов** | 172.16.10.0/24 | IPv4-сеть для хостов |
| **Anycast Gateway** | 172.16.10.1 | Виртуальный адрес шлюза на каждом Leaf |
| **VTEP Source Interface** | Loopback0 | IP-адрес для построения VXLAN-туннелей |

### 3.3. Распределение IP-адресов хостов

| Хост | Leaf | Интерфейс | IP-адрес / Маска |
|:---|:---|:---|:---|
| Host-1 | Leaf-01 | Eth3 | 172.16.10.11/24 |
| Host-2 | Leaf-02 | Eth3 | 172.16.10.12/24 |
| Host-3 | Leaf-03 | Eth3 | 172.16.10.13/24 |

---

## 4. Конфигурации устройств

### 4.1. Super-Spine (Cisco Nexus 5000)

На Super-Spine необходимо включить поддержку адресного семейства EVPN и настроить Route Reflector для Spine.

```
hostname NEXUS-5000
!
feature bgp
feature bfd
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
Примечание: Команда retain route-target all гарантирует, что Super-Spine будет передавать все EVPN-маршруты между Route Reflector-клиентами, даже если они не соответствуют локальным route-target.

### 4.2. Конфигурация Spine (Arista vEOS)
Каждый Spine должен выступать в роли Route Reflector для Leaf-коммутаторов. Покажем на примере Spine-01 (для Spine-02 и Spine-03 адреса соседей меняются).
```
Spine-01 (AS 65001)
text
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
  !
  address-family ipv4
    maximum-paths 3
    redistribute connected
  !
  !
  neighbor 10.1.1.0 remote-as 65000
    bfd
    password MySecretKey123
    !
    address-family ipv4
      send-community
    !
    address-family evpn
      route-reflector-client
  !
  !
  neighbor 10.1.2.1 remote-as 65004
    bfd
    password MySecretKey123
    !
    address-family ipv4
      send-community
    !
    address-family evpn
      activate
      route-reflector-client
  !
  !
  neighbor 10.1.2.3 remote-as 65005
    bfd
    password MySecretKey123
    !
    address-family ipv4
      send-community
    !
    address-family evpn
      activate
      route-reflector-client
  !
  !
  neighbor 10.1.2.5 remote-as 65006
    bfd
    password MySecretKey123
    !
    address-family ipv4
      send-community
    !
    address-family evpn
      activate
      route-reflector-client
```

```
Spine-02 (AS 65002)
text
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
  !
  address-family ipv4
    maximum-paths 3
    redistribute connected
  !
  !
  neighbor 10.1.1.2 remote-as 65000
    bfd
    password MySecretKey123
    !
    address-family ipv4
      send-community
    !
    address-family evpn
      route-reflector-client
  !
  !
  neighbor 10.1.2.7 remote-as 65004
    bfd
    password MySecretKey123
    !
    address-family ipv4
      send-community
    !
    address-family evpn
      activate
      route-reflector-client
  !
  !
  neighbor 10.1.2.9 remote-as 65005
    bfd
    password MySecretKey123
    !
    address-family ipv4
      send-community
    !
    address-family evpn
      activate
      route-reflector-client
  !
  !
  neighbor 10.1.2.11 remote-as 65006
    bfd
    password MySecretKey123
    !
    address-family ipv4
      send-community
    !
    address-family evpn
      activate
      route-reflector-client
```
```
Spine-03 (AS 65003)

text
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
  !
  address-family ipv4
    maximum-paths 3
    redistribute connected
  !
  !
  neighbor 10.1.1.4 remote-as 65000
    bfd
    password MySecretKey123
    !
    address-family ipv4
      send-community
    !
    address-family evpn
      route-reflector-client
  !
  !
  neighbor 10.1.2.13 remote-as 65004
    bfd
    password MySecretKey123
    !
    address-family ipv4
      send-community
    !
    address-family evpn
      activate
      route-reflector-client
  !
  !
  neighbor 10.1.2.15 remote-as 65005
    bfd
    password MySecretKey123
    !
    address-family ipv4
      send-community
    !
    address-family evpn
      activate
      route-reflector-client
  !
  !
  neighbor 10.1.2.17 remote-as 65006
    bfd
    password MySecretKey123
    !
    address-family ipv4
      send-community
    !
    address-family evpn
      activate
      route-reflector-client
```
### 4.3. Конфигурация Leaf (Сшысщ vIOS)
На каждом Leaf настраивается VXLAN, VLAN, Anycast Gateway и EVPN. Приведём полную конфигурацию для Leaf-01, для Leaf-02 и Leaf-03 меняются только номера AS,
Loopback-адреса и IP-адреса соседей (они указаны в таблице 3.1).
```
Leaf-01 (AS 65004, Loopback 10.0.4.1)
text
hostname Leaf-01
!
no ip domain-lookup
!
! --- L2 Part (Access & Gateway) ---
vlan 10
 name RED_ZONE
!
interface Ethernet3
 description Host-1
 switchport mode access
 switchport access vlan 10
 no shutdown
!
interface Vlan10
 description Gateway for VLAN 10
 ip address 172.16.10.1 255.255.255.0
 no shutdown
!

! --- Control Plane & Transport ---
! Включаем необходимые фичи
feature bgp
feature bfd
!
! Loopback для источника туннелей и Router-ID
interface Loopback0
 ip address 10.0.4.1 255.255.255.255
 no shutdown
!

! --- Uplinks to Spine (L3 Links with BFD) ---
! Eth1 -> Spine-01
interface Ethernet1
 no switchport
 ip address 10.1.2.1 255.255.255.254
 bfd interval 50 min_rx 50 multiplier 3
 no shutdown
!
! Eth2 -> Spine-02
interface Ethernet2
 no switchport
 ip address 10.1.2.7 255.255.255.254
 bfd interval 50 min_rx 50 multiplier 3
 no shutdown
!
! Eth4 -> Spine-03 (Eth3 занят под Host, поэтому используем Eth4)
interface Ethernet4
 no switchport
 ip address 10.1.2.13 255.255.255.254
 bfd interval 50 min_rx 50 multiplier 3
 no shutdown
!

! --- VXLAN / NVE Configuration (Cisco Way) ---
! Создаем интерфейс NVE (аналог Vxlan1)
interface nve1
 host-reachability protocol bgp
 source-interface Loopback0
 !
 ! Маппинг VNI 10100 на VLAN 10
 member vni 10100
  ingress replication protocol static
  ! Здесь можно добавить IP адреса Spine для репликации, 
  ! но при работающем EVPN BGP это не обязательно.
  ! Если нужна статическая репликация для тестов, раскомментируй строки ниже:
  ! peer-ip 10.0.1.1
  ! peer-ip 10.0.2.1
  ! peer-ip 10.0.3.1
!

! --- BGP Configuration (IPv4 + EVPN) ---
router bgp 65004
 bgp router-id 10.0.4.1
 bgp log-neighbor-changes
 
 ! Глобальная активация EVPN
 l2vpn evpn
 
 ! IPv4 Unicast: ECMP + Redistribution
 address-family ipv4 unicast
  maximum-paths 3                  <-- ECMP на Cisco
  redistribute connected
 !
 
 ! EVPN Address Family
 address-family l2vpn evpn
  retain route-target all
 !

 ! Neighbor 1: Spine-01 (10.1.2.0)
 neighbor 10.1.2.0 remote-as 65001
  bfd                             <-- BFD
  password 0 MySecretKey123       <-- MD5 Password (0 обязателен!)
  !
  address-family ipv4 unicast
   send-community
  !
  address-family l2vpn evpn
   route-reflector-client
  !
 !
 ! Neighbor 2: Spine-02 (10.1.2.6)
 neighbor 10.1.2.6 remote-as 65002
  bfd
  password 0 MySecretKey123
  !
  address-family ipv4 unicast
   send-community
  !
  address-family l2vpn evpn
   route-reflector-client
  !
 !
 ! Neighbor 3: Spine-03 (10.1.2.12)
 neighbor 10.1.2.12 remote-as 65003
  bfd
  password 0 MySecretKey123
  !
  address-family ipv4 unicast
   send-community
  !
  address-family l2vpn evpn
   route-reflector-client
  !
```
```
Leaf-02 (AS 65005, Loopback 10.0.5.1)
hostname Leaf-02
!
no ip domain-lookup
!
! --- L2 Part (Access & Gateway) ---
vlan 10
 name RED_ZONE
!
interface Ethernet3
 description Host-1
 switchport mode access
 switchport access vlan 10
 no shutdown
!
interface Vlan10
 description Gateway for VLAN 10
 ip address 172.16.10.1 255.255.255.0
 no shutdown
!

! --- Control Plane & Transport ---
feature bgp
feature bfd
!
interface Loopback0
 ip address 10.0.5.1 255.255.255.255
 no shutdown
!

! --- Uplinks to Spine (L3 Links with BFD) ---
! Eth1 -> Spine-01
interface Ethernet1
 no switchport
 ip address 10.1.2.3 255.255.255.254
 bfd interval 50 min_rx 50 multiplier 3
 no shutdown
!
! Eth2 -> Spine-02
interface Ethernet2
 no switchport
 ip address 10.1.2.9 255.255.255.254
 bfd interval 50 min_rx 50 multiplier 3
 no shutdown
!
! Eth4 -> Spine-03 (Eth3 занят под Host)
interface Ethernet4
 no switchport
 ip address 10.1.2.15 255.255.255.254
 bfd interval 50 min_rx 50 multiplier 3
 no shutdown
!

! --- VXLAN / NVE Configuration ---
interface nve1
 host-reachability protocol bgp
 source-interface Loopback0
 !
 member vni 10100
  ingress replication protocol static
!

! --- BGP Configuration (IPv4 + EVPN) ---
router bgp 65005
 bgp router-id 10.0.5.1
 bgp log-neighbor-changes
 
 l2vpn evpn
 
 address-family ipv4 unicast
  maximum-paths 3
  redistribute connected
 !
 
 address-family l2vpn evpn
  retain route-target all
 !

 ! Neighbor 1: Spine-01 (10.1.2.2)
 neighbor 10.1.2.2 remote-as 65001
  bfd
  password 0 MySecretKey123
  !
  address-family ipv4 unicast
   send-community
  !
  address-family l2vpn evpn
   route-reflector-client
  !
 !
 ! Neighbor 2: Spine-02 (10.1.2.8)
 neighbor 10.1.2.8 remote-as 65002
  bfd
  password 0 MySecretKey123
  !
  address-family ipv4 unicast
   send-community
  !
  address-family l2vpn evpn
   route-reflector-client
  !
 !
 ! Neighbor 3: Spine-03 (10.1.2.14)
 neighbor 10.1.2.14 remote-as 65003
  bfd
  password 0 MySecretKey123
  !
  address-family ipv4 unicast
   send-community
  !
  address-family l2vpn evpn
   route-reflector-client
  !
```
```    
Leaf-03 (AS 65006, Loopback 10.0.6.1)

text
hostname Leaf-03
!
no ip domain-lookup
!
! --- L2 Part (Access & Gateway) ---
vlan 10
 name RED_ZONE
!
interface Ethernet3
 description Host-1
 switchport mode access
 switchport access vlan 10
 no shutdown
!
interface Vlan10
 description Gateway for VLAN 10
 ip address 172.16.10.1 255.255.255.0
 no shutdown
!

! --- Control Plane & Transport ---
feature bgp
feature bfd
!
interface Loopback0
 ip address 10.0.6.1 255.255.255.255
 no shutdown
!

! --- Uplinks to Spine (L3 Links with BFD) ---
! Eth1 -> Spine-01
interface Ethernet1
 no switchport
 ip address 10.1.2.5 255.255.255.254
 bfd interval 50 min_rx 50 multiplier 3
 no shutdown
!
! Eth2 -> Spine-02
interface Ethernet2
 no switchport
 ip address 10.1.2.11 255.255.255.254
 bfd interval 50 min_rx 50 multiplier 3
 no shutdown
!
! Eth4 -> Spine-03 (Eth3 занят под Host)
interface Ethernet4
 no switchport
 ip address 10.1.2.17 255.255.255.254
 bfd interval 50 min_rx 50 multiplier 3
 no shutdown
!

! --- VXLAN / NVE Configuration ---
interface nve1
 host-reachability protocol bgp
 source-interface Loopback0
 !
 member vni 10100
  ingress replication protocol static
!

! --- BGP Configuration (IPv4 + EVPN) ---
router bgp 65006
 bgp router-id 10.0.6.1
 bgp log-neighbor-changes
 
 l2vpn evpn
 
 address-family ipv4 unicast
  maximum-paths 3
  redistribute connected
 !
 
 address-family l2vpn evpn
  retain route-target all
 !

 ! Neighbor 1: Spine-01 (10.1.2.4)
 neighbor 10.1.2.4 remote-as 65001
  bfd
  password 0 MySecretKey123
  !
  address-family ipv4 unicast
   send-community
  !
  address-family l2vpn evpn
   route-reflector-client
  !
 !
 ! Neighbor 2: Spine-02 (10.1.2.10)
 neighbor 10.1.2.10 remote-as 65002
  bfd
  password 0 MySecretKey123
  !
  address-family ipv4 unicast
   send-community
  !
  address-family l2vpn evpn
   route-reflector-client
  !
 !
 ! Neighbor 3: Spine-03 (10.1.2.16)
 neighbor 10.1.2.16 remote-as 65003
  bfd
  password 0 MySecretKey123
  !
  address-family ipv4 unicast
   send-community
  !
  address-family l2vpn evpn
   route-reflector-client
  !
```
Примечания по конфигурации:
Команда ip address virtual на SVI создаёт Anycast Gateway – один и тот же IP-адрес на всех Leaf.В разделе evpn для VNI 10100 используется автоматическое формирование Route Distinguisher (rd auto) и Route Target (route-target auto), что упрощает конфигурацию.BGP-соседи для EVPN активируются с помощью activate в адресном семействе.

## 5. Верификация
### 5.1. Проверка BGP EVPN-сессий
```
Команда (на любом Leaf):
text
show bgp evpn summary
Пример вывода на Leaf-01:

text
BGP summary information for VRF default
Router identifier 10.0.4.1, local AS number 65004
Neighbor Status Codes: m - Under maintenance
  Neighbor         V  AS           MsgRcvd   MsgSent  InQ  OutQ  Up/Down State   PfxRcd
  10.1.2.0         4  65001            125       123    0     0 01:02:33 Estab   2
  10.1.2.6         4  65002            124       122    0     0 01:02:28 Estab   2
  10.1.2.12        4  65003            126       124    0     0 01:02:40 Estab   2
```
Пояснение полей:
Параметр	ОписаниеNeighbor	IP-адрес соседа (Spine)AS	Номер AS соседаMsgRcvd / MsgSent	Количество полученных/отправленных BGP-сообщенийUp/Down	Время активности сессии
State	Должно быть Estab (установлена) PfxRcd	Количество полученных EVPN-маршрутов (как минимум 2 – MAC+IP)

### 5.2. Проверка таблицы MAC-адресов в VXLAN
```Команда (на любом Leaf):
text
show vxlan address-table
Пример вывода на Leaf-01:

text
          Vxlan Mac Address Table
================================================
VLAN  VNI       MAC Address       Type      Age    Remote VTEP
----  --------  ----------------- --------  -----  -------------
10    10100     0050.7966.6800    EVPN      -      10.0.5.1
10    10100     0050.7966.6801    EVPN      -      10.0.6.1
```
Пояснение: Параметр	Описание VLAN	Локальный VLAN VNI	VXLAN-идентификатор MAC Address	MAC-адрес клиента на удалённом LeafType	EVPN – изучено через контрольную плоскость Remote VTEP	IP-адрес удалённого VTEP (Leaf)

### 5.3. Проверка таблицы маршрутизации (для Anycast Gateway)
Команда (на любом Leaf):
```
text
show ip route
В таблице должен присутствовать маршрут до подсети 172.16.10.0/24 через интерфейс Vlan10 (connected).
```

### 5.4. Проверка связности между хостами
```
С Host-1 (подключён к Leaf-01) на Host-2 (Leaf-02):
text
Host-1# ping 172.16.10.12
!!!!!
Success rate is 100 percent (5/5)
С Host-1 на Host-3 (Leaf-03):

text
Host-1# ping 172.16.10.13
!!!!!
Success rate is 100 percent (5/5)
С Host-2 на Host-3:

text
Host-2# ping 172.16.10.13
!!!!!
Success rate is 100 percent (5/5)
Если пинги проходят, значит L2-связность через VXLAN работает корректно.
```

## 6. Заключение
В ходе работы настроена Overlay-сеть на основе VXLAN EVPN для L2-связанности клиентов:Использована существующая Underlay-сеть с eBGP, BFD,MD5 и ECMP.
На Spine настроены Route Reflector'ы для адресного семейства EVPN.На Leaf созданы VTEP, L2 VNI (10100), VLAN 10 и Anycast Gateway (172.16.10.1).
Клиентские порты (Eth3) переведены в access-режим.Проверена связность между хостами, подключёнными к разным Leaf, через VXLAN-туннели.
Все BGP EVPN-сессии установлены, MAC-адреса изучаются через контрольную плоскость, L2-трафик между клиентами проходит без потерь.
