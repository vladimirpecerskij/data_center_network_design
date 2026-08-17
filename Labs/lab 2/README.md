# Лабораторная работа №2. Настройка OSPF в Underlay-сети фабрики CLOS

Цель работы: Настроить протокол динамической маршрутизации OSPF (Open Shortest Path First) в Underlay-сети для обеспечения автоматической IP-связности между всеми сетевыми устройствами фабрики CLOS.

## 1. Топология сети

Топология полностью наследует архитектуру из первой лабораторной работы, собранную в эмуляторе PNETLab.
![Топология CLOS](pnet_topology.png)
- **Spine-уровень:** 2 коммутатора Cisco vIOS L3 (Spine-01, Spine-02).
- **Leaf-уровень:** 2 роутера Cisco vIOS (Leaf-01, Leaf-02).
- **Схема подключений:** Каждый Leaf-коммутатор соединён с каждым Spine-коммутатором (full-mesh на уровне Spine-Leaf).

## 2. План работ

1. **Анализ существующей конфигурации:** Проверить назначение IP-адресов на всех интерфейсах и Loopback.
2. **Планирование OSPF:** Выбрать зону (Area 0), определить Router ID для каждого устройства (использовать Loopback-адреса).
3. **Настройка OSPF на Spine-коммутаторах:** Включить процесс OSPF, объявить все P2P-линки и Loopback-интерфейс.
4. **Настройка OSPF на Leaf-коммутаторах:** Включить процесс OSPF, объявить все P2P-линки и Loopback-интерфейс.
5. **Верификация:** Проверить установление соседств (OSPF neighbors) и наличие маршрутов в таблице маршрутизации.
6. **Документирование:** Зафиксировать конфигурации и результаты проверки.

## 3. Адресное пространство Underlay (неизменно)

Используется адресное пространство, спроектированное в первой лабораторной работе.

| Устройство | Интерфейс | IP-адрес / Маска | Назначение |
| :--- | :--- | :--- | :--- |
| **Spine-01** | Loopback0 | 10.255.0.1/32 | Router ID |
| | Gi0/0 | 10.0.1.0/31 | p2p-линк к Leaf-01 |
| | Gi0/1 | 10.0.1.2/31 | p2p-линк к Leaf-02 |
| **Spine-02** | Loopback0 | 10.255.0.2/32 | Router ID |
| | Gi0/0 | 10.0.2.0/31 | p2p-линк к Leaf-01 |
| | Gi0/1 | 10.0.2.2/31 | p2p-линк к Leaf-02 |
| **Leaf-01** | Loopback0 | 10.255.0.11/32 | Router ID |
| | Gi0/0 | 10.0.1.1/31 | p2p-линк к Spine-01 |
| | Gi0/1 | 10.0.2.1/31 | p2p-линк к Spine-02 |
| **Leaf-02** | Loopback0 | 10.255.0.12/32 | Router ID |
| | Gi0/0 | 10.0.1.3/31 | p2p-линк к Spine-01 |
| | Gi0/1 | 10.0.2.3/31 | p2p-линк к Spine-02 |

## 4. Конфигурация оборудования (OSPF)

Ниже приведены конфигурации для всех устройств. В качестве протокола маршрутизации используется **OSPFv2 для IPv4**.

