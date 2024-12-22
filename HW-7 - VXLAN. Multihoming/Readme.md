# VxLAN. EVPN L3

## Цель
* Настроить отказоустойчивое подключение клиентов с использованием EVPN Multihoming

**Ожидаемый результат**
* Клиент подключен двумя линками к различным Leaf
* Со стороны клиента настроен агрегированный канал (LACP)
* Настроен multihoming для работы в Overlay сети. В нашем случае - ESI LAG.
* Зафиксирован в документации - план работы, адресное пространство, схема сети, конфигурацию устройств.
* Протестирована отказоустойчивость. Необходимо убедиться, что связнность не теряется при отключении одного из линков.

## Достижение результата
### Введение

Вводятся новые понятия, а именно:

* ES - ethernet segment. An Ethernet segment is associated with an access-facing interface of a VTEP and represents the connection
  with a host device. Each Ethernet segment is assigned a unique value known as Ethernet segment identifier
  (ESI). When a host device is connected to more than one VTEPs, then the ESI for these connections remains
  the same. [Источник](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-9/configuration_guide/vxlan/b_179_bgp_evpn_vxlan_9300_cg/bgp_evpn_vxlan_overview.pdf)