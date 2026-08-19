# Лабораторная работа №2. Настройка OSPF в Underlay-сети фабрики CLOS

**Цель работы:** Настроить протокол динамической маршрутизации OSPF (Open Shortest Path First) в Underlay-сети для обеспечения автоматической IP-связности между всеми сетевыми устройствами фабрики CLOS.

---

## 1. Топология сети

Топология полностью наследует архитектуру из первой лабораторной работы, собранную в эмуляторе PNETLab.

![Топология CLOS](pnet_topology.png)

* **Spine-уровень:** 2 коммутатора Cisco vIOS L3 (`Spine-01`, `Spine-02`).
* **Leaf-уровень:** 2 роутера Cisco vIOS (`Leaf-01`, `Leaf-02`).
* **Схема подключений:** Каждый Leaf-коммутатор соединён с каждым Spine-коммутатором (full-mesh на уровне Spine-Leaf).

---

## 2. План работ

1. **Анализ существующей конфигурации:** Проверить назначение IP-адресов на всех интерфейсах и Loopback.
2. **Планирование OSPF:** Выбрать зону (Area 0), определить Router ID для каждого устройства (использовать Loopback-адреса).
3. **Настройка OSPF на Spine-коммутаторах:** Включить процесс OSPF, объявить все P2P-линки и Loopback-интерфейс.
4. **Настройка OSPF на Leaf-коммутаторах:** Включить процесс OSPF, объявить все P2P-линки и Loopback-интерфейс.
5. **Верификация:** Проверить установление соседств (OSPF neighbors) и наличие маршрутов в таблице маршрутизации.
6. **Документирование:** Зафиксировать конфигурации и результаты проверки.

---

## 3. Адресное пространство Underlay (неизменно)

Используется адресное пространство, спроектированное в первой лабораторной работе.

| Устройство | Интерфейс | IP-адрес / Маска | Назначение |
| :--- | :--- | :--- | :--- |
| **Spine-01** | Loopback0 | `10.255.0.1/32` | Router ID |
| | GigabitEthernet0/0 | `10.0.1.0/31` | p2p-линк к Leaf-01 |
| | GigabitEthernet0/1 | `10.0.1.2/31` | p2p-линк к Leaf-02 |
| **Spine-02** | Loopback0 | `10.255.0.2/32` | Router ID |
| | GigabitEthernet0/0 | `10.0.2.0/31` | p2p-линк к Leaf-01 |
| | GigabitEthernet0/1 | `10.0.2.2/31` | p2p-линк к Leaf-02 |
| **Leaf-01** | Loopback0 | `10.255.0.11/32` | Router ID |
| | GigabitEthernet0/0 | `10.0.1.1/31` | p2p-линк к Spine-01 |
| | GigabitEthernet0/1 | `10.0.2.1/31` | p2p-линк к Spine-02 |
| **Leaf-02** | Loopback0 | `10.255.0.12/32` | Router ID |
| | GigabitEthernet0/0 | `10.0.1.3/31` | p2p-линк к Spine-01 |
| | GigabitEthernet0/1 | `10.0.2.3/31` | p2p-линк к Spine-02 |

---

## 4. Конфигурация оборудования (OSPF)

Ниже приведены конфигурации для всех устройств. В качестве протокола маршрутизации используется **OSPFv2 для IPv4**.

### 4.1. Конфигурация Spine-01

```cisco
hostname Spine-01
!
interface Loopback0
 ip address 10.255.0.1 255.255.255.255
!
interface GigabitEthernet0/0
 no switchport
 ip address 10.0.1.0 255.255.255.254
 ip ospf network point-to-point
 no shutdown
!
interface GigabitEthernet0/1
 no switchport
 ip address 10.0.1.2 255.255.255.254
 ip ospf network point-to-point
 no shutdown
!
router ospf 1
 router-id 10.255.0.1
 network 10.0.1.0 0.0.0.1 area 0
 network 10.0.1.2 0.0.0.1 area 0
 network 10.255.0.1 0.0.0.0 area 0
!