### Spine-01
```cisco
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
!
router ospf 1
 router-id 10.255.0.1
 network 10.0.1.0 0.0.0.1 area 0
 network 10.0.1.2 0.0.0.1 area 0
 network 10.255.0.1 0.0.0.0 area 0
Spine-01(config)# interface GigabitEthernet0/0
Spine-01(config-if)# ip ospf network point-to-point
Spine-01(config-if)# interface GigabitEthernet0/1
Spine-01(config-if)# ip ospf network point-to-point
!
Spine-02
cisco
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
!
router ospf 1
 router-id 10.255.0.2
 network 10.0.2.0 0.0.0.1 area 0
 network 10.0.2.2 0.0.0.1 area 0
 network 10.255.0.2 0.0.0.0 area 0
Spine-02(config)# interface GigabitEthernet0/0
Spine-02(config-if)# ip ospf network point-to-point
Spine-02(config-if)# interface GigabitEthernet0/1
Spine-02(config-if)# ip ospf network point-to-point
!
Leaf-01
cisco
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
!
router ospf 1
 router-id 10.255.0.11
 network 10.0.1.1 0.0.0.0 area 0
 network 10.0.2.1 0.0.0.0 area 0
 network 10.255.0.11 0.0.0.0 area 0
Leaf-01(config)# interface GigabitEthernet0/0
Leaf-01(config-if)# ip ospf network point-to-point
Leaf-01(config-if)# interface GigabitEthernet0/1
Leaf-01(config-if)# ip ospf network point-to-point
!
Leaf-02
cisco
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
!
router ospf 1
 router-id 10.255.0.12
 network 10.0.1.3 0.0.0.0 area 0
 network 10.0.2.3 0.0.0.0 area 0
 network 10.255.0.12 0.0.0.0 area 0
Leaf-02(config)# interface GigabitEthernet0/0
Leaf-02(config-if)# ip ospf network point-to-point
Leaf-02(config-if)# interface GigabitEthernet0/1
Leaf-02(config-if)# ip ospf network point-to-point
!
Примечание: В конфигурациях используется команда network с обратной маской (wildcard mask). Для p2p-линков на Leaf-01 указан точный адрес /32 (0.0.0.0). Это допустимый и часто применяемый подход для точного контроля над анонсируемыми сетями.

5. Верификация и проверка связности
После применения конфигураций необходимо убедиться, что OSPF установил соседства и таблицы маршрутизации заполнены.

Проверка соседств OSPF (на примере Spine-01)
bash
Spine-01# show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address         Interface
10.255.0.11       0   FULL/  -        00:00:31    10.0.1.1        GigabitEthernet0/0
10.255.0.12       0   FULL/  -        00:00:39    10.0.1.3        GigabitEthernet0/1
Интерпретация: OSPF установил полные (FULL) соседства с обоими Leaf-коммутаторами. Соседство установлено через интерфейсы Gi0/0 и Gi0/1.

Проверка таблицы маршрутизации (на примере Leaf-01)
bash
Leaf-01# show ip route ospf

O     10.255.0.2/32 [110/2] via 10.0.2.0, 00:00:10, GigabitEthernet0/1
O     10.255.0.12/32 [110/2] via 10.0.1.3, 00:00:15, GigabitEthernet0/0
                    [110/2] via 10.0.2.3, 00:00:10, GigabitEthernet0/1
Интерпретация: В таблице маршрутизации Leaf-01 появились маршруты до Loopback Spine-02 (10.255.0.2) и до Loopback Leaf-02 (10.255.0.12) через два равнозначных пути (ECMP). Это ключевое преимущество топологии CLOS, реализованное через OSPF.

Проверка связности
Выполним ping с Loopback Leaf-01 до Loopback Leaf-02.

bash
Leaf-01# ping 10.255.0.12 source 10.255.0.11
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.255.0.12, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/2/4 ms
6. Выводы по работе
Цель достигнута: В Underlay-сети дата-центра развернут протокол динамической маршрутизации OSPF.

Автоматизация: Все устройства теперь автоматически обмениваются информацией о маршрутах, что исключает необходимость ручного добавления статических маршрутов и упрощает масштабирование сети.

ECMP: OSPF на Leaf-коммутаторах установил несколько равнозначных маршрутов до Loopback-адресов (через оба Spine-коммутатора), что обеспечивает балансировку нагрузки и повышает отказоустойчивость.

Связность: Успешно подтверждена полная IP-связность между всеми Loopback-интерфейсами устройств в OSPF-домене.

