# Underlay. BGP

## Цель
* Настроить BGP для Underlay сети

**Ожидаемый результат**
1. Настроен BGP в Underlay сети и IP-связанность между всему сетевыми устройствами.
2. В документации зафиксирован план работ, адресное пространство, схема сети и конфигурация устройств.
3. Подтверждена IP-связанность между устройствами.

## Планирование AS

<details><summary>Под катом планирование номеров AS</summary>

_Disclaimer_

_После сложной рабочей недели планирование AS пришлось делать с помощью чат-бота_

Предлагается следующая схема использования нумерации AS:

* Используем 32-битную нумерацию.
* Используем 32-битные частные ASNs из диапазона 4200000000–4294967294.
* Первые 16 бит отводятся для обозначения страны, остальные для использования внутри страны.

Кодируем каждую страну символом:

| Страна              | Код страны |
|---------------------|------------|
| Новая Зеландия (NZ) | 1          | 
| Норвегия (NO)       | 2          |
| Бразилия (BR)       | 3          |
| Зарезервировано     | 4–65535    |

Базовый AS номер для каждой страны вычисляется по формуле:

    Базовый AS = 4200000000 + (Код страны × 65536)

**Пример для Норвегии:**

    AS = 4200000000 + (2 × 65536) = 4200131072


Структура внутри страны:

    [Код города (8 бит)][Код региона/ДЦ (8 бит)]

Пример для Норвегии (NO)

| Город           | Код города |
|-----------------|------------|
| Осло (OSL)      | 1          |
| Тёнсберг (TBG)  | 2          |
| Зарезервировано | 3-255      |

Базовый AS для города вычисляется следующим образом:

    Базовый AS города = Базовый AS страны + (Код города × 256)

**Пример для Осло (OSL)**

    AS = 4200131072 + (1 × 256) = 4200131328

Расчет распределения AS внутри города по датацентрам.

| ДЦ               | Код ДЦ |
|------------------|--------|
| DC1 ("Мидгард")  | 1      |
| DC2 ("Альвхейм") | 2      |
| Зарезервировано  | 3-255  |

Для DC1 ("Мидгард")

    AS = 4200131328 + 1 = 4200131329

Spine-коммутаторы: AS 4200131329

Leaf-коммутаторы: диапазон 4200131331–4200131583 для уникальных ASNs.


_Заметка._

_Распределение AS нужно пересмотреть в пределах Города-ДЦ. Пока используем так._


------------------------------

</details>


* Для Spine-коммутаторов внутри DC1 ("Мидгард") используем следующую AS: 4200131329
* Для каждого Leaf-коммутатора используем уникальный номер AS:
  * no-osl-dc1-f1-r03k01-lf01 - 4200131331
  * no-osl-dc1-f1-r03k02-lf01 - 4200131332
  * no-osl-dc1-f1-r03k03-lf01 - 4200131333

То же самое, но табличкой:

| Unit                       | Simple Name | AS Number   |
|----------------------------|-------------|-------------|
| no-osl-dc1-f1-r01k01-spn01 | Spine01     | 4200131329  |
| no-osl-dc1-f1-r02k01-spn01 | Spine02     | 4200131329  |
| no-osl-dc1-f1-r03k01-lf01  | Leaf01      | 4200131331  |
| no-osl-dc1-f1-r03k02-lf01  | Leaf02      | 4200131332  |
| no-osl-dc1-f1-r03k03-lf01  | Leaf03      | 4200131333  |

На картинке выглядит так:

![](images/HW-4-map.png)

## Достижение результата

В результате поисков путей реализации лабораторный Netbox был дополнен двумя, на мой взгляд, полезными плагинами:
* NetBox BGP Plugin https://github.com/netbox-community/netbox-bgp/ - документирование BGP.
* NextBox UI Plugin https://github.com/iDebugAll/nextbox-ui-plugin - визуализация топологий.

