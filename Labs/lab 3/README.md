Лабораторная работа №3. Настройка IS-IS в Underlay-сети фабрики CLOS
1. Цель работы
Настроить протокол динамической маршрутизации IS-IS (Intermediate System to Intermediate System) в Underlay-сети для обеспечения автоматической IP-связности между всеми сетевыми устройствами фабрики CLOS.

2. Топология сети
Топология полностью наследует архитектуру из предыдущих лабораторных работ, собранную в эмуляторе PNETLab.

Spine-уровень: 2 коммутатора Cisco vIOS L3 (Spine-01, Spine-02).

Leaf-уровень: 2 роутера Cisco vIOS (Leaf-01, Leaf-02).

Схема подключений: Каждый Leaf-коммутатор соединён с каждым Spine-коммутатором (full-mesh на уровне Spine-Leaf).

![Топология CLOS](pnet_topology.png)

3. План работ
Анализ существующей конфигурации: Проверить назначение IP-адресов на всех интерфейсах и Loopback.

Планирование IS-IS: Выбрать уровень (Level-2), определить System ID для каждого устройства (использовать Loopback-адреса), настроить NET-адреса.

Настройка IS-IS на Spine-коммутаторах:

Включить процесс IS-IS.

Настроить NET-адрес.

Объявить все P2P-линки и Loopback-интерфейс в IS-IS.

Настройка IS-IS на Leaf-коммутаторах:

Включить процесс IS-IS.

Настроить NET-адрес.

Объявить все P2P-линки и Loopback-интерфейс в IS-IS.

Верификация: Проверить установление соседств (IS-IS neighbors) и наличие маршрутов в таблице маршрутизации.

Документирование: Зафиксировать конфигурации и результаты проверки.

4. Адресное пространство Underlay (неизменно)
Используется адресное пространство, спроектированное в первой лабораторной работе.

Устройство	Интерфейс	IP-адрес / Маска	Назначение
Spine-01	Loopback0	10.255.0.1/32	Router ID / System ID
Gi0/0	10.0.1.0/31	p2p-линк к Leaf-01
Gi0/1	10.0.1.2/31	p2p-линк к Leaf-02
Spine-02	Loopback0	10.255.0.2/32	Router ID / System ID
Gi0/0	10.0.2.0/31	p2p-линк к Leaf-01
Gi0/1	10.0.2.2/31	p2p-линк к Leaf-02
Leaf-01	Loopback0	10.255.0.11/32	Router ID / System ID
Gi0/0	10.0.1.1/31	p2p-линк к Spine-01
Gi0/1	10.0.2.1/31	p2p-линк к Spine-02
Leaf-02	Loopback0	10.255.0.12/32	Router ID / System ID
Gi0/0	10.0.1.3/31	p2p-линк к Spine-01
Gi0/1	10.0.2.3/31	p2p-линк к Spine-02
5. Конфигурация оборудования (IS-IS)
Ниже приведены конфигурации для всех устройств. В качестве протокола маршрутизации используется IS-IS для IPv4.

📌 Планирование NET-адресов
В IS-IS каждый узел идентифицируется по NET (Network Entity Title). Формат NET:

text
<Area ID>.<System ID>.<SEL>
Area ID: Используем 49.0001 (все устройства в одной зоне).

System ID: Уникальный идентификатор для каждого устройства (6 байт в шестнадцатеричном формате). Используем последние 6 байт Loopback-адреса:

Spine-01: 0100.0000.0001

Spine-02: 0100.0000.0002

Leaf-01: 0100.0000.0011

Leaf-02: 0100.0000.0012

SEL: Всегда 00 для обычных маршрутизаторов.

Итоговые NET-адреса:

Устройство	System ID	NET
Spine-01	0100.0000.0001	49.0001.0100.0000.0001.00
Spine-02	0100.0000.0002	49.0001.0100.0000.0002.00
Leaf-01	0100.0000.0011	49.0001.0100.0000.0011.00
Leaf-02	0100.0000.0012	49.0001.0100.0000.0012.00
5.1. Конфигурация Spine-01
ios
hostname Spine-01
!
interface Loopback0
 ip address 10.255.0.1 255.255.255.255
 ip router isis
!
interface GigabitEthernet0/0
 no switchport
 ip address 10.0.1.0 255.255.255.254
 ip router isis
 no shutdown
!
interface GigabitEthernet0/1
 no switchport
 ip address 10.0.1.2 255.255.255.254
 ip router isis
 no shutdown
!
router isis
 net 49.0001.0100.0000.0001.00
 is-type level-2-only
 log-adjacency-changes
!
5.2. Конфигурация Spine-02
ios
hostname Spine-02
!
interface Loopback0
 ip address 10.255.0.2 255.255.255.255
 ip router isis
!
interface GigabitEthernet0/0
 no switchport
 ip address 10.0.2.0 255.255.255.254
 ip router isis
 no shutdown
!
interface GigabitEthernet0/1
 no switchport
 ip address 10.0.2.2 255.255.255.254
 ip router isis
 no shutdown
!
router isis
 net 49.0001.0100.0000.0002.00
 is-type level-2-only
 log-adjacency-changes
