# Лабораторная работа №1. Проектирование адресного пространства фабрики CLOS

**Цель работы:** Проектирование Underlay-сети для двухуровневой фабрики CLOS (Spine-and-Leaf) с использованием эффективного распределения IP-адресов.

---

## 1. Топология сети
Топология собрана в эмуляторе PNETLab. Архитектура построена по классическому принципу Spine-and-Leaf, где каждый Leaf подключен ко всем Spine. 

* **Spine-уровень:** 2 коммутатора Cisco vIOS L3 (`Spine-01`, `Spine-02`).
* **Leaf-уровень:** 2 роутера Cisco vIOS (`Leaf-01`, `Leaf-02`).

![Топология CLOS](pnet_topology.png)

---

## 2. План адресного пространства (Underlay)

Для построения Underlay-инфраструктуры дата-центра используется пул приватных адресов `10.0.0.0/8`. 

* **Loopback-интерфейсы (Router ID / BGP Peering):** Выделена сеть `10.255.0.0/24`. Каждому устройству назначен уникальный адрес с маской `/32`.
* **Стыковочные каналы (Point-to-Point линки):** Выделен общий пул `10.0.0.0/16`. Для экономии адресного пространства внутри этого пула нарезаются подсети с маской `/31`.

### Таблица распределения IP-адресов

| Устройство | Интерфейс | IP-адрес / Маска | Назначение |
| :--- | :--- | :--- | :--- |
| **Spine-01** | Loopback0 | `10.255.0.1/32` | Идентификатор устройства (Router ID) |
| | GigabitEthernet0/0 | `10.0.1.0/31` | p2p-линк к Leaf-01 (Gi0/0) |
| | GigabitEthernet0/1 | `10.0.1.2/31` | p2p-линк к Leaf-02 (Gi0/0) |
| **Spine-02** | Loopback0 | `10.255.0.2/32` | Идентификатор устройства (Router ID) |
| | GigabitEthernet0/0 | `10.0.2.0/31` | p2p-линк к Leaf-01 (Gi0/1) |
| | GigabitEthernet0/1 | `10.0.2.2/31` | p2p-линк к Leaf-02 (Gi0/1) |
| **Leaf-01** | Loopback0 | `10.255.0.11/32` | Идентификатор устройства (Router ID) |
| | GigabitEthernet0/0 | `10.0.1.1/31` | p2p-линк к Spine-01 (Gi0/0) |
| | GigabitEthernet0/1 | `10.0.2.1/31` | p2p-линк к Spine-02 (Gi0/0) |
| **Leaf-02** | Loopback0 | `10.255.0.12/32` | Идентификатор устройства (Router ID) |
| | GigabitEthernet0/0 | `10.0.1.3/31` | p2p-линк к Spine-01 (Gi0/1) |
| | GigabitEthernet0/1 | `10.0.2.3/31` | p2p-линк к Spine-02 (Gi0/1) |


---

## 3. Верификация и проверка связности

После переноса конфигураций на оборудование была проведена проверка базовой сетевой связности между уровнями фабрики (Underlay линки).

Пример верификации линка с устройства `Leaf-01` до интерфейса `Spine-01` (`10.0.1.0`):
```text
Leaf-01# ping 10.0.1.0
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.0.1.0, timeout is 2 seconds:
.!!!!
Success rate is 80 percent (4/5), round-trip min/avg/max = 10/12/17 ms
```
*Примечание: Первая потеря пакета обусловлена успешной отработкой протокола ARP.*

---

## 4. Конфигурация оборудования

Ниже приведены файлы конфигурации для каждого устройства фабрики. На L3-коммутаторах (Spine) интерфейсы переведены в режим L3 с помощью команды `no switchport`.

### Spine-01
```ios
hostname Spine-01
!
interface Loopback0
 ip address 10.255.0.1 255.255.255.255
!
interface GigabitEthernet0/0
 no switchport
 ip address 10.0.1.0 255.255.255.254
 no shutdown
!
interface GigabitEthernet0/1
 no switchport
 ip address 10.0.1.2 255.255.255.254
 no shutdown
```

### Spine-02
```ios
hostname Spine-02
!
interface Loopback0
 ip address 10.255.0.2 255.255.255.255
!
interface GigabitEthernet0/0
 no switchport
 ip address 10.0.2.0 255.255.255.254
 no shutdown
!
interface GigabitEthernet0/1
 no switchport
 ip address 10.0.2.2 255.255.255.254
 no shutdown
```

### Leaf-01
```ios
hostname Leaf-01
!
interface Loopback0
 ip address 10.255.0.11 255.255.255.255
!
interface GigabitEthernet0/0
 ip address 10.0.1.1 255.255.255.254
 no shutdown
!
interface GigabitEthernet0/1
 ip address 10.0.2.1 255.255.255.254
 no shutdown
```

### Leaf-02
```ios
hostname Leaf-02
!
interface Loopback0
 ip address 10.255.0.12 255.255.255.255
!
interface GigabitEthernet0/0
 ip address 10.0.1.3 255.255.255.254
 no shutdown
!
interface GigabitEthernet0/1
 ip address 10.0.2.3 255.255.255.254
 no shutdown
```

---

## 5. Выводы по работе
1. Спроектирована и развернута отказоустойчивая двухъярусная архитектура CLOS (Spine-and-Leaf).
2. Реализована эффективная схема адресации p2p каналов с использованием масок `/31`, что исключает потерю адресов на Network/Broadcast.
3. Успешно проверена базовая IP-связность внутри созданной Underlay-сети.