Итак, мы имеем установленный и настроенный Netbox с плагином "NetBox BGP Plugin". Далее, нам нужно заполнить информацию о BGP, а именно:
1. Создать в разделе `IPAM -> Aggregates -> RIRs` хотя бы один [RIR](https://en.wikipedia.org/wiki/Regional_Internet_registry)
![](images/netbox_rir.png)
2. Создать в разделе `IPAM -> ASNS -> ASNs` автономные системы согласно таблички выше и соотнести их к RIR:
![](images/netbox_asns.png)
3. Заполнить информацию о BGP сессиях в разделе `Plugins ->BGP -> Sessions` :
![](images/netbox_bgp_sessions.png)

_Примечание_

* _Для массового добавления нужно использовать импорт._
* _Высокий риск допустить ошибку при ручном заполнении._

4. Применяем новый шаблон для генерации конфигурации устройств и BGP [netbox_bgp_template](files/netbox_bgp_template.jinja2).
5. Применяем Rendered Config к устройствам (все еще вручную)

* [no-osl-dc1-f1-r01k01-spn01.conf](files/no-osl-dc1-f1-r01k01-spn01.conf)
* [no-osl-dc1-f1-r02k01-spn01.conf](files/no-osl-dc1-f1-r02k01-spn01.conf)
* [no-osl-dc1-f1-r03k01-lf01.conf](files/no-osl-dc1-f1-r03k01-lf01.conf)
* [no-osl-dc1-f1-r03k02-lf01.conf](files/no-osl-dc1-f1-r03k02-lf01.conf)
* [no-osl-dc1-f1-r03k03-lf01.conf](files/no-osl-dc1-f1-r03k03-lf01.conf)

## Результат настройки BGP

<details><summary>no-osl-dc1-f1-r03k01-lf01: sh ip route bgp</summary>

    VRF: default
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
    
    B E      10.16.0.1/32 [200/0] via 10.16.2.0, Ethernet1
    B E      10.16.0.2/32 [200/0] via 10.16.2.6, Ethernet2
    B E      10.16.1.2/32 [200/0] via 10.16.2.0, Ethernet1
    via 10.16.2.6, Ethernet2
    B E      10.16.1.3/32 [200/0] via 10.16.2.0, Ethernet1
    via 10.16.2.6, Ethernet2
    B E      10.16.2.2/31 [200/0] via 10.16.2.0, Ethernet1
    B E      10.16.2.4/31 [200/0] via 10.16.2.0, Ethernet1
    B E      10.16.2.8/31 [200/0] via 10.16.2.6, Ethernet2
    B E      10.16.2.10/31 [200/0] via 10.16.2.6, Ethernet2
    B E      10.16.4.2/32 [200/0] via 10.16.2.0, Ethernet1
    via 10.16.2.6, Ethernet2
    B E      10.16.4.3/32 [200/0] via 10.16.2.0, Ethernet1
    via 10.16.2.6, Ethernet2
</details>
<details><summary>no-osl-dc1-f1-r03k02-lf01: sh ip route bgp</summary>

    VRF: default
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
    
     B E      10.16.0.1/32 [200/0] via 10.16.2.2, Ethernet1
     B E      10.16.0.2/32 [200/0] via 10.16.2.8, Ethernet2
     B E      10.16.1.1/32 [200/0] via 10.16.2.2, Ethernet1
                                   via 10.16.2.8, Ethernet2
     B E      10.16.1.3/32 [200/0] via 10.16.2.2, Ethernet1
                                   via 10.16.2.8, Ethernet2
     B E      10.16.2.0/31 [200/0] via 10.16.2.2, Ethernet1
     B E      10.16.2.4/31 [200/0] via 10.16.2.2, Ethernet1
     B E      10.16.2.6/31 [200/0] via 10.16.2.8, Ethernet2
     B E      10.16.2.10/31 [200/0] via 10.16.2.8, Ethernet2
     B E      10.16.4.1/32 [200/0] via 10.16.2.2, Ethernet1
                                   via 10.16.2.8, Ethernet2
     B E      10.16.4.3/32 [200/0] via 10.16.2.2, Ethernet1
                                   via 10.16.2.8, Ethernet2

</details>
<details><summary>no-osl-dc1-f1-r03k03-lf01: sh ip route bgp</summary>

    VRF: default
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
    
     B E      10.16.0.1/32 [200/0] via 10.16.2.4, Ethernet1
     B E      10.16.0.2/32 [200/0] via 10.16.2.10, Ethernet2
     B E      10.16.1.1/32 [200/0] via 10.16.2.4, Ethernet1
                                   via 10.16.2.10, Ethernet2
     B E      10.16.1.2/32 [200/0] via 10.16.2.4, Ethernet1
                                   via 10.16.2.10, Ethernet2
     C        10.16.1.3/32 is directly connected, Loopback0
     B E      10.16.2.0/31 [200/0] via 10.16.2.4, Ethernet1
     B E      10.16.2.2/31 [200/0] via 10.16.2.4, Ethernet1
     C        10.16.2.4/31 is directly connected, Ethernet1
     B E      10.16.2.6/31 [200/0] via 10.16.2.10, Ethernet2
     B E      10.16.2.8/31 [200/0] via 10.16.2.10, Ethernet2
     C        10.16.2.10/31 is directly connected, Ethernet2
     B E      10.16.4.1/32 [200/0] via 10.16.2.4, Ethernet1
                                   via 10.16.2.10, Ethernet2
     B E      10.16.4.2/32 [200/0] via 10.16.2.4, Ethernet1
                                   via 10.16.2.10, Ethernet2
     C        10.16.4.3/32 is directly connected, Loopback10

</details>

Проверяем доступность loopback интерфейсов

<details><summary>ping</summary>

    no-osl-dc1-f1-r03k01-lf01#ping 10.16.1.2 source 10.16.1.1
    PING 10.16.1.2 (10.16.1.2) from 10.16.1.1 : 72(100) bytes of data.
    80 bytes from 10.16.1.2: icmp_seq=1 ttl=63 time=10.2 ms
    80 bytes from 10.16.1.2: icmp_seq=2 ttl=63 time=6.67 ms
    80 bytes from 10.16.1.2: icmp_seq=3 ttl=63 time=6.87 ms
    80 bytes from 10.16.1.2: icmp_seq=4 ttl=63 time=7.18 ms
    80 bytes from 10.16.1.2: icmp_seq=5 ttl=63 time=6.53 ms
    
    --- 10.16.1.2 ping statistics ---
    5 packets transmitted, 5 received, 0% packet loss, time 38ms
    rtt min/avg/max/mdev = 6.539/7.504/10.256/1.396 ms, ipg/ewma 9.632/8.831 ms

    no-osl-dc1-f1-r03k01-lf01#ping 10.16.1.3 source 10.16.1.1
    PING 10.16.1.3 (10.16.1.3) from 10.16.1.1 : 72(100) bytes of data.
    80 bytes from 10.16.1.3: icmp_seq=1 ttl=63 time=8.93 ms
    80 bytes from 10.16.1.3: icmp_seq=2 ttl=63 time=6.03 ms
    80 bytes from 10.16.1.3: icmp_seq=3 ttl=63 time=6.54 ms
    80 bytes from 10.16.1.3: icmp_seq=4 ttl=63 time=6.92 ms
    80 bytes from 10.16.1.3: icmp_seq=5 ttl=63 time=6.67 ms
    
    --- 10.16.1.3 ping statistics ---
    5 packets transmitted, 5 received, 0% packet loss, time 34ms
    rtt min/avg/max/mdev = 6.033/7.023/8.933/1.003 ms, ipg/ewma 8.546/7.960 ms

    no-osl-dc1-f1-r03k01-lf01#ping 10.16.4.2 source 10.16.4.1
    PING 10.16.4.2 (10.16.4.2) from 10.16.4.1 : 72(100) bytes of data.
    80 bytes from 10.16.4.2: icmp_seq=1 ttl=63 time=8.34 ms
    80 bytes from 10.16.4.2: icmp_seq=2 ttl=63 time=7.16 ms
    80 bytes from 10.16.4.2: icmp_seq=3 ttl=63 time=11.9 ms
    80 bytes from 10.16.4.2: icmp_seq=4 ttl=63 time=7.64 ms
    80 bytes from 10.16.4.2: icmp_seq=5 ttl=63 time=9.38 ms
    
    --- 10.16.4.2 ping statistics ---
    5 packets transmitted, 5 received, 0% packet loss, time 37ms
    rtt min/avg/max/mdev = 7.162/8.896/11.944/1.700 ms, ipg/ewma 9.480/8.645 ms

    no-osl-dc1-f1-r03k01-lf01#ping 10.16.4.3 source 10.16.4.1
    PING 10.16.4.3 (10.16.4.3) from 10.16.4.1 : 72(100) bytes of data.
    80 bytes from 10.16.4.3: icmp_seq=1 ttl=63 time=11.0 ms
    80 bytes from 10.16.4.3: icmp_seq=2 ttl=63 time=10.6 ms
    80 bytes from 10.16.4.3: icmp_seq=3 ttl=63 time=9.23 ms
    80 bytes from 10.16.4.3: icmp_seq=4 ttl=63 time=8.72 ms
    80 bytes from 10.16.4.3: icmp_seq=5 ttl=63 time=11.5 ms
    
    --- 10.16.4.3 ping statistics ---
    5 packets transmitted, 5 received, 0% packet loss, time 48ms
    rtt min/avg/max/mdev = 8.722/10.251/11.557/1.090 ms, ipg/ewma 12.173/10.661 ms

</details>