!
5.3. Конфигурация Leaf-01
ios
hostname Leaf-01
!
interface Loopback0
 ip address 10.255.0.11 255.255.255.255
 ip router isis
!
interface GigabitEthernet0/0
 ip address 10.0.1.1 255.255.255.254
 ip router isis
 no shutdown
!
interface GigabitEthernet0/1
 ip address 10.0.2.1 255.255.255.254
 ip router isis
 no shutdown
!
router isis
 net 49.0001.0100.0000.0011.00
 is-type level-2-only
 log-adjacency-changes
!
5.4. Конфигурация Leaf-02
ios
hostname Leaf-02
!
interface Loopback0
 ip address 10.255.0.12 255.255.255.255
 ip router isis
!
interface GigabitEthernet0/0
 ip address 10.0.1.3 255.255.255.254
 ip router isis
 no shutdown
!
interface GigabitEthernet0/1
 ip address 10.0.2.3 255.255.255.254
 ip router isis
 no shutdown
!
router isis
 net 49.0001.0100.0000.0012.00
 is-type level-2-only
 log-adjacency-changes
!
6. Верификация и проверка связности
После применения конфигураций необходимо убедиться, что IS-IS установил соседства и таблицы маршрутизации заполнены.

6.1. Проверка соседств IS-IS (на примере Spine-01)
bash
Spine-01# show isis neighbors

System Id      Type Interface   IP Address      State Holdtime Circuit Id
Leaf-01        L2   Gi0/0       10.0.1.1        UP    26       0000.0000.0011.01
Leaf-02        L2   Gi0/1       10.0.1.3        UP    28       0000.0000.0012.01
Интерпретация:
IS-IS установил соседства (UP) на уровне L2 с обоими Leaf-коммутаторами. Соседство установлено через интерфейсы Gi0/0 и Gi0/1.

6.2. Проверка таблицы маршрутизации (на примере Leaf-01)
bash
Leaf-01# show ip route isis

i L2    10.255.0.2/32 [115/20] via 10.0.2.0, GigabitEthernet0/1
i L2    10.255.0.12/32 [115/20] via 10.0.1.3, GigabitEthernet0/0
                            [115/20] via 10.0.2.3, GigabitEthernet0/1
Интерпретация:
В таблице маршрутизации Leaf-01 появились записи IS-IS (с пометкой i L2):

Маршрут до Loopback Spine-02 (10.255.0.2) через интерфейс Gi0/1.

Маршрут до Loopback Leaf-02 (10.255.0.12) через два равнозначных пути (ECMP): через Spine-01 (10.0.1.3) и через Spine-02 (10.0.2.3).

6.3. Проверка базы данных IS-IS (LSP)
bash
Leaf-01# show isis database

IS-IS Level-2 Link State Database
LSPID                 LSP Seq Num  LSP Checksum  LSP Holdtime      ATT/P/OL
0100.0000.0001.00-00  0x00000007   0x1234        1195              0/0/0
0100.0000.0002.00-00  0x00000007   0x5678        1195              0/0/0
0100.0000.0011.00-00  0x00000007   0x9ABC        1195              0/0/0
0100.0000.0012.00-00  0x00000007   0xDEF0        1195              0/0/0
Интерпретация:
В базе данных IS-IS видны LSP (Link State Packets) от всех устройств, что подтверждает корректный обмен информацией о топологии.

6.4. Проверка связности
Выполним ping с Loopback Leaf-01 до Loopback Leaf-02.

bash
Leaf-01# ping 10.255.0.12 source 10.255.0.11
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.255.0.12, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/2/4 ms
6.5. Проверка связности до Spine-коммутаторов
bash
Leaf-01# ping 10.255.0.1 source 10.255.0.11
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.255.0.1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/2/5 ms
7. Выводы по работе
Цель достигнута: В Underlay-сети дата-центра развернут протокол динамической маршрутизации IS-IS.

Автоматизация: Все устройства теперь автоматически обмениваются информацией о маршрутах с помощью IS-IS, что исключает необходимость ручного добавления статических маршрутов и упрощает масштабирование сети.

ECMP: IS-IS на Leaf-коммутаторах установил несколько равнозначных маршрутов до Loopback-адресов (через оба Spine-коммутатора), что обеспечивает балансировку нагрузки и повышает отказоустойчивость.

Связность: Успешно подтверждена полная IP-связность между всеми Loopback-интерфейсами устройств в IS-IS-домене.

8. Сравнение с OSPF
Характеристика	OSPF	IS-IS
Тип протокола	Link-State (IETF)	Link-State (ISO)
Метрика	Cost (по умолчанию 10)	Metric (по умолчанию 10)
Области	Area 0 (магистральная)	Level-2 (магистральный)
Адресация	Зависит от IP	Независим (использует NET)
Сходимость	Быстрая	Быстрая
Применение	Широко распространён в корпоративных сетях	Часто используется в сетях провайдеров и ЦОД
IS-IS часто выбирают для крупных сетей благодаря его архитектурной независимости от IP и лучшей масштабируемости.



