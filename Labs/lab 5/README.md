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
router bgp 65000
  router-id 10.0.0.1
  !
  address-family l2vpn evpn
    retain route-target all
  !
  neighbor 10.1.1.1 remote-as 65001
    address-family l2vpn evpn
      route-reflector-client
  !
  neighbor 10.1.1.3 remote-as 65002
    address-family l2vpn evpn
      route-reflector-client
  !
  neighbor 10.1.1.5 remote-as 65003
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
router bgp 65001
  router-id 10.0.1.1
  !
  address-family evpn
    neighbor 10.1.2.1 activate
    neighbor 10.1.2.1 route-reflector-client
    neighbor 10.1.2.3 activate
    neighbor 10.1.2.3 route-reflector-client
    neighbor 10.1.2.5 activate
    neighbor 10.1.2.5 route-reflector-client
```

```
Spine-02 (AS 65002)
text
hostname Spine-02
!
router bgp 65002
  router-id 10.0.2.1
  !
  address-family evpn
    neighbor 10.1.2.7 activate
    neighbor 10.1.2.7 route-reflector-client
    neighbor 10.1.2.9 activate
    neighbor 10.1.2.9 route-reflector-client
    neighbor 10.1.2.11 activate
    neighbor 10.1.2.11 route-reflector-client
```
```
Spine-03 (AS 65003)

text
hostname Spine-03
!
router bgp 65003
  router-id 10.0.3.1
  !
  address-family evpn
    neighbor 10.1.2.13 activate
    neighbor 10.1.2.13 route-reflector-client
    neighbor 10.1.2.15 activate
    neighbor 10.1.2.15 route-reflector-client
    neighbor 10.1.2.17 activate
    neighbor 10.1.2.17 route-reflector-client
```
### 4.3. Конфигурация Leaf (Arista vEOS)
На каждом Leaf настраивается VXLAN, VLAN, Anycast Gateway и EVPN. Приведём полную конфигурацию для Leaf-01, для Leaf-02 и Leaf-03 меняются только номера AS,
Loopback-адреса и IP-адреса соседей (они указаны в таблице 3.1).
```
Leaf-01 (AS 65004, Loopback 10.0.4.1)
text
hostname Leaf-01
!
vlan 10
   name RED_ZONE
!
interface Ethernet3
   description Host-1
   switchport access vlan 10
   no shutdown
!
interface Vlan10
   description Gateway for VLAN 10
   ip address virtual 172.16.10.1/24
   no shutdown
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10100
!
router bgp 65004
  router-id 10.0.4.1
  !
  address-family evpn
    neighbor 10.1.2.0 activate   ! Spine-01
    neighbor 10.1.2.6 activate   ! Spine-02
    neighbor 10.1.2.12 activate  ! Spine-03
!
evpn
  vni 10100 l2
    rd auto
    route-target import auto
    route-target export auto
```
```
Leaf-02 (AS 65005, Loopback 10.0.5.1)
hostname Leaf-02
!
vlan 10
   name RED_ZONE
!
interface Ethernet3
   description Host-2
   switchport access vlan 10
   no shutdown
!
interface Vlan10
   description Gateway for VLAN 10
   ip address virtual 172.16.10.1/24
   no shutdown
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10100
!
router bgp 65005
  router-id 10.0.5.1
  !
  address-family evpn
    neighbor 10.1.2.2 activate   ! Spine-01
    neighbor 10.1.2.8 activate   ! Spine-02
    neighbor 10.1.2.14 activate  ! Spine-03
!
evpn
  vni 10100 l2
    rd auto
    route-target import auto
    route-target export auto
```
```    
Leaf-03 (AS 65006, Loopback 10.0.6.1)

text
hostname Leaf-03
!
vlan 10
   name RED_ZONE
!
interface Ethernet3
   description Host-3
   switchport access vlan 10
   no shutdown
!
interface Vlan10
   description Gateway for VLAN 10
   ip address virtual 172.16.10.1/24
   no shutdown
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10100
!
router bgp 65006
  router-id 10.0.6.1
  !
  address-family evpn
    neighbor 10.1.2.4 activate   ! Spine-01
    neighbor 10.1.2.10 activate  ! Spine-02
    neighbor 10.1.2.16 activate  ! Spine-03
!
evpn
  vni 10100 l2
    rd auto
    route-target import auto
    route-target export auto
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
Пояснение полей:
```
Параметр	Описание
Neighbor	IP-адрес соседа (Spine)
AS	Номер AS соседа
MsgRcvd / MsgSent	Количество полученных/отправленных BGP-сообщений
Up/Down	Время активности сессии
State	Должно быть Estab (установлена)
PfxRcd	Количество полученных EVPN-маршрутов (как минимум 2 – MAC+IP)

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
Пояснение:

Параметр	Описание
VLAN	Локальный VLAN
VNI	VXLAN-идентификатор
MAC Address	MAC-адрес клиента на удалённом Leaf
Type	EVPN – изучено через контрольную плоскость
Remote VTEP	IP-адрес удалённого VTEP (Leaf)
```
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
В ходе работы настроена Overlay-сеть на основе VXLAN EVPN для L2-связанности клиентов:Использована существующая Underlay-сеть с eBGP, BFD и MD5.
На Spine настроены Route Reflector'ы для адресного семейства EVPN.На Leaf созданы VTEP, L2 VNI (10100), VLAN 10 и Anycast Gateway (172.16.10.1).
Клиентские порты (Eth3) переведены в access-режим.Проверена связность между хостами, подключёнными к разным Leaf, через VXLAN-туннели.
Все BGP EVPN-сессии установлены, MAC-адреса изучаются через контрольную плоскость, L2-трафик между клиентами проходит без потерь.
