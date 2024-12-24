# VxLAN. Multihoming.

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

В качестве технического долга имеем:
* zero touch provisioning устройств 
* раскатывание конфигурации в автоматическом режиме


Вводятся новые понятия, а именно:

* **ES - ethernet segment**. An Ethernet segment is associated with an access-facing interface of a VTEP and represents the connection
  with a host device. Each Ethernet segment is assigned a unique value known as Ethernet segment identifier
  (ESI). When a host device is connected to more than one VTEPs, then the ESI for these connections remains
  the same. [Источник](https://www.juniper.net/documentation/us/en/software/nce/evpn-lag-multihoming-guide/topics/concept/evpn-lag-guide-introduction.html)
* **ESI - Ethernet Segment Identifier**. An EVPN LAG is identified using an Ethernet segment identifier (ESI). An ESI is a mandatory attribute that is required to enable EVPN LAG server multihoming. [Источник](https://www.juniper.net/documentation/us/en/software/nce/evpn-lag-multihoming-guide/topics/task/evpn-lag-guide-esi-types-lacp.html)

  В EVPN-сетях — это ключ, по которому несколько PE-маршрутизаторов/коммутаторов идентифицируют общий Ethernet-сегмент (Multi-Homing). Он позволяет:
  1. Обнаруживать и управлять мультихомингом (Active/Active или Active/Standby).
  2. Избегать петель и дублирования кадров через split-horizon.
  3. Корректно маршрутизировать/рассылать трафик в сценариях отказоустойчивости и балансировки нагрузки.

Без ESI протокол EVPN не мог бы грамотно различать отдельные физические сегменты и обеспечивать мультихоминг с сохранением изоляции и согласованности маршрутов/кадров.

#### Схема сети

За основу принимается всё та же топология из задания [Часть 5. VxLAN. EVPN L2](https://github.com/anton-sap/otus-dc-network-design/tree/master/HW-5%20-%20VxLAN%20EVPN%20L2)

В нее добавлена еще одна пара leaf'ов. Заодно проверим на надежность "конфигуратор" из Netbox и задокументируем процесс онбоардинга новых устройств xD 

Для упрощения и экономии ресурсов будем работать внутри Rack 3 и Rack 4 согласно схемы
* Внутри Rack 4 будет работать устройство на базе RHEL 8.4 с агрегированным (Bond0) интерфейсом, состоящим из двух подключений: первое - в сторону no-osl-dc1-f1-r03k04-lf01, второе - в сторону no-osl-dc1-f1-r03k04-lf02.

Это эмуляция подключения конечного сервера в двум ToR коммутаторам в пределах серверной стойки.

![](images/HW-7-map.png)

Тут мне попадается офигенная композиция в Яндекс.Музыке [Valhalla Calling](https://music.yandex.ru/album/31716158/track/127207559?utm_source=desktop&utm_medium=copy_link) и еще один трек [Ghost Riders in the Sky](https://music.yandex.ru/album/23613260/track/107575010?utm_source=desktop&utm_medium=copy_link). Фак, чистый кайф!

    VXLAN thunders in twilight’s dome,
    ESI’s banner guides the roaming band.
    Multihome stands as bonds of home,
    Valhalla calls—EVPN at hand!
    А. С. ChatGPT (c.)

Тут мы останавливаемся думать за генерацию ESI - он же Ethernet Segment Identifier. Самый распространённый формат — тип 0x00 (Manual), где 9 байт могут быть заданы вручную (часто встречается тип 00:aa:bb:cc:dd:...).

~~Предполагается, что ESI сегмент не распространяется за пределы серверной стойки. В таком случае мы можем привязаться к физической стойке (Rack)~~

В Netbox вводятся два новых поля  в `Customization -> Custom Fields` для документирования значений полей:
* ESI ID - идентификатор Ethernet сегмента с проверкой по Regex `^[0-9A-Fa-f]{4}(:[0-9A-Fa-f]{4}){4}$`
* LACP System-id - идентификатор LACP для коммутаторов с проверкой по Regex `^[0-9A-Fa-f]{4}(.[0-9A-Fa-f]{4}){2}$`

![](images/netbox_custom_fields.png)

Шаблон для генерации конфигурации для leaf'ов  претерпел значительные изменения и заставил потратить на себя кучу времени. Основные изменения коснулись секций для интерфейсов. Теперь по каждому типу интерфейсов собственный цикл обработки. ниже приведен пример для секции Port-Channel и кастомных полей:
    
    {%- for interface in device.interfaces.all() %}
      {%- if interface.name.startswith('Port-Channel') %}
    interface {{ interface.name }}
        {%- set ip_list = interface.ip_addresses.all() %}
        {%- if ip_list|length > 0 %}
          {%- for ip in ip_list %}
      mtu 9214
      no switchport
      bfd interval 200 min-rx 200 multiplier 3
      ip address {{ ip.address }}
          {%- endfor %}
        {%- else %}
          {%- if interface.mode == 'access' and interface.untagged_vlan %}
      switchport
      switchport mode access
      switchport access vlan {{ interface.untagged_vlan.vid }}
      !
          {%- elif interface.mode == 'tagged' and interface.tagged_vlans.all() %}
      switchport
      switchport mode trunk
            {%- if interface.untagged_vlan %}
      switchport trunk native vlan {{ interface.untagged_vlan.vid }}
            {%- endif %}
      switchport trunk allowed vlan {{ interface.tagged_vlans.all()|map(attribute='vid')|join(',') }}
      !
          {%- else %}
      switchport
      switchport mode access
      switchport access vlan 1
          {%- endif %}
        {%- endif %}
        {%- if interface.custom_field_data["esi_id"] %}                       # Настройка ESI
      evpn ethernet-segment
        identifier {{ interface.custom_field_data["esi_id"] }}
        {%- endif %}
        {%- if interface.custom_field_data["lacp_sid"] %}                      # Настройка LACP
      lacp system-id {{ interface.custom_field_data["lacp_sid"] }}
        {%- endif %}
        {%- if not interface.enabled %}
      shutdown
        {%- else %}
      no shutdown
        {%- endif %}
        {%- if interface.description %}
      description {{ interface.description }}
        {%- endif %}
    !
      {%- endif %}
    {%- endfor %